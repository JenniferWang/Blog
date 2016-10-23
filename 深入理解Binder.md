深入理解Binder

# 概述
Binder是Android系统提供的一种IPC（进程间通信）机制。由于Android是基于Linux内核的，因此，除了Binder外，还存在其他的IPC机制，例如pipe和socket等。**Binder**相对于其他IPC机制来说，就**更加灵活和方便**了。对于初学Android的朋友而言，最难却又最想掌握的恐怕就是Binder机制了，因为**Android系统基本上可以看作是一个基于Binder通信的C/S架构**。Binder就像网络一样，把系统各个部分连接在了一起，因此它是非常重要的。

在基于Binder通信的C/S架构体系中，除了C/S架构所包括的Client端和Server端外，Android还有一个**全局**的`ServiceManager`端，它的作用是管理系统中的各种服务（Service）。Client、Server和`ServiceManager`这三者之间的交互关系，如图6-1所示 ：

<img src="http://52.53.208.118/wp-content/uploads/2016/09/Screen-Shot-2016-09-20-at-9.05.33-AM-300x135.png" alt="screen-shot-2016-09-20-at-9-05-33-am" width="300" height="135" class="alignnone size-medium wp-image-68" />

* Server进程要先注册Service到`ServiceManager`中。
* 如果某个Client进程要使用某个Service，必须先到`ServiceManager`中获取该Service的相关信息。
* Client根据得到的Service信息建立与Service所在的Server process通信的通路，然后就可以直接与Service交互了。
* 最重要的一点是，三者的交互都是基于Binder通信的，所以通过任意两者之间的关系，都可以揭示Binder的奥秘。

这里，要重点强调的是Binder通信与C/S架构之间的关系。**Binder只是为C/S架构提供了一种通信的方式，我们完全可以采用其他IPC方式进行通信**，例如，系统中有很多其他的程序采用的就是socket或pipe的方法进行进程间通信。很多初学者可能觉得Binder较复杂，尤其是看到诸如`BpXXX`、`BnXXX`之类的定义便感到头晕，这很有可能是把Binder通信层结构和应用的业务层结构搞混了。如果能搞清楚这二者的关系，完全可以自己实现一个不使用`BpXXX`和`BnXXX`的Service。须知，`ServiceManager`可并没有使用它们。

# 详解`MediaServer`
`MediaServer`（简称MS）是一个可执行程序，虽然Android的SDK提供Java层的API，但Android系统本身还是一个完整的基于Linux内核的操作系统，所以_不会是所有程序都用Java编写_，这里的MS就是一个用C++编写的可执行程序。之所以选择`MediaServer`作为切入点，是因为这个Server是系统诸多重要Service的栖息地，它们包括：

* `AudioFlinger`：音频系统中的重要核心服务。
* `AudioPolicyService`：音频系统中关于音频策略的重要服务。
* **`MediaPlayerService`**：多媒体系统中的重要服务。
* `CameraService`：有关摄像/照相的重要服务。

可以看到，MS除了不涉及Surface系统外，其他重要的服务基本上都涉及到了，本章将以其中的`MediaPlayerService`为主切入点进行分析, 先来分析`MediaServer`本身。

## `MediaServer`的入口函数
MS是一个可执行程序，入口函数是`main`，代码如下所示：

```cpp
// Main_MediaServer.cpp
int main(int argc, char** argv) {
  // 1. 获得一个ProcessState instance
  sp<ProcessState> proc(ProcessState::self());

  // 2. MS作为ServiceManager的客户端，需要向ServiceManger注册服务
  // 调用defaultServiceManager，得到一个IServiceManager。
  sp<IServiceManager> sm = defaultServiceManager();

  // 初始化音频系统的AudioFlinger服务
  AudioFlinger::instantiate();

  // 3. 多媒体系统的MediaPlayer服务，我们将以它作为主切入点
  MediaPlayerService::instantiate();
 
  // CameraService服务
  CameraService::instantiate();

  // 音频系统的AudioPolicy服务
  AudioPolicyService::instantiate();

  // 4. 创建线程池
  ProcessState::self()->startThreadPool();

  // 5. 将自己加入到刚才的线程池中
  IPCThreadState::self()->joinThreadPool();
}
```
上面的代码中，确定了5个关键点，让我们通过对这5个关键点逐一进行深入分析，来认识和理解`Binder`。

## `ProcessState`和`IPCThreadState`
我们在`main`函数的开始处便碰见了`ProcessState`。在Android中`ProcessState`是客户端和服务端公共的部分，作为`Binder`通信的基础，`ProcessState`是一个`singleton`类（一般通过`ProcessState::self()`获取），**每个进程只有一个对象**，这个对象负责打开**Binder驱动**，**建立线程池**，让其进程里面的所有线程都能通过Binder通信。

