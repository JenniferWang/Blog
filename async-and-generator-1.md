# Async and Generator -- `async` keyword and `Promise`

In JS, `async` keyword provides an easy mechanism to write a `Promise`. But how does it work? What is the overhead? Let's look at an easy example.

```javascript
// How you write with 'async'
async function getFoo() {
  return 'foo'
}

// Write async function in a sync manner
async function waitFooAndGetBar() {
  const foo = await getFoo(); // wait until the promise is resolved
  return 'bar'
}

// Babel transform.
var getFoo = function () {
  var _ref = _asyncToGenerator(regeneratorRuntime.mark(function _callee() {
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            return _context.abrupt('return', 'foo');

          case 1:
          case 'end':
            return _context.stop();
        }
      }
    }, _callee, this);
  }));

  return function getFoo() {
    return _ref.apply(this, arguments);
  };
}();

var waitFooAndGetBar = function () {
  var _ref2 = _asyncToGenerator(regeneratorRuntime.mark(function _callee2() {
    // 'foo' is in the outer closure of the generator wrapper.
    var foo;
    return regeneratorRuntime.wrap(function _callee2$(_context2) {
      // This is a kind of state machine
      while (1) {
        // Assign next state to previous state
        switch (_context2.prev = _context2.next) {
          case 0:
            // initialize and proceed to next state
            _context2.next = 2;
            return getFoo();

          case 2:
            // get back the value from the resolved promise
            foo = _context2.sent;
            // jump out of the loop
            return _context2.abrupt('return', 'bar');

          case 4:
          case 'end':
            return _context2.stop();
        }
      }
    }, _callee2, this);
  }));

  return function waitFooAndGetBar() {
    return _ref2.apply(this, arguments);
  };
}();

function _asyncToGenerator(fn) {
  // This essentially returns a thunk -- in Javascript, it is represented as a
  // function with no arguments that wraps an action.
  return function () {
    // Initialize a generator/iterator which has two methods -- 'next' and
    // 'throw'
    var gen = fn.apply(this, arguments);
    return new Promise(function (resolve, reject) {
      // 'key' = 'next' | 'throw'
      // 'arg' is used in 'gen.next(arg)', see the 'Generator' section.
      function step(key, arg) {
        try {
          // 'info' is an object that returns by an iterator. It is in the shape
          // of '{value: any, done: boolean}';
          var info = gen[key](arg);
          var value = info.value;
        } catch (error) {
          reject(error);
          return;
        }
        if (info.done) {
          // if the iterator reaches end
          resolve(value);
        } else {
          // if the iterator has next... recursively call 'step'
          return Promise.resolve(value).then(
            function (value) {
              step("next", value);
            }, function (err) {
              step("throw", err);
            }
          );
        }
      }
      // Equivalent to call 'gen.next()', starts the iteration with empty 'arg'.
      return step("next");
    });
  };
}
```
The above transformation gives us a clue how `async` keyword does the magic to allow us write asynchronous function call in a synchronous way -- it wraps the function into a generator that support 'pause in execution'.
