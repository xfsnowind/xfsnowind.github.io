---
title: "Learning Notes From New React Website"
date: 2023-06-10T16:06:56+08:00
author: "Feng Xue"
tags: ["Javascript", "Learning Notes", "React"]
toc: true
usePageBundles: false
draft: false
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

```jsx
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

Then we would rethink the `useState` hook. The second `setter` method returned by this hook will theoritically trigger the `re-rendering`. Everytime it's called, it will inform `React` to re-render the page and read the `state` value.

### Batching

`Batching` means the UI will not be updated until the event handler, with codes inside of it, finishes. So if there are multiple `setter` methods, before the next re-render, all of them will be executed. This is a kind of 'old' new feature, it used to happen that the following `setter` methods are not executed in React's old version.

### Updating the state in Object or Array

Since Object and array are the reference, not plain variable type, so `React` would not recognise the changing of the state if you just `mutate` the member values of them. The correct solution is to **create** a new one instead of passing the old one to `setter` method.

## Position matters

Check this official example first

<iframe src="https://codesandbox.io/embed/wizardly-edison-8lrye8?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
  title="react-state-presever-same-position-in-tree"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

We might expect the state to be resetted when we check/uncheck the `Use fancy styling` checkbox. But it isn't. This is **because both of these `<Counter />` tags are rendered at the same position**. Since this two `Counter` has the same structure, so React treat them as the same `Counter`.

Do remember `React`'s explanation:

> React will keep the state around for as long as you render the same component at the same position.
>
> React preserves a component’s state for as long as it’s being rendered at its position in the UI tree.

### Different components in the same position

But if there are different components in the same position, `React` will reset the state of the whole subtree.

### Reset state in the same position

So how to distinguish the component if we have the same components with different props in the same position? For example, we have a video, and want to change its source url when click one button:

<iframe src="https://codesandbox.io/embed/video-auto-change-forked-45yjxf?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:800px; border:0; border-radius: 4px; overflow:hidden;"
  title="Video-auto-change (forked)"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

The expect behaviour is when we click on the button, video player auto plays the different videos. But it does not happen because the above reason, `React` applies the same props to the component in the same position. So even though the value `isPlayingOne` changes, the player does not reload.

One imperative solution would be adding a reference to the video element and a `useEffect` function to the `Video` component, when the `src` url changes, we reload the player. Check the code of file `ImperativeVideo.tsx`.

According to the solutions from `React` official website, we also have two declarative solutions:

One is to render the component in the different positions:

```jsx
<h2>Video with different position</h2>
{isPlayingOne && (
  <OriginalVideo
    src={
      "http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4"
    }
  />
)}
{!isPlayingOne && (
  <OriginalVideo
    src={
      "http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ElephantsDream.mp4"
    }
  />
)}
```

The other would be assigning the key to the component, `React` would recognise the component with the key, check the file `VideoWithKey.tsx`. The key is only required to be unique within the parent, not globally.

```jsx
<Video
  key={isPlayingOne ? "one" : "two"} // NOTE: Add this line
  src={
    isPlayingOne
      ? "http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4"
      : "http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ElephantsDream.mp4"
  }
/>
```

## Context -- sharing state with deep children

Normally, we do not use `Context` that often, only on the top level to pass the theme and `Redux`'s store. One of the reason is it would not be testable for the component and using `Context` would make the props unpredictable. Actually it's very good to make it clear that the props passed to the component, especially for the person who maintain this explicit data flow component.

And according to the official doc, `Context` can also be avoided by passing the components as `children`. Like instead of using `<Layout posts={posts} />`, we can make `Layout` to accept the component as `children`, `<Layout><Posts posts={posts} /></Layout>`.

## About Effect

The main principle of `useEffect` is it should mainly be used to specify the side effect caused by the rendering or external system, like fetching. There are several things we should not use Effect to do:

1. **Do not use Effect to calculate data from props and state and call the setter method.** The reason for this is it introduces the unnecessary procedure to re-render the page. When the page gets rendered, after it finishes, it will trigger the `useEffect` method, as we called the `setter` method in the Effect. With `setter` method, the page will be re-rendered again.
2. **Cache the expensive data.** use `useMemo` to cache the expensive data instead of calculating every render.
3. **Mostly do not update state according to props change.** When you need to do this, re-think of your component in these ways:
   * compute the entire state based on the current state or props, use `useMemo` when too complicated
   * reset the entire component with using `key`
   * update the state in the related event handler
   However, in the rare case, it may still need to update the state from the rendering. Then be careful and remember to update it `without` in the Effect. Like this official example:

    ```jsx
    function CountLabel({ count }) {
      const [prevCount, setPrevCount] = useState(count);
      const [trend, setTrend] = useState(null);
      if (prevCount !== count) {
        setPrevCount(count);
        setTrend(count > prevCount ? 'increasing' : 'decreasing');
      }
      return (
        <>
          <h1>{count}</h1>
          {trend && <p>The count is {trend}</p>}
        </>
      );
    }
    ```

4. **Use Effects only for code that should run because the component was displayed to the user, not user's behaviour.**  If a logic is shared, instead of creating Effect, think of creating a function for it.
5. **use `useSyncExternalStore` for subscribing to an external store.**
6. **Only use the reactive variables as dependency list.** All the variables inside of the component, including props, states and generated from props and states are **reactive**, because they are calculated during the rendering and participate in the `React` data flow. `React` requires to add these reactive variables in the Effects' dependency list. But the **global and mutable** values like, `location.pathname`, `ref.current` could not be dependencies. Because changing these mutable variables could happen outside of the component and it would not trigger a `re-render`

### Difference between event handlers and Effects

1. Event handler reacts to user's specific actions, while Effects runs because synchronization is required;
2. Logic inside Event handlers is **not** reactive, which is mostly triggered by user, while logic inside Effects is reactive since something `side effects` should happen according to certain reactive values such as props, state changed;

### Start question to use Effects

1. **Should this code be moved to event handler or be an Effect?** 
2. **Is the Effect doing several unrelated things?** Several things should splitted to different Effects.
3. **Does some reactive value change unintentionally?**
4. **Are you reading some state to calculate the next state?**
   1. Move static objects and functions outside your component
   2. Move dynamic objects and functions inside your Effect
   3. Read primitive values from objects
   4. Calculate primitive values from functions

## `useEvent` -- an event handler allowing side effects

React proposed a RFC hook [`useEvent`](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md) to improve the way of using event handler, even though it's not official published yet, we have already use it generally in our codes.

To understand the motivation of `useEvent`, you can check the [official explanation](https://github.com/reactjs/rfcs/blob/useevent/text/0000-useevent.md) or [this blog](https://blog.logrocket.com/what-you-need-know-react-useevent-hook-rfc). But here I want to explain the usage of `useEvent` as event dispatcher and handler.

```jsx
import { useMutation } from 'react-query'

const addMutation = useMutation()
const [internalState, setInternalState] = useState({ contact: 'name' })

const dispatch = useEvent((event) => eventHandler({event, state: internalState, setState: setInternalState, sideEffects: { addMutation }}))

const eventHandler = ({ event, state, setState, sideEffects: { addMutation }}) => {

  switch(event) {
    case 'add':
      addMutation.mutate(state.contact, {
        onSuccess: (succcess) => {
          setState((prevState) => ({
            ...prevState,
            success,
          }))
        }
      })
      break;
    default:
      history.push('/test')
  }
}

return (
  <button onClick={() => dispatch('add')}>
)
```

So from the above code, I defined a event handler called `eventHandler` which takes the parameter `event` from the function passed to the `useEvent`, the defined variables from `useState` and the last one `sideEffects` which I will explain it later. So I named the value returned by the `useEvent` `dispatch`, it can be called anywhere except in the rendering. In the codes, I called it when we click on the button.

In the defined `eventHandler`, I check the `event` type, when it's `'add'`, I will call the `addMutation` passed from the parameter `sideEffects`, and set the state once it succeeds. In the default branch, I want to change the route to the '/test'.

So the good thing of using this event handler with `useEvent` is it moves all the updating logic and side effects operations to the independent method `eventHandler`, so it's more clear to understand the modification operations since they are moved together and easier to understand. Besides it makes both the component and handler pure and testable. We can of course move this `eventHandler` to a single file.

### Why not `useReducer`?

Yes, it comes a question automatically: why don't we use another state dispatcher and handler hook `useState`, since that seems simpler, at least it's not required to explicitly define the state, pass both state and setter method to the event handler.

```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init)
```

The quick answer is `useReducer` cannot handle the side effect, since its `reducer` has to be pure, and return the updated state every time, while `eventHandler` has more choices, not just setting the state, but also can do the side effects with returning void. So as a `event handler`, `useEvent` provides more flexibility to handle the side effects.
