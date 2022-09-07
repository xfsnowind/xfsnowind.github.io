---
title: "Promise Others Methods"
date: 2022-08-17T22:24:58+02:00
author: "Feng Xue"
draft: true
---

So I had implemented the core functionalities of `Promise` [here](/blogs/promise), but native `Promise` provides some other useful methods:

- Promise.all
- Promise.allSettled
- Promise.any
- Promise.prototype.catch
- Promise.prototype.finally
- Promise.race
- Promise.reject
- Promise.resolve

We do not have the requirement for these methods, we can follow the description of [mozilla's doc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).

Before we start, you may have notice the differences of these methods. Only `catch` and `finally` are prototyped, while all others are static methods, which means they would not handle with the data of instances of the class.


## Promise.resolve

I would implement the `resolve` and `reject` first, since they would be used by other methods as well.

> The Promise.resolve() method "resolves" a given value to a Promise. If the value is a promise, that promise is returned; if the value is a thenable, Promise.resolve() will call the then() method with two callbacks it prepared; otherwise the returned promise will be fulfilled with the value.
> 
## Promise.reject


> The Promise.reject() method returns a Promise object that is rejected with a given reason.



## Promise.all

After checking the description from MDN, we should 
1. take an iterable object as parameter, like array;
2. return a promise that resolves each value in the input array if all are fulfilled;
3. reject immediately if any rejects or throws an error

Here is the example code from MDN:

```javascript
const promise1 = Promise.resolve(3);
const promise2 = 42;
const promise3 = new Promise((resolve, reject) => {
  setTimeout(resolve, 100, "foo");
});

Promise.all([promise1, promise2, promise3]).then((values) => {
  console.log(values);
});
// expected output: Array [3, 42, "foo"]
```

## Promise.allSettled

> The Promise.allSettled() method returns a promise that fulfills after all of the given promises have either fulfilled or rejected, with an array of objects that each describes the outcome of each promise.

## Promise.any

> Promise.any() takes an iterable of Promise objects. It returns a single promise that fulfills as soon as any of the promises in the iterable fulfills, with the value of the fulfilled promise. If no promises in the iterable fulfill (if all of the given promises are rejected), then the returned promise is rejected with an AggregateError, a new subclass of Error that groups together individual errors.

## Promise.prototype.catch

> The catch() method returns a Promise and deals with rejected cases only. It behaves the same as calling Promise.prototype.then(undefined, onRejected) (in fact, calling obj.catch(onRejected) internally calls obj.then(undefined, onRejected)). This means that you have to provide an onRejected function even if you want to fall back to an undefined result value - for example obj.catch(() => {}).

## Promise.prototype.finally

> The finally() method of a Promise schedules a function, the callback function, to be called when the promise is settled. Like then() and catch(), it immediately returns an equivalent Promise object, allowing you to chain calls to another promise method, an operation called composition.

## Promise.race

> The Promise.race() method returns a promise that fulfills or rejects as soon as one of the promises in an iterable fulfills or rejects, with the value or reason from that promise.