与之相关的是`IPCThreadState`，每个线程都有一`个IPCThreadState`实例登记在Linux线程的上下文附属数据中，主要负责`Binder`的读取，写入和请求处理框架。`IPCThreadState`在构造的时候获取进程的`ProcessState`并记录在自己的成员变量`mProcess`中，通过`mProcess`可以获得`Binder`的句柄。

我们将依照`main`函数执行顺序依次深入。

```cpp
// Main_MediaServer.cpp

// 1. 获得一个ProcessState instance
sp<ProcessState> proc(ProcessState::self());
```

下面，来进一步分析这个独一无二的`ProcessState`。

### 单例的ProcessState
`ProcessState`的代码如下所示：

```cpp
// ProcessState.h
class ProcessState : public virtual RefBase
{
  ...
  struct handle_entry {
    IBinder* binder;
    RefBase::weakref_type* refs;
  };
  ...
  // 当前process维护的Service代理对象列表 
  Vector<handle_entry> mHandleToObject; 
  ...
}

// ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
  // gProcess是在Static.cpp中定义的一个全局变量
  // 程序刚开始执行，gProcess一定为空
  if (gProcess != NULL) return gProcess;

  AutoMutex_l(gProcessMutex);

  // 创建一个ProcessState对象，并赋值给gProcess
  if (gProcess == NULL) gProcess = new ProcessState;
  return gProcess;
}
```
`self`函数采用了**单例**模式(singleton class)，**这很明确地告诉了我们一个信息：每个进程只有一个`ProcessState`对象。** 

`ProcessState`为当前进程维护了一个Service代理对象列表（`mHandleToObject`），而客户端可以通过`handle`来查找（`getStrongProxyForHandle`）Service的代理（`BpBinder`）进而和Service通讯。

<img src="http://52.53.208.118/wp-content/uploads/2016/09/Screen-Shot-2016-09-20-at-10.57.17-AM-300x95.png" alt="screen-shot-2016-09-20-at-10-57-17-am" width="600" class="alignnone size-medium wp-image-73" />

### `ProcessState`的构造
再来看`ProcessState`的构造函数。这个函数非常重要，它悄悄地打开了Binder设备。代码如下所示：

```cpp
// ProcessState.cpp
ProcessState::ProcessState()
  : mDriverFD(open_driver()) // 悄悄地打开Binder设备
  , mVMStart(MAP_FAILED) // 映射内存的起始地址
  , mManagesContexts(false)
  , mBinderContextCheckFunc(NULL)
  , mBinderContextUserData(NULL)
  , mThreadPoolStarted(false)
  , mThreadPoolSeq(1)
{
  if (mDriverFD >= 0) {
    // BIDNER_VM_SIZE定义为(1 * 1024 * 1024) - (4096 * 2) = 1M - 8K
    // 将mDriverFD这个文件从0开始的所有内容映射到大小为BIDNER_VM_SIZ的内存区域中，并把该内存的地址返回给mVMStart。
    // mmap the binder, providing a chunk of virtual address space to receive transactions.
    mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
  }
  ...
}
```

### 打开binder设备
`open_driver`的作用就是打开`/dev/binder`这个设备，它是android在内核中专门用于完成进程间通信而设置的一个**虚拟设备**，具体实现如下所示：

```cpp
// ProcessState.cpp
static int open_driver() {
  int fd = open("/dev/binder", O_RDWR); // 打开/dev/binder设备
  if (fd >= 0) {
    ...
    size_t maxThreads = 15;
    // 通过ioctl方式告诉binder驱动，这个fd支持的最大线程数是15个
    result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);  
  }
  return fd;
}
```

至此，`Process::self`函数就分析完了。它到底干了什么呢？通过前面的分析，总结如下：

* 打开`/dev/binder`设备，这就相当于与内核的Binder驱动有了交互的通道。
* 对返回的`fd`使用`mmap`，这样Binder驱动就会**分配一块内存来接收数据**。
* 由于`ProcessState`的惟一性，因此**一个进程只打开设备一次**。

分析完`ProcessState`，接下来将要分析第二个关键函数`defaultServiceManager`。

## 时空穿越魔术——defaultServiceManager
`defaultServiceManager`函数的实现在`IServiceManager.cpp`中完成。它会返回一个`IServiceManager`对象，通过这个对象，我们可以神奇地与**另一个进程**`ServiceManager`进行交互。是不是有一种观看时空穿越魔术表演的感觉？

### 魔术前的准备工作
先来看看`defaultServiceManager`都调用了哪些函数？返回的这个`IServiceManager`到底是什么？具体实现代码如下所示：

```cpp
// IServiceManager.cpp
sp<IServiceManager> defaultServiceManager() {
  if (gDefaultServiceManager != NULL) {
    return gDefaultServiceManager;
  }

  {
    AutoMutex_l(gDefaultServiceManagerLock);
    if (gDefaultServiceManager == NULL) {
      // 真正的gDefaultServiceManager是在这里创建的。
      gDefaultServiceManager = interface_cast<IServiceManager>(
                                 ProcessState::self()->getContextObject(NULL));

    }
  }
  return gDefaultServiceManager;
}
```
哦，是调用了`ProcessState`的`getContextObject`函数！注意：传给它的参数是`NULL`，即`0`。

```cpp
// ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& caller) {
  // caller的值为0！注意，该函数返回的是IBinder。它是什么？我们后面再说。
  // supportsProcesses函数根据openDriver函数打开设备是否成功来判断是否支持process
  // 真实设备肯定支持process。
  if (supportsProcesses()) {
    // 真实设备上肯定是支持进程的，所以会调用下面这个函数
    return getStrongProxyForHandle(0);
  } else {
    return getContextObject(String16("default"), caller);
  }
}
```
`getStrongProxyForHandle`在前文已经提到，是用于根据client提供的handle（其实就是索引值）得到相应Service的代理对象`BpBinder`。

```cpp
// ProcessState.cpp
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle) {
  sp<IBinder> result;
  AutoMutex_l(mLock);

  // 根据索引查找对应资源。如果lookupHandleLocked发现没有对应的资源项，则会创建一个新的项并返回。
  // 这个新项的内容需要填充。
  handle_entry* e = lookupHandleLocked(handle);
  if (e != NULL) {
    IBinder* b = e->binder;
    if (b == NULL || !e->refs->attemptIncWeak(this)) {
      // 对于新创建的资源项，它的binder为空，所以走这个分支。注意，handle的值为0
      b = new BpBinder(handle); // 创建一个BpBinder
      e->binder = b; // 填充entry的内容
      if (b) {
        e->refs = b->getWeakRefs();
      }
      result = b;
    } else {
      result.force_set(b);
      e->refs->decWeak(this);
    }
  }
  return result; // 返回BpBinder(handle)，注意，handle的值为0
}

// 在mHandleToObject表中查找entry。
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
  const size_t N = mHandleToObject.size();
  if (N <= (size_t)handle) {
    handle_entry e;
    e.binder = NULL;
    e.refs = NULL;
    status_t err = mHandleToObject.insertAt(e, N, handle + 1 - N);
    if (err < NO_ERROR) return NULL;
  }
  return &mHandleToObject.editItemAt(handle);
}
```
### 魔术表演的道具 —— `BpBinder` 和 `BBinder`
`BpBinder`和`BBinder`都是Android中与Binder通信相关的代表，它们都从`IBinder`类中派生而来。

* `BpBinder`是客户端用来与Server交互的代理类，`B`即Binder, `p`即Proxy。
* `BBinder`则是proxy相对的一端，它是**proxy交互的目的端**。如果说Proxy代表客户端，那么`BBinder`则代表服务端。这里的`BpBinder`和`BBinder`是一一对应的，即**某个`BpBinder`只能和对应的`BBinder`交互**。我们当然不希望通过`BpBinderA`发送的请求，却由`BBinderB`来处理。

刚才我们在`defaultServiceManager()`函数中创建了这个`BpBinder`。这里有两个问题：

1. 为什么创建的不是`BBinder`？
  > 因为我们是ServiceManager的客户端，当然得使用代理端以与ServiceManager交互了。

2.  前面说了，`BpBinder`和`BBinder`是一一对应的，那么`BpBinder`如何标识它所对应的`BBinder`端呢？ 
  > 答案是Binder系统通过`handle`来对应BBinder。以后我们会确认这个`handle`值的作用。

_注意：我们给`BpBinder`构造函数传的参数`handle`的值是0。这个0在整个Binder系统中有重要含义--因为0代表的就是`ServiceManager`所对应的`BBinder`。_

`BpBinder`是如此重要，必须对它进行深入分析，其代码如下所示:

```cpp
// BpBinder.cpp
BpBinder::BpBinder(int32_t handle)
    : mHandle(handle) // handle是0
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
{
  extendObjectLifetime(OBJECT_LIFETIME_WEAK);

  // 另一个重要对象是IPCThreadState，我们稍后会详细讲解。
  IPCThreadState::self()->incWeakHandle(handle);
}
```
看上面的代码，会觉得`BpBinder`确实简单，不过再仔细查看，你或许会发现，`BpBinder`、`BBinder`这两个类没有任何地方操作`ProcessState`打开的那个`/dev/binder`设备，换言之，**这两个`Binder`类没有和binder设备直接交互**。那为什么说`BpBinder`会与通信相关呢？注意本小节的标题，`BpBinder`只是道具嘛！所以它后面一定还另有机关。不必急着揭秘，还是先回顾一下道具出场的历程。

我们是从下面这个函数开始分析的：

