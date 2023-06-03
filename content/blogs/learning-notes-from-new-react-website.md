---
title: "Learning Notes From New React Website"
date: 2023-05-28T16:06:56+08:00
author: "Feng Xue"
draft: true
---

It has been a while after React published its new webiste, I would write down what I learnt from the new webiste.

## Describe the UI

### List Key

I know this is an old issue, but I used to use index as the key of list. To summarize it, I will note the correct way to use the key:

* Why do we have to use key for the list item? 
    Because the item in the list could be modified, such as insert, deleted or re-sorted, so the React need to know which item the component is responible to.
* Why do we cannot use `index` as the key?
    If the list is not changed, it would be fine to use index as the key. But if we have the modificaiton operation to the list, like add, insert, delete or resort, React would be confused.
* Why do we cannot use a `random (non-duplicated)` number as the key?
    Because with random new generated key, it would violate the reason React uses key. React would not know the item components it used to render and has to recreate all the related components and DOMs, which is not only slow, but also lose all the user inputs.

So the principles to use key are:

1. It should be unique within the list locally, unnecessary to be globally
2. Use the data which is unique naturally
3. Do not change the key value accross rendering, it should be persistant.

## Think in React with lifecycle

Actually, I used to have a question: why the local variable inside of a component would not be updated when I changed it, like this:

```js
export default function Gallery() {
  let index = 0;

  function handleClick() {
    index = index + 1;
  }

  let sculpture = sculptureList[index];
  return (
    <>
      <button onClick={handleClick}>
        Next
      </button>
      <h3>  
        ({index + 1} of {sculptureList.length})
      </h3>
      <p>
        {sculpture.description}
      </p>
    </>
  );
}
```

When we click on the button, the variable `sculpture` is not updated based on the `index`. So why? The reason for this is I only consider the native javascript, not taking one point knowledge of the frontend development into account. This knowledge is the `rendering`. `React` will render the component to the DOM and it uses the hooks to save the internal variables or states among the renderings. So either this local variable `index` will be recreated during each rendering which causes the click action useless, or updating the `index` value would not trigger the `rendering` which may reflect the updated value to the DOM.

So although we understand this, where does `React` save the internal states? Actually, `React` just save all the hooks in an internal array, and all the hooks would be identified by its `index` value. So this is also why the hooks can only be defined in the top level of component and cannot be used conditionally. Because if we do that, React would not be able to find the previous correct hooks. 

Then we would rethink the `useState` hook. The second `set` method returned by this hook will theoritically trigger the `re-rendering`. Everytime it's called, it will inform `React` to re-render the page and read the `state` value.

### Batching

`Batching` means the UI will not be updated until the event handler, with codes inside of it, finishes. So if there are multiple `set` methods, before the next re-render, all of them will be executed. This is a kind of 'old' new feature, it used to happen that the following `set` methods are not executed in React's old version.

### Updating the state in Object or Array

Since Object and array are the reference, not plain variable type, so React would not recognise the changing of the state if you just mutate the values of them. The correct solution is to create a new one instead of passing the old one to `set` method.