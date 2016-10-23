# Generator

## An example
Javascript generator has been added to ES6 while back in ES5, had facebook already released a generic, state-machine based `regenerator` module that did the same work. Generator is also the backbone of `async` syntax support. Let's start with an example.

```javascript
// declare a generator with '*'
function *askFoo() {
  yield 'foo';
}

function *askFooAndBars() {
  yield *askFoo(); // delegate to another generator
  yield 'bar1';
  yield 'bar2';
}

var it = askFooAndBars(); // initialize a generator
for (var result = it.next(); !result.done; result = it.next()) {
  // 'it.next()' returns an object of shape '{value: any, done: boolean}'
  console.log(result);
}
// stdout:
// { value: 'foo', done: false }
// { value: 'bar1', done: false }
// { value: 'bar2', done: false }

// Babel transform
var _marked = [askFoo, askFooAndBars].map(regeneratorRuntime.mark);

function askFoo() {
  return regeneratorRuntime.wrap(function askFoo$(_context) {
    while (1) {
      switch (_context.prev = _context.next) {
        case 0:
          _context.next = 2;
          return 'foo';

        case 2:
        case 'end':
          return _context.stop();
      }
    }
  }, _marked[0], this);
}

function askFooAndBars() {
  return regeneratorRuntime.wrap(function askFooAndBars$(_context2) {
    while (1) {
      // We meet state-machine again!
      switch (_context2.prev = _context2.next) {
        case 0:
          return _context2.delegateYield(askFoo(), 't0', 1);

        case 1:
          _context2.next = 3;
          return 'bar1';

        case 3:
          _context2.next = 5;
          return 'bar2';

        case 5:
        case 'end':
          return _context2.stop();
      }
    }
  }, _marked[1], this);
}
...
```
Basically, a generator function is a normal function wrapped in a state machine. Things are getting more interesting when we send back a 'value' to the state machine and tell it to resume with a new state.

```javascript
function *askFooAndBars() {
  var bar = yield 'foo'
  yield bar;
}

var it = askFooAndBars();
console.log(it.next());
console.log(it.next());
// stdout:
// Object {
//   "done": false,
//   "value": "foo"
// }
// Object {
//   "done": false,
//   "value": undefined
// }

// Babel transform
var _marked = [askFooAndBars].map(regeneratorRuntime.mark);

function askFooAndBars() {
  var bar;
  return regeneratorRuntime.wrap(function askFooAndBars$(_context) {
    while (1) {
      switch (_context.prev = _context.next) {
        case 0:
          _context.next = 2;
          return 'foo';

        case 2:
          // note that bar get its value from the 'context'.
          bar = _context.sent;
          _context.next = 5;
          return bar;

        case 5:
        case 'end':
          return _context.stop();
      }
    }
  }, _marked[0], this);
}
```

Actually, when `it.next()` is called the first time, it will stop right before the assignment of `var bar` in `askFooAndBars`. After `it.next(arg)` is called the second time (with `arg = undefined` in our example), `arg` will be assigned to `boo` and that's why the iterator will give `undefined` instead of something else.

```javascript
var it = askFooAndBars();
console.log(it.next());
console.log(it.next('bar')); // set 'context.sent = 'bar'

// stdout:
// Object {
//   "done": false,
//   "value": "foo"
// }
// Object {
//   "done": false,
//   "value": "bar"
// }
```

## Chain Async function Calls without Pain

```javascript
// Some async function
function asyncIdentity(x, callback) {
  // There is some problem without setTimeout. Not a JS expert ┐(´д`)┌
  setTimeout(function() {
    callback(null, x);
  }, 10);
}

// A function that chain async function calls
function* printSomething(done) {
  var x1 = yield asyncIdentity('x1', done);
  console.log(x1);
  var x2 = yield asyncIdentity('x2', done);
  console.log(x2);
  var x3 = yield asyncIdentity('x3', done);
  console.log(x3);
}

// The dumb way to execute LOOP the generator function `printSomething`
var it = printSomething(done);
function done(err, result) {
  if (err) {
    console.log('error ', error);
    return;
  }
  // recursively iterate the generator
  it.next(result);
}

// start the iteration with initial state
it.next();
```

An improved version
```javascript
function run(generator) {
  var it = generator(done);

  function done(err, result) {
    if (err) {
      console.log('error ', error);
      return;
    }
    // recursively iterate the generator
    it.next(result);
  }
  it.next(); // or 'done()'
}

run(printSomething);
```

## Back to `async` and `await`
The way we write async function calls in `printSomething` is almost identical to how write with `await` and `async` using `Promise`. Yes, we could easily implement `async` and `await` in terms of generator.

```javascript
function identityPromise(x) {
  return Promise.resolve(x);
}

// Use promise
async function printSomethingPromise() {
  var x1 = await identityPromise('x1');
  console.log(x1);
  var x2 = await identityPromise('x2');
  console.log(x2);
  var x3 = await identityPromise('x3');
  console.log(x3);
}
printSomethingPromise();

// Use generator
function* printSomethingGenerator() {
  var x1 = yield identityPromise('x1');
  console.log(x1);
  var x2 = yield identityPromise('x2');
  console.log(x2);
  var x3 = yield identityPromise('x3');
  console.log(x3);
}

function run (generator) {
  var it = generator();

  // Now the generator yield a promise
  function go(result) {
    if (result.done) {
      return result.value;
    }
    return result.value.then(function (value) {
      return go(it.next(value));
    }, function (error) {
      return go(it.throw(error));
    });
  }
  go(it.next());
}

run(printSomethingGenerator);
```

## References
* http://x-team.com/2015/04/generators-work/
* http://chrisbuttery.com/articles/synchronous-asynchronous-javascript-with-es6-generators/
* https://nodeschool.io/#workshoppers