```cpp
gDefaultServiceManager = interface_cast<IServiceManager>(
                             ProcessState::self()->getContextObject(NULL));

```
现在这个函数调用将变成如下所示（代入`getContextObject(NULL)`的返回值）：

```cpp
gDefaultServiceManager = interface_cast<IServiceManager>(new BpBinder(0));
```
这里出现了一个`interface_cast`。它是什么？其实是一个障眼法！下面就来具体分析它。

### 障眼法——`interface_cast`

```cpp
// IInterface.h
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj) {
  return INTERFACE::asInterface(obj);
}
```

哦，仅仅是一个模板函数，所以`interface_cast<IServiceManager>()`等价于：

```cpp
inline sp<IServiceManager>interface_cast(const sp<IBinder>& obj) {
  return IServiceManager::asInterface(obj);
}
```

又转移到`IServiceManager`对象中去了，这难道不是障眼法吗？既然找到了“真身”，不妨就来见识见识它吧。

### 拨开浮云见月明——`IServiceManager`
刚才提到，`IBinder`家族的`BpBinder`和`BBinder`是与通信业务相关的，那么业务层的逻辑又是如何巧妙地架构在`Binder`机制上的呢？关于这些问题，可以用一个绝好的例子来解释，它就是`IServiceManager`。

#### 定义业务逻辑
先回答第一个问题：**如何表述应用的业务层逻辑。** 可以先分析一下`IServiceManager`是怎么做的。`IServiceManager`定义了`ServiceManager`所提供的服务，看它的定义可知，其中有很多有趣的内容。

```cpp
// IServiceManager.h
class IServiceManager : public IInterface {
public:
  // 关键无比的宏！
  DECLARE_META_INTERFACE(ServiceManager);

  // 下面是ServiceManager所提供的业务函数
  virtual sp<IBinder> getService(const String16& name) const = 0;
  virtual sp<IBinder> checkService(const String16& name) const = 0;
  virtual status_t addService(const String16& name, const sp<IBinder>& service) = 0;
  virtual Vector<String16> listServices() = 0;
  ...
};
```

#### 业务与通信的挂钩
Android巧妙地通过`DECLARE_META_INTERFACE`和`IMPLENT`宏，将业务和通信牢牢地钩在了一起。`DECLARE_META_INTERFACE`和`IMPLEMENT_META_INTERFACE`这两个宏都定义在刚才的`IInterface.h`中。先看`DECLARE_META_INTERFACE`这个宏，如下所示：

```cpp
// IInterface.h::DECLARE_META_INTERFACE
#define DECLARE_META_INTERFACE(INTERFACE)                               \
    static const android::String16 descriptor;                          \
    static android::sp<I##INTERFACE> asInterface(                       \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();    
```

将`IServiceManager`的`DELCARE`宏进行相应的替换后得到的代码如下所示：

```cpp
// DECLARE_META_INTERFACE(IServiceManager)

// 定义一个描述字符串
static const android::String16 descriptor;

// 定义一个asInterface函数
static android::sp<IServiceManager> asInterface(
    const android::sp<android::IBinder>& obj)

// 定义一个getInterfaceDescriptor函数，估计就是返回descriptor字符串
virtual const android::String16& getInterfaceDescriptor() const;

// 定义IServiceManager的构造函数和析构函数
IServiceManager();                                                   
virtual ~IServiceManager();
```
`DECLARE`宏声明了一些函数和一个变量，那么，`IMPLEMENT`宏的作用肯定就是定义它们了。`IMPLEMENT`的定义在`IInterface.h`中，`IServiceManager`是如何使用了这个宏呢？只有一行代码，在`IServiceManager.cpp`中，如下所示：

```cpp
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");
```

很简单，可直接将`IServiceManager`中的`IMPLEMENT`宏的定义展开，如下所示：

```cpp
const android::String16 IServiceManager::descriptor(“android.os.IServiceManager”);

const android::String16& IServiceManager::getInterfaceDescriptor() const { 
  // 返回字符串descriptor，值是“android.os.IServiceManager”
  return IServiceManager::descriptor;
}    

android::sp<IServiceManager>
    IServiceManager::asInterface(const android::sp<android::IBinder>& obj) {
  android::sp<IServiceManager> intr;
  if (obj != NULL) {                                              
    intr = static_cast<IServiceManager *>(                         
               obj->queryLocalInterface(IServiceManager::descriptor).get());  
    if (intr == NULL) {
      // obj是我们刚才创建的那个BpBinder(0)
      intr = new BpServiceManager(obj);
    }
  }
  return intr;
}

IServiceManager::IServiceManager () {}

IServiceManager::~ IServiceManager() {}
```

`interface_cast`是如何把`BpBinder`指针转换成一个`IServiceManager`指针的呢？答案就在`asInterface`函数的一行代码中，如下所示：

```cpp
intr = new BpServiceManager(obj);
```
**`interface_cast`不是指针的转换，而是利用`BpBinder`对象作为参数新建了一个`BpServiceManager`对象。**我们已经知道`BpBinder`和`BBinder`与通信有关系，这里怎么突然冒出来一个`BpServiceManager`？它们之间又有什么关系呢？

#### `IServiceManager`家族
要搞清这个问题，必须先了解`IServiceManager`家族之间的关系，先来看图6-3，它展示了`IServiceManager`的家族图谱。

<img src="http://52.53.208.118/wp-content/uploads/2016/09/Screen-Shot-2016-09-20-at-9.01.26-AM-300x276.png" alt="screen-shot-2016-09-20-at-9-01-26-am" width="400" class="alignnone size-medium wp-image-66" />

* `IServiceManager`、`BpServiceManager`和`BnServiceManager`都与业务逻辑相关。
* `BnServiceManager`从`BBinder`派生，表示它可以直接参与`Binder`通信。
* `BnServiceManager`是一个虚类，它的业务函数最终需要子类来实现。

_重要说明：以上这些关系很复杂，但`ServiceManager`并没有使用错综复杂的派生关系，它直接打开`Binder`设备并与之交互。后文，还会详细分析它的实现代码。_

`BpServiceManager`就是SM的`Binder`代理。既然是代理，那肯定希望对用户是透明的，那就是说头文件里边不会有这个Bp的定义。果然，`BpServiceManager`就在刚才的`IServiceManager.cpp`中定义。`BpServiceManager`，既然不像它的兄弟`BnServiceManager`那样直接与`Binder`有血缘关系，那么它又是如何与`Binder`交互的呢？
> 简言之，`BpRefBase`中的`mRemote`的值就是`BpBinder`。

```cpp
class BpServiceManager : public BpInterface<IServiceManager> {
public:

// 这里传入的impl就是new BpBinder(0)
BpServiceManager(const sp<IBinder>& impl) : BpInterface<IServiceManager>(impl) {}

virtual status_t addService(const String16& name, const sp<IBinder>& service) {
  ...     
}
```

`BpInterface<IServiceManager>`的构造函数如下：

```cpp
inline BpInterface<IServiceManager>::BpInterface(const sp<IBinder>& remote) 
    : BpRefBase(remote) {}

BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(NULL), mState(0) {

  // mRemote就是刚才的BpBinder(0)
   ...
}
```

原来，`BpServiceManager`的一个变量`mRemote`是指向了`BpBinder`。至此，我们的魔术表演完了，回想一下`defaultServiceManager`函数，可以得到以下两个关键对象：

* 有一个`BpBinder`对象，它的`handle`值是0。
* 有一个`BpServiceManager`对象，它的`mRemote`值是`BpBinder`。

`BpServiceManager`对象实现了`IServiceManager`的业务函数，现在又有`BpBinder`作为通信的代表，接下来的工作就简单了。下面，要通过分析`MediaPlayerService`的注册过程，进一步分析业务函数的内部是如何工作的。

## 注册`MediaPlayerService`

### 业务层的工作
回到MediaPlayer的`main`函数：

```cpp
// MediaPlayerService.cpp
void MediaPlayerService::instantiate() {
　// defaultServiceManager就是上面所述得到的BpServiceManager对象
　defaultServiceManager()->addService(
　　　String16("media.player"), new MediaPlayerService());
}
```
根据前面的分析，`defaultServiceManager()`实际返回的对象是`BpServiceManager`，它是`IServiceManager`的后代，代码如下所示：

```cpp
// IServiceManager.cpp::BpServiceManager的addService()函数
virtual status_t addService(const String16& name, const sp<IBinder>& service) {
  // 生成数据包Parcel
  Parcel data, reply;
  data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
  data.writeString16(name);
  data.writeStrongBinder(service);

  // remote()返回的是mRemote，也就是BpBinder对象
  status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
  retur nerr == NO_ERROR ? reply.readInt32() : err;
}
```
别急着往下走，应先思考以下两个问题：

* 调用`BpServiceManager`的`addService`是不是一个业务层的函数？
* `addService`函数中把**请求数据**打包成`data`后，传给了`BpBinder`的`transact`函数，这是不是把通信的工作交给了`BpBinder`？
两个问题的答案都是肯定的。至此，**业务层**的工作原理应该是很清晰了，它的作用就是将请求信息打包后，再交给**通信层**去处理。

### 通信层的工作
`remote()->transact`到`BpBinder`中：

```cpp
status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
  if (mAlive) {
    // BpBinder果然是道具，它把transact工作交给了IPCThreadState
    // 到当前进程IPCThreadState中 mHandle == 0
    status_t status = IPCThreadState::self()->transact(mHandle, code, data, reply, flags);
    if (status == DEAD_OBJECT) mAlive = 0;
      return status;
  }
  return DEAD_OBJECT;
}
```

#### “劳者一份”的`IPCThreadState`
谁是“劳者”？线程，是进程中真正干活的伙计，所以它正是劳者。而“劳者一份”，就是每个伙计一份的意思。

```cpp
// IPCThreadState.cpp
IPCThreadState* IPCThreadState::self() {
  // TLS是Thread Local Storage（线程本地存储空间）的简称。
  if (gHaveTLS) { // 第一次进来为false
restart:
    const pthread_key_t k = gTLS;
    // 从线程本地存储空间中获得保存在其中的IPCThreadState对象。
    // 有调用pthread_getspecific的地方，肯定也有调用pthread_setspecific的地方
    IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
    if (st) return st;
    return new IPCThreadState;
  }

  if (gShutdown) return NULL;

  pthread_mutex_lock(&gTLSMutex);
  if (!gHaveTLS) {
    if (pthread_key_create(&gTLS, threadDestructor) != 0) {
      pthread_mutex_unlock(&gTLSMutex);
      return NULL;
    }
    gHaveTLS = true;
  }

  pthread_mutex_unlock(&gTLSMutex);
  goto restart;
}
```
接下来，有必要转向分析它的构造函数`IPCThreadState()`，如下所示：

```cpp
// IPCThreadState.cpp
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self())
    , mMyThreadId(androidGetTid())
{
  // 在构造函数中，把自己设置到线程本地存储中去。
  pthread_setspecific(gTLS, this);
  clearCaller();

  // mIn和mOut是两个Parcel。把它看成是发送和接收命令的缓冲区即可。
  mIn.setDataCapacity(256);
  mOut.setDataCapacity(256);
}
```
**每个线程都有一个`IPCThreadState`**，每个`IPCThreadState`中都有一个`mIn`、一个`mOut`，其中

* `mIn`是用来接收来自`Binder`设备的数据的
* `mOut`则是用来存储发往`Binder`设备的数据的。

#### 勤劳的`transact`
传输工作是很辛苦的。我们刚才看到`BpBinder`的`transact`调用了`IPCThreadState`的`transact`函数，这个函数实际完成了与`Binder`通信的工作，如下面的代码所示：

```cpp
// IPCThreadState.cpp
// 注意，handle的值为0，代表了通信的目的端
status_t IPCThreadState::transact( 
    int32_t handle,
    uint32_tcode, 
    const Parcel& data,
    Parcel* reply, uint32_t flags) 
{
  status_t err = data.errorCheck();
  flags |=TF_ACCEPT_FDS;
  ...
  // 调用writeTransactionData 发送数据
  // 注意这里的第一个参数BC_TRANSACTION，它是应用程序向binder设备发送消息的消息码，
  // 而binder设备向应用程序回复消息的消息码以BR_开头。消息码的定义在binder_module.h中，
  // 请求消息码和回应消息码的对应关系，需要查看Binder驱动的实现才能将其理清楚，我们这里暂时用不上。
  err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
  ...
  // 等回复
  err = waitForResponse(reply);
  ...
  return err;
}
```
多熟悉的流程：先发数据，然后等结果。再简单不过了！不过，我们有必要确认一下`handle`这个参数到底起了什么作用。先来看`writeTransactionData`函数，它的实现如下所示：

```cpp
// IPCThreadState.cpp
status_t IPCThreadState::writeTransactionData(
    int32_t cmd, 
    uint32_t binderFlags,
    int32_t handle, 
    uint32_t code, 
    const Parcel& data, 
    status_t* statusBuffer)
{
  // binder_transaction_data 是和binder设备通信的数据结构。 
  // 以下binder_transaction_data，然后写到mOut中，
  // mOut是命令的缓冲区，也是一个Parcel  
  binder_transaction_data tr;

  // handle的值传递给了target，用来标识目的端，其中0是ServiceManager的标志。
  tr.target.handle = handle;

  // code是消息码，用来switch/case的！
  tr.code = code;
  tr.flags = binderFlags;

  const status_t err = data.errorCheck();
  if (err == NO_ERROR) {
    tr.data_size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    tr.offsets_size = data.ipcObjectsCount() * sizeof(size_t);
    tr.data.ptr.offsets = data.ipcObjects();
  } else if (statusBuffer) {
    tr.flags |= TF_STATUS_CODE;
    *statusBuffer = err;
    tr.data_size = sizeof(status_t);
    tr.data.ptr.buffer = statusBuffer;
    tr.offsets_size = 0;
    tr.data.ptr.offsets = NULL;
  } else {
    return (mLastError = err);
  }

  // 把命令写到mOut中， 而不是直接发出去，可见这个函数有点名不副实。
  mOut.writeInt32(cmd);
  mOut.write(&tr, sizeof(tr));
  return NO_ERROR;
}
```
现在，已经把`addService`的请求信息写到`mOut`中了。接下来再看**发送请求**和**接收回复**部分的实现，代码在`waitForResponse`函数中，如下所示：

```cpp
// IPCThreadState.cpp
status_t IPCThreadState::waitForResponse(Parcel* reply, status_t* acquireResult)
{
  int32_t cmd;
  int32_t err;

  while (1) {
    if ((err = talkWithDriver()) < NO_ERROR) break;

    err = mIn.errorCheck();
    if (err < NO_ERROR) break;
    if (mIn.dataAvail() == 0) continue;

    // 把mOut发出去，然后从driver中读到数据放到mIn中了。
    cmd = mIn.readInt32();
    switch (cmd) {
      case BR_TRANSACTION_COMPLETE:
        if (!reply && !acquireResult) goto finish;
        break;
        ...
      default:
        err = executeCommand(cmd); // 看这个！
        if (err != NO_ERROR) goto finish;
        break;
    }
  }
finish:
  if (err!= NO_ERROR) {
    if (acquireResult) *acquireResult = err;
    if (reply) reply->setError(err);

    mLastError = err;
  }
  return err;
}
```
现在，我们已发送了请求数据，假设马上就收到了回复，后续该怎么处理呢？来看`executeCommand`函数，如下所示：

```cpp
// IPCThreadState.cpp
status_t IPCThreadState::executeCommand(int32_tcmd) 
{
  BBinder* obj;
  RefBase::weakref_type* refs;
  status_tresult = NO_ERROR;

  switch(cmd) {
    case BR_ERROR:
      result = mIn.readInt32();
      break;
      ...
    case BR_TRANSACTION:
    {
      binder_transaction_data tr;
      result = mIn.read(&tr, sizeof(tr));
      if (result != NO_ERROR) break;

      Parcel buffer;
      Parcel reply;
      if (tr.target.ptr) {
        // 看到了BBinder，想起图6-3了吗？BnServiceXXX从BBinder派生，
        // 这里的b实际上就是实现BnServiceXXX的那个对象，关于它的作用，我们要在6.5节中讲解。
        sp<BBinder> b((BBinder*)tr.cookie);
        const status_t error = b->transact(tr.code, buffer, &reply, 0);
        if (error < NO_ERROR) reply.setError(error);
      } else {
        // the_context_object是IPCThreadState.cpp中定义的一个全局变量，
        // 可通过setTheContextObject函数设置
        const status_t error = the_context_object->transact(tr.code,buffer, &reply, 0);
        if (error < NO_ERROR) reply.setError(error);
      } 
    } break;
    ...
    case BR_DEAD_BINDER:
    {
      // 收到binder驱动发来的service死掉的消息，看来只有Bp端能收到了
      BpBinder *proxy = (BpBinder*)mIn.readInt32();
      proxy->sendObituary();
      mOut.writeInt32(BC_DEAD_BINDER_DONE);
      mOut.writeInt32((int32_t)proxy);
    } break;
    ...
    case BR_SPAWN_LOOPER:
      // 特别注意，这里将收到来自驱动的指示以创建一个新线程，用于和Binder通信。
      mProcess->spawnPooledThread(false);
      break;
    default:
      result = UNKNOWN_ERROR;
      break;
  }
  ...
  if (result != NO_ERROR) {
    mLastError = result;
  }
  return result;
}
```

#### `talkwithDriver`

```cpp
// IPCThreadState.cpp
status_t IPCThreadState::talkWithDriver(booldoReceive)
{
  // binder_write_read是用来与Binder设备交换数据的结构
  // 本质上是把mOut数据和mIn接收数据的处理后赋值给bwr
  binder_write_read bwr;

  const bool needRead = mIn.dataPosition() >= mIn.dataSize();
  const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

  // 请求命令的填充
  bwr.write_size = outAvail;
  bwr.write_buffer = (long unsigned int)mOut.data();

  if (doReceive && needRead) {
    // 接收数据缓冲区信息的填充。如果以后收到数据，就直接填在mIn中了。
    bwr.read_size = mIn.dataCapacity();
    bwr.read_buffer = (long unsigned int)mIn.data();
  } else {
    bwr.read_size = 0;
  }

  if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

  bwr.write_consumed = 0;
  bwr.read_consumed = 0;
  status_terr;

  do {
#ifdefined(HAVE_ANDROID_OS)
    // 用ioctl来读写
    if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
      err = NO_ERROR;
    else
      err = -errno;
#else
    err = INVALID_OPERATION;
#endif
  } while (err == -EINTR);

  if (err>= NO_ERROR) {
    if (bwr.write_consumed > 0) {
      if (bwr.write_consumed < (ssize_t)mOut.dataSize())
        mOut.remove(0, bwr.write_consumed);
      else
        mOut.setDataSize(0);
    }

    if (bwr.read_consumed > 0) {
      mIn.setDataSize(bwr.read_consumed);
      mIn.setDataPosition(0);
    }

    return NO_ERROR;
  }
  return err;
}
```

较为深入地分析了MediaPlayerService的注册过程后，下面还剩最后两个函数了，就让我们向它们发起进攻吧！

## `startThreadPool`和`joinThreadPool`
### `startThreadPool`

```cpp
// ProcessState.cpp
void ProcessState::startThreadPool()
{
  AutoMutex_l(mLock);
  // 如果要是已经startThreadPool的话，这个函数就没有什么实质作用了
  if (!mThreadPoolStarted) {
    mThreadPoolStarted = true;
    spawnPooledThread(true); // 注意，传进去的参数是true
  }
}
```
`spawnPooledThread`函数的实现如下：

```cpp
// ProcessState.cpp
void ProcessState::spawnPooledThread(bool isMain)
{
  // 注意，isMain参数是true。
  if (mThreadPoolStarted) {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    char buf[32];
    sprintf(buf, "Binder Thread #%d", s);
    sp<Thread> t = new PoolThread(isMain);
    // 在run函数中创建线程
    t->run(buf);
  }
}
```
`PoolThread`是在`IPCThreadState`中定义的一个`Thread`子类：

```cpp
// IPCThreadState.h::PoolThread类
class PoolThread : public Thread
{
public:
  PoolThread(bool isMain) : mIsMain(isMain){}

protected:
  virtual bool threadLoop()
  {
    // 线程函数如此简单，不过是在这个新线程中又创建了一个IPCThreadState。
    // 你还记得它是每个伙计都有一个的吗？
    IPCThreadState::self()->joinThreadPool(mIsMain);
    return false;
  }

  const bool mIsMain;
};

// Thread
Thread::Thread(bool canCallJava) // canCallJava默认值是true
    : mCanCallJava(canCallJava)
    , mThread(thread_id_t(-1))
    , mLock("Thread::mLock")
    , mStatus(NO_ERROR)
    , mExitPending(false)
    , mRunning(false)
{
}

status_t Thread::run(const char* name, int32_t priority, size_t stack)
{
  bool res;
  if (mCanCallJava) {
    res = createThreadEtc(_threadLoop, // 线程函数是_threadLoop
              this, name, priority, stack, &mThread);
  }
  ...
}
```

### `joinThreadPool`
还需要看看`IPCThreadState`的`joinThreadPool`的实现，因为新创建的线程也会调用这个函数，具体代码如下所示：

```cpp
// IPCThreadState.cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
  // 注意，如果isMain为true，我们需要循环处理。把请求信息写到mOut中，待会儿一起发出去
  mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
  androidSetThreadSchedulingGroup(mMyThreadId, ANDROID_TGROUP_DEFAULT);

  status_t result;
  do {
    int32_t cmd;
    if (mIn.dataPosition() >= mIn.dataSize()) {
      size_t numPending = mPendingWeakDerefs.size();
      if (numPending > 0) {
        for (size_t i = 0; i < numPending; i++) {
           RefBase::weakref_type* refs = mPendingWeakDerefs[i];
           refs->decWeak(mProcess.get());
        }
        mPendingWeakDerefs.clear();
      }

      // 处理已经死亡的BBinder对象
      numPending = mPendingStrongDerefs.size();
      if (numPending > 0) {
        for (size_t i = 0; i < numPending; i++) {
          BBinder* obj = mPendingStrongDerefs[i];
          obj->decStrong(mProcess.get());
        }
        mPendingStrongDerefs.clear();
      }
    }
    // talkWithDriver读取client端发送的请求
    result = talkWithDriver();
    if (result >= NO_ERROR) {
      size_t IN = mIn.dataAvail();
      if (IN < sizeof(int32_t)) continue;

      cmd = mIn.readInt32();
      result = executeCommand(cmd); // 调用executeCommand处理消息
    }
    ...
  } while(result != -ECONNREFUSED && result != -EBADF);

  mOut.writeInt32(BC_EXIT_LOOPER);
  talkWithDriver(false);
}
```

### 有几个线程在服务?
到底有多少个线程在为Service服务呢？目前看来是两个：

* `startThreadPool`中新启动的线程通过`joinThreadPool`读取`Binder`设备，查看是否有请求。
* 主线程也调用`joinThreadPool`读取`Binder`设备，查看是否有请求。看来，binder设备是支持多线程操作的，其中一定是做了同步方面的工作。

MediaServer这个进程一共注册了4个服务，繁忙的时候，两个线程会不会显得有点少呢？另外，如果实现的服务负担不是很重，完全可以不调用`startThreadPool`创建新的线程，使用主线程即可胜任。

## 小结
我们以MediaServer为例，分析了`Binder`的机制，这里还是有必要再次强调一下`Binder`通信和基于Binder通信的业务之间的关系。

* `Binder`是通信机制。
* 业务可以基于`Binder`通信，当然也可以使用别的IPC方式通信。
