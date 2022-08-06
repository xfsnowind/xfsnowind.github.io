---
title: "Promise"
date: 2022-07-28T22:03:24+02:00
author: "Feng Xue"
draft: true
---

Promise is a general concept to handle the asynchronize development in javascript. It's not hard to use, but when it comes with some other concepts, like react's hook, its internal chain etc. It always takes me some time to think through it. Especially when check the code of some open-source libraries, find the way they use Promise is quite fancy and also hard to understand, which reminders me that I am still stay on the level of using, far away from deep understanding.

Inspired by [this blog](https://developpaper.com/source-code-implementation-of-promise-perfect-in-accordance-with-promise-a-specification/), I think it's a good idea to write promise by myself. It's not only because it's not that hard and complex, it can also help me in the future work when I meet it again. Thesedays, function programing is quite popular in js development, but object-orient coding is still used in lots of scenarioes. So I would like to try to implement it with both function and class, even though javascript's prototype is almost equal to class concept, which was introduced in ES2015. It can also train these basic skills again. I would deploy them in my github and write blogs to record it.

Thanks to [promise/A+](https://promisesaplus.com/), we get the requirement analysis, clear logic and [library](https://github.com/promises-aplus/promises-tests) to test the solution.

## Steps

According to the Promise/A+'s requirements, I think it would be four steps:

1. Basic `then` function -- 1.1, 1.2
2. Fulfill `Promise` and `then` parameters -- from 2.1 to 2.2.5
3. Return a `Promise` in `then` -- 2.2.6, 2.2.7
4. `resolve` function -- 2.3

## Basic `thenable`

According to the Terminology, the first two are `promise` and `thenable`. The former should take a function (an `executor`) as parameter which would take two functions (`resolve` and `reject`) as parameters as well. Check [Higher-order functions](https://en.wikipedia.org/wiki/Higher-order_function). These `resolve` and `reject` would be used in the `executor` and defined in the `then`'s parameters. Let's simply implement this `thenable` first.

```javascript
function Promise(executor) {
  let self = this;
  self.executor = executor;
}

Promise.prototype.then = function (onFulfilled, onRejected) {
  let self = this;
  self.executor(onFulfilled, onRejected);
};
```

As we see, we just delay the execution of `executor` from `Promise` to `then` and the fulfill and reject functions are used in `executor` while defined or passed in `then`.

We can use below code to test it. **_NB_**: do not use arrow function in definition, because arrow function does not own `this`. If we use `this` inside of it, it would refer to the outer function. Check [here](https://javascript.info/object-methods#arrow-functions-have-no-this).

```javascript
const a = new Promise((resolve) => setTimeout(() => resolve("result"), 100));

a.then((data) => console.log("Data", data));
// console.log -> Data: result
```

But obviously here, we only have one `executor` and the parameters are not validated. Besides `executor` runs in the `then`, not in the `Promise`. Currently, the code is simple, but it would cause problems. We will mention this below, let's go further.

## Fulfill `Promise` and `then` parameters

### Validate `then` parameters

> 2.2.1 Both onFulfilled and onRejected are optional arguments:  
> &nbsp;&nbsp;&nbsp;2.2.1.1 If onFulfilled is not a function, it must be ignored.  
> &nbsp;&nbsp;&nbsp;2.2.1.2 If onRejected is not a function, it must be ignored.

`onFulfilled` and `onRejected` should be validated:

```javascript
let fulfillFunc = isFunction(onFulfilled) ? onFulfilled : (value) => value;
let rejectFunc = isFunction(onRejected)
  ? onRejected
  : (e) => {
      throw e;
    };
```

### Update state

After validating the parameters, we can implement the point `2.2.2` and `2.2.3`. Translate here:

> If onFulfilled/onRejected is a function,  
> &nbsp;&nbsp;&nbsp;it must be called after promise is fulfilled/rejected, with promise's value/reason as its first argument;  
> &nbsp;&nbsp;&nbsp;it must not be called before promise is fulfilled/rejected;  
> &nbsp;&nbsp;&nbsp;it must not be called more than once;

So we need to set the state and pass the value or reason to related functions. To do this, I wrap the validated functions and update internal states inside of them:

```javascript
// 2.2.2.1 onFulfilled must be called after promise is fulfilled, with promise’s value as its first argument.
function resolve(value) {
  if (self.state === STATE.PENDING) {
    // 2.2.2.2 it must not be called before promise is fulfilled.
    self.state = STATE.FULFILLED;
    self.value = value;
    fulfillFunc(value);
  }
}

// 2.2.3.1 onRejected must be called after promise is rejected, with promise’s reason as its first argument.
function reject(err) {
  if (self.state === STATE.PENDING) {
    // 2.2.3.2 it must not be called before promise is rejected.
    self.state = STATE.REJECTED;
    self.value = err;
    rejectFunc(err);
  }
}
```

### Asynchronized execution

According to the first point `2.2.4` refering NOTE `3.1`, we should execute the fulfill and reject functions asynchronously, which is the main reason for people using it. In javascript, we can utilize `setTimeout`.

```javascript
setTimeout(() => fulfillFunc(value), 0);
```

## Return `Promise` in `then`

Before we continue implementing the rest requirement, we need to reorganize the codes first. We need to move the `executor` from `then` to the constructor of `Promise`.

Why? One main reason is we will execute the `executor` multiple times if it's in `then`, since one promise can have multiple `then`s. This is definitely unacceptable because we only need to execute `executor` once.

So til now, the `Promise` function looks like this:

```javascript
function Promise(executor) {
  let self = this;

  // set the state as pending, 2.1.1
  self.state = STATE.PENDING;

  // 2.2.2.1 onFulfilled must be called after promise is fulfilled, with promise’s value as its first argument.
  function resolve(value) {
    if (self.state === STATE.PENDING) {
      // 2.2.2.2 it must not be called before promise is fulfilled.
      self.state = STATE.FULFILLED;
      self.value = value;
      setTimeout(() => resolveFunc(value), 0);
    }
  }

  // 2.2.3.1 onRejected must be called after promise is rejected, with promise’s reason as its first argument.
  function reject(err) {
    if (self.state === STATE.PENDING) {
      // 2.2.3.2 it must not be called before promise is rejected.
      self.state = STATE.REJECTED;
      self.value = err;
      setTimeout(() => rejectFunc(err), 0);
    }
  }

  try {
    // executor is function whose parameters is resolve and reject functions,
    // which would be called inside of executor.
    if (isFunction(executor)) executor(resolve, reject);
  } catch (err) {
    reject(err);
  }
}
```

And you may notice we use the function `resolveFunc` and `rejectFunc`, but we haven't define them. They would work together with the requirement of multiple calling of `then`.

### Call `then` mutiple times 2.2.6

> 2.2.6 `then` may be called multiple times on the same promise.

So all the respective _fulfill/reject_ functions should be called in the order of original calls. Obviously, we can apply a queue here. Create a callback queue, add `then`'s `onFulfilled` and `onRejected` parameters to it and handle it when fulfilled/rejected. And this is how we handle the above `resolveFunc` and `rejectFunc`.

```javascript
function resolve(value) {
  ...
  // 2.2.6.1
  setTimeout(
    () => self.callback.forEach(({ resolveFunc }) => resolveFunc(value)),
    0
  );
}

function reject(err) {
  ...
  // 2.2.6.2
  setTimeout(
    () => self.callback.forEach(({ rejectFunc }) => rejectFunc(err)),
    0
  );
}
```

When `Promise` is fulfilled/rejected, we would execute all the related functions in the queue. And obviously, we have to push the callback functions to the queue in `then` when the state is still pending.

```javascript
Promise.prototype.then = function (onFulfilled, onRejected) {
  let self = this;

  // 2.2.1 Both onFulfilled and onRejected are optional arguments, if any is not function, must ignore it
  let fulfillFunc = isFunction(onFulfilled) ? onFulfilled : (value) => value;
  let rejectFunc = isFunction(onRejected)
    ? onRejected
    : (e) => {
        throw e;
      };

  switch (self.state) {
    // if the state is fulfilled or rejected, just execute the related function and pass the result to the resolvePromise
    case STATE.FULFILLED:
    case STATE.REJECTED:
      return setTimeout(() => {
        try {
          let func = self.state == STATE.FULFILLED ? fulfillFunc : rejectFunc;
          func(self.value);
        } catch (e) {
          rejectFunc(e);
        }
      }, 0);
    case STATE.PENDING:
      // if it's still pending, push the resolve/reject to callback queue. All the callback functions would be executed once state are changed
      return self.callback.push({
        resolveFunc: () => {
          try {
            fulfillFunc(self.value);
          } catch (e) {
            rejectFunc(e);
          }
        },
        rejectFunc: () => {
          try {
            rejectFunc(self.value);
          } catch (e) {
            rejectFunc(e);
          }
        },
      });
  }
};
```

### Return `Promise` 2.2.7

Here comes the difficult part, `then` should return a `Promise` which would support the chaining feature.

> then must return a promise [3.3].  
> &nbsp;&nbsp;&nbsp;`promise2 = promise1.then(onFulfilled, onRejected);`

After reading other implementations, here comes a question. Would this `promise2` have its own `executor`? Yes or no would have different implementations.

Let's first implement the simple one - **Yes**. It would have its own `resolve/reject` functions in `executor`. I would implement the optimized one -- **No**, with an empty promise, in another blog.

Another thing we need to think of is what if the value returned by `promise2` is a promise. This is what the `2.3 Promise Resolution Procedure` would do. Let's preserve this to later chapter and assume the value returned by `promise2` is **_NOT_** another promise
Then the logic of this `promise2`'s `executor` should be:

1. If the state is fulfilled, the value returned by the `onFulfilled` function should be passed to `resolve2` function in `promise2`'s executor
2. If the state is rejected, the value returned by the `onRejected` function should be passed to `reject2` function in `promise2`'s executor
3. If the state is still pending, pass the `resolveFunc` and `rejectFunc` which would call `resolve2` and `reject2`, to callback queue
4. Any exception throwed by `onFulfilled` or `onRejected` should be handled by `reject2`

And we can extract the process of handling `self.value` as a function to reuse code.

```javascript
function handleResult(resolve2, reject2) {
  return () => {
    try {
      // 2.2.7.1, 2.2.7.2
      let func = self.state == STATE.FULFILLED ? fulfillFunc : rejectFunc;
      let func2 = self.state == STATE.FULFILLED ? resolve2 : reject2;
      func2(func(self.value));
    } catch (e) {
      reject2(e);
    }
  };
}
```

Til now, we have implement the features

1. Return Promise in `then` to support chaining
2. Multiple `then` of one Promise

The codes should be like this:

```javascript
const STATE = {
  PENDING: Symbol.for("pending"),
  FULFILLED: Symbol.for("fulfilled"),
  REJECTED: Symbol.for("rejected"),
};

const isFunction = (func) => func && typeof func === "function";
const isObject = (arg) => arg && typeof arg === "object";

function Promise(executor) {
  let self = this;

  // set the state as pending, 2.1.1
  self.state = STATE.PENDING;

  self.callback = [];

  // 2.2.2.1 onFulfilled must be called after promise is fulfilled, with promise’s value as its first argument.
  function resolve(value) {
    if (self.state === STATE.PENDING) {
      // 2.2.2.2 it must not be called before promise is fulfilled.
      self.state = STATE.FULFILLED;
      self.value = value;
      // 2.2.6.1
      setTimeout(
        () => self.callback.forEach(({ resolveFunc }) => resolveFunc(value)),
        0
      );
    }
  }

  // 2.2.3.1 onRejected must be called after promise is rejected, with promise’s reason as its first argument.
  function reject(err) {
    if (self.state === STATE.PENDING) {
      // 2.2.3.2 it must not be called before promise is rejected.
      self.state = STATE.REJECTED;
      self.value = err;
      // 2.2.6.2
      setTimeout(
        () => self.callback.forEach(({ rejectFunc }) => rejectFunc(err)),
        0
      );
    }
  }

  try {
    // executor is function whose parameters is resolve and reject functions,
    // which would be called inside of executor.
    if (isFunction(executor)) executor(resolve, reject);
  } catch (err) {
    reject(err);
  }
}

Promise.prototype.then = function (onFulfilled, onRejected) {
  let self = this;

  // 2.2.1 Both onFulfilled and onRejected are optional arguments, if any is not function, must ignore it
  let fulfillFunc = isFunction(onFulfilled) ? onFulfilled : (value) => value;
  let rejectFunc = isFunction(onRejected)
    ? onRejected
    : (e) => {
        throw e;
      };

  function handleResult(resolve2, reject2) {
    return () => {
      try {
        // 2.2.7.1, 2.2.7.2
        let func = self.state == STATE.FULFILLED ? fulfillFunc : rejectFunc;
        let func2 = self.state == STATE.FULFILLED ? resolve2 : reject2;
        func2(func(self.value));
      } catch (e) {
        reject2(e);
      }
    };
  }

  return new Promise((resolve2, reject2) => {
    switch (self.state) {
      // if the state is fulfilled or rejected, just execute the related function and pass the result to the resolvePromise
      case STATE.FULFILLED:
        return setTimeout(handleResult(resolve2, reject2), 0);
      case STATE.REJECTED:
        return setTimeout(handleResult(resolve2, reject2), 0);
      case STATE.PENDING:
        // if it's still pending, push the resolve/reject to callback queue. All the callback functions would be executed once state are changed
        return self.callback.push({
          resolveFunc: handleResult(resolve2, reject2),
          rejectFunc: handleResult(resolve2, reject2),
        });
    }
  });
};
```

Let's simply test it:

```javascript
const p1 = new Promise((resolve, reject) => {
  setTimeout(() => resolve("resolved first one"), 3000);
});

p1.then((res) => {
  console.log("then1: ", res);
  return res;
}).then((res) => {
  setTimeout(() => console.log("then2: ", res), 1000);
});

p1.then((res) => {
  console.log("another then: ", res);
});

// then1:  resolved first one
// another then:  resolved first one
// then2:  resolved first one
```

## Promise Resolution Procedure 2.3

Here comes the last step, implement the `Promise Resolution Procedure`. According to the description of requirement:

> This treatment of thenables allows promise implementations to interoperate, as long as they expose a Promises/A+-compliant then method. It also allows Promises/A+ implementations to “assimilate” nonconformant implementations with reasonable then methods.

And before we begin to implement, let's think of why we have to have this `resolvePromise`, since it calls itself inside recursively. Comparing the text explanation, let me present one test case from [Promise/A+](https://github.com/promises-aplus/promises-tests/blob/master/lib/tests/2.3.3.js#L332). The case is generated by two loops, I just pick one here.

### Adapter

In the test case, an adapter is utilized and explained in the Promise/A+ as well, check [adapter](https://github.com/promises-aplus/promises-tests#adapters);

```javascript
Promise.defer = Promise.deferred = function () {
  let dfd = {};
  dfd.promise = new Promise((resolve, reject) => {
    dfd.resolve = resolve;
    dfd.reject = reject;
  });
  return dfd;
};

adapter.resolved = function (value) {
  var d = adapter.deferred();
  d.resolve(value);
  return d.promise;
};
```

Definitely, we have seen this `resolved` many times, but what it does exactly? Through its code, we can understand it

1. Create a promise with a very simple `executor` which normally executes the logic of asynchronous codes
2. Resolve the promise with the value immediately by updating state and saving the value before we have this `resolve` method
3. Return this created promise

So for the code `resolved(value)`, actually it just preserves the value and waiting the `resolve` method from its `then`. When its `then` is called, the value would be passed to `resolve` method immediately.

### Test case

The test case's description is

> `y` is an already-fulfilled promise for a synchronously-fulfilled custom thenable, `then` calls `resolvePromise` synchronously

And I can simplify the test code, removing all the wrapped test functions:

```javascript
const result = { result };

var promise = resolved({ dummy }).then(function onBasePromiseFulfilled() {
  return {
    then: function (resolvePromise) {
      resolvePromise(
        resolved({
          then: function (onFulfilled) {
            onFulfilled(result);
          },
        })
      );
    },
  };
});

promise.then(function onPromiseFulfilled(value) {
  assert.strictEqual(value, result);
  done();
});
```

Yes, HHHHHHHHeadache!!!!

Definitely there are a lot of promises wrapped like matryoshka doll, and so hard to dig into it. Yes, I know, but we can analysis the codes step by step, at leas we can simply count how many promises are here:

1. `resolved({ dummy })` uses `resolved`. As explained above, it returns a resolved promise and waits for the `resolve` method from its `then`. Let's call this promise as `promise-TEMP`;
2. `promise-TEMP` called `then` which passes the function `onBasePromiseFulfilled` as `resolve`. **_AND_** it returns our first promise -- `promise`
3. `onBasePromiseFulfilled` return a `thenable` object as value of `promise`. Let's call it `x`;
4. `x`, which we can simply treat it as a `Promise` as well -- `promise2`, has the `then` function which would call its parameter `resolvePromise` further. The called value would be the value of `promise2`. Let's call it `y`;
5. The value passed to `resolvePromise` is another resolved `promise` -- `promise3`, which is fulfilled and has no `resolve` method. But its value is another thenable object -- or `promise4`;
6. Finally we reach the bottom level, the resolve method `onFulfilled` of `promise4`'s `then` calls the `result`;

So let's count how many promises and values we have here (ignore the `promise-TEMP`):

1. `promise` -- the only one having name in our test codes
   1. its value `x` -> `promise2`
2. `promise2` -- first thenable object
   1. its value `y` -> `promise3`
3. `promise3` -- a resolved `promise`
   1. its value -> `promise4`
4. `promise4` -- second thenable object
   1. its value -> `result`, an object

So we can see the value of `promise` is a promise -- `promise2` which wraps another two promises -- `promise3` and `promise4`. And the `assert` sits inside of the resolve `onPromiseFulfilled` method of the first `promise`, it would expect the value returned by `onPromiseFulfilled` to be the same with `result`, which is the value of `promise4`.

We can conclude some points here:

1. The thenable object can be treated as a promise, which means they can be handled as the same codes. (_This is not a principle, we can discuss this in another blog_)
2. If the value of a promise is a thenable object, the promise's resolve/reject methods would be passed to the value until the final value is not a promise and handled by the original resolve/reject methods.

### Implementation

OK, it's enough to learn from the test case, even though it's what I learned after I passed all the test cases. Let's go back to the requirements and implement it.

Since thenable object can be treated as promise, we would ignore the requirement of `2.3.2`:

> 2.3.2 If x is a promise, adopt its state [3.4]:  
> &nbsp;&nbsp;&nbsp; 2.3.2.1 If x is pending, promise must remain pending until x is fulfilled or rejected.  
> &nbsp;&nbsp;&nbsp;2.3.2.2 If/when x is fulfilled, fulfill promise with the same value.  
> &nbsp;&nbsp;&nbsp;2.3.2.3 If/when x is rejected, reject promise with the same reason.

This point can be merged with `2.3.3`. Others would not be hard, just follow the steps:

```javascript
function resolvePromise(promise, x, resolve2, reject2) {
  if (promise == x) {
    return reject2(
      new TypeError("Resolved result should not be the same promise!")
    );
  } else if (x && (isFunction(x) || isObject(x))) {
    let called = false; // 2.3.3.3.3

    try {
      let then = x.then;

      if (isFunction(then)) {
        then.call( // 2.3.3.3
          x,
          function (y) {
            if (called) return; // 2.3.3.3.3
            called = true;
            return resolvePromise(promise, y, resolve2, reject2); // 2.3.3.3.1
          },
          function (r) {
            if (called) return;
            called = true;
            return reject2(r); // 2.3.3.3.2
          }
        );
      } else {
        resolve2(x); // 2.3.3.4
      }
    } catch (err) {
      if (called) return; // 2.3.3.3.4.1
      called = true;
      reject2(err); // 2.3.3.3.4.2
    }
  } else {
    resolve2(x); // 2.3.4
  }
}
```

## Test

Now we have implemented all the mandatory codes, we can run the test the codes with npm lib [promises-aplus-tests](https://www.npmjs.com/package/promises-aplus-tests).
```bash
npm i -g promises-aplus-tests
promises-aplus-tests promise1.js // promise1.js is the file name
```

The full code can be checked [here](https://github.com/xfsnowind/promise-implementation). There are different versions with different techniques, you can choose anyone.

## Summary

This solution is not perfect and it just simply follows the rules of Promise/A+ without any better architecture and design. There are a lot of good solutions which restructures the codes with their own idea and logic. I would rewrite the promise with other techniques later.

Besides, this version just completes the basic part. Promise also has `resolve`, `catch`, `finally`, etc. I will implement them as well and write another article about them.

Thanks to [this article](https://developpaper.com/source-code-implementation-of-promise-perfect-in-accordance-with-promise-a-specification/), which inspires me to make decision to implement the Promise and [Zhi Sun's article](https://medium.com/swlh/implement-a-simple-promise-in-javascript-20c9705f197a), which reminders me of taking steps to implement the hard part.