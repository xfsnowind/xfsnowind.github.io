---
title: "Learning Notes - React Hooks"
date: 2022-08-07T14:50:28+02:00
author: "Feng Xue"
tags: ["Javascript", "React Hooks", "Learning Notes"]
toc: true
usePageBundles: false
draft: false
---

This article is a summary when I learned the React Hooks course in GeekTime. Even though we all use React Hooks in the frontend development, I cannot say I understand the internal core and logic of it. So this is a good time to re-learn it through this course. According to my experience, it's definitely better to write down the idea and thoughts down -- The palest ink is better than the best memory. Therefore have this article. And I would follow the course's original structure to organize the article.

## Basic chapter

### Reason to Hooks

The nature of React is mapping the Model to View and the Model is the Component's props and state. So when Model's data change, React does not care how they change, it only focuses on the difference. And this is what we called declarative and this is implemented by React's `diff` function.

So UI presentation is more like execution of function. Model is parameter, View is function and the result would be the Dom's change. React just confirms to execute the process of changing in an optimized way.

Before we use class to create React Component, but actually it's not suit for React Component:

1. We rarely use inheritance in React Component;
2. React is state driven and does not need generated instance's methods;

But function also has its own limitation:

1. Function cannot provide internal state, it must be a pure function;
2. Function cannot provide the entire lifecycle;

The comes React Hooks.

React Hooks "binds" or "hooks" the target to some may changed data or event source. And when the hooked data or event change, the target would be executed again to generate the new result.

The hooked source can be not only data, also the result of another Hooks execution.

Hooks is created with the background of using High order Component, and it solves the problems of wrapper hell and code hard to understanding.

### Hooks basic usage

* `useState` -- The principle of using `useState` is unnecessary to save the value which can be gotton from calculating.
* `useEffect` -- Should only be used to execute the code which would not effect the current result, not effect the rendered UI.
  - And also be careful for the `deps`, it use reference to check if values have been changed, so take care of object and array.
  - ***NB***: `useEffect` is called after rendering
* `useCallback` -- The purpose is when need to pass function as parameter to UI, avoid triggering React render component.
* `useMemo` -- can be treated as combination of `useEffect` and `useState`. When `deps` change, execute `useEffect` to calculate the value and set the value through `useState`
* `useRef`
  * Share data between multiple rendering
  * Save the ref of a Dom node
* `useContext` -- define the global state

`useEffect` can be equivalent to `componentDidMount`, `componentDidUpdate` and `componentWillUnmount`, but not exactly. The difference is
* `useEffect`'s callback functions are triggered only when `deps` change. While `componentDidUpdate` would be called every rendering
* `useEffect`'s callback function return a function, which is used to clean, would be triggered before `deps` change or component unmount.

If we need a constructor feature, we can use the code below:
```javascript
function useSingleton(callback) {
  // 用一个 called ref 标记 callback 是否执行过
  const called = useRef(false);
  // 如果已经执行过，则直接返回
  if (called.current) return;
  // 第一次调用时直接执行
  callBack();
  // 设置标记为已执行过
  called.current = true;
}
```

***NB***: Hooks can implement most functionalities of lifecycle, but not for `getSnapshotBeforeUpdate`, `componentDidCatch`, `getDerivedStateFromError`. These can only be implemented by class.

## Practice

### Data consistence

The principle of using `useState` is keep state minimum.

If the data can be calculated or generated by the existing ones, then we should not store them in state.

When we define the new state, ask yourself: is this state necessary? Can it be obtained by calculation? Is it just a middleware state?

### Handle rendering scenario

Since Hooks cannot be handled in conditions and loops, we should move the condition into `useEffect` og create a wrapper for the component and return null in some conditions. 

Although we have Hooks now, it can only be used for logical reuse. If it comes to the UI behaviour, Hooks cannot play a role then. Therefore, we can use `Render props` mode.

Render props Mode is just another presentation of High-ordering Component, which means Component takes functions as paramter or return functions. And then we can use the passed function to render, kind of like dependency injection. For example:

```javascript
function CounterRenderProps({ children }) {
  const [count, setCount] = useState(0);
  const increment = useCallback(() => {
    setCount(count + 1);
  }, [count]);
  const decrement = useCallback(() => {
    setCount(count - 1);
  }, [count]);

  return children({ count, increment, decrement });
}

function CounterRenderPropsExample() {
  return (
    <CounterRenderProps>
      {({ count, increment, decrement }) => {
        ...
      }}
    </CounterRenderProps>
  );
}
```

So we leave `children` to render to make code reusable. Here, it does not have to be `children`, it can be any functions.

### Self defined event

When we bind an event to a node, because of Virtual Dom, React would bind the event to the app's root node. Before version 17, it's on `document`, after version 17, it's the react's root node.
The reason to do this:
1. When Virtual Dom renders, the node may have not been mounted to the page, so it cannot bind;
2. It can block all the details from low level and avoid browser's compatible problem

So React's event actually is the callback function.

### Organize project structure via business

To reduce the complexibility, we can organize the project based on service characteristic, so each feature can be independent and easy to manage and maintain.

To meet the requirement of low coupling, we can define some high level, abstract components to be reused among components.


### Form

React is state driven, while Form is event driven。
The difference of React's `onChange` and html's `onchange` is `onChange` would be called whenever user inputs, while `onchange` is only triggered when the input loses focus.

#### Controlled vs uncontrolled

For uncontrolled component, it would not pass the value to component, can only get the value actively, like `useRef`. The advantage is it would not toggle the rendering, although we cannot see the change of value as well.

While for controlled component, it accepts value as props and add a callback function to update it.

![Form elements](/images/react-hooks/form_element.png "Form elements")

If we use Controlled component to build form, it would have three core parts:
1. the type of form element
2. bind the value
3. handle the onChange event

So Hooks' contribution to form is, we can save the form's values to Hooks and provide the function to handle them through `useState`. For example:

```javascript
import { useState, useCallback } from "react";

const useForm = (initialValues = {}) => {
  // define the state for the whole form：values
  const [values, setValues] = useState(initialValues);

  // provide a method to set the value of some field
  const setFieldValue = useCallback((name, value) => {
    setValues((values) => ({
      ...values,
      [name]: value,
    }));
  }, []);

  // return the values and the method
  return { values, setFieldValue };
};
```