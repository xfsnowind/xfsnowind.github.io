---
title: "How to implement a FolderFlip 2"
date: 2022-10-13T22:55:11+08:00
author: "Feng Xue"
tags: ["Javascript", "React", "FolderFlip", "Frontend"]
toc: true
usePageBundles: false
draft: false
---

As we presented in the [previous article](/blogs/folderflip/), we have showed how to implement the FolderFlip with limited number (like 3) items with the `position: sticky` and `IntersectionObserver`. 

## The Problem

But it only allows limited number, if it comes more items or the screen is smaller, the items would not be able to scroll. So how would we display if the items are more and the titles take most of the screen.

<div style="text-align:center;">
  <img src="/images/folderflip/FolderFlip-6items.png" alt="Folder Flip with 6 items" width="400"/>
</div>

## Clear the logic first

If you want the final answer, just jump to [here](https://codesandbox.io/s/folder-flip-reducer-j8h6xz). Otherwises, I would explain the solutions and the procedures below, also some problems I met.

The idea is we only display a certain number of items in the screen, when the items' number reaches the limit with scrolling down/up, the next/previous one would float out to leave the room for the new item, which can be implemented by changing `postion` to `relative` like what we have done in the [previous blog](https://xfsnowind.github.io/blogs/folderflip/#floating-with-intersectionobserver). So it's like the state transition. I call the state `sticky` before some item reaches the threshold, when it reaches, the whole component would transit to a state named `float`. And in the `float` state, the first item (according to the scroll direction) would be moved out of screen.

And as we know, React is a declarative library, which means you just need to give the required state, React would render it for you anyway, you do not need to know how it's implemented. So it would be good to use state machine diagram to explain the different states and easy to convert the diagram to codes.

### State machine diagram

Here are the diagram:

![Folder Flip State machin diagram](/images/folderflip/FolderFlip-flat.jpg "State machine diagram")

### Define variables and states

We define some concepts first:

1. We define a **window** here, which means the items shown in the screen, and we set it as 3 here;
2. `WS` or `windowStart` is the start value of window and its value is `START`. The original value is 0 and betwee 0 and `LENGTH - windowSize`;
3. `edge element` is the upper element which would be checked if it reaches threshold 100%
4. `showup element` is the lower element which would be checked if it reaches threshold 0

And we can see the variables as well:
  1. `reach100` -> boolean, indicates if the current observed edge element reach threshold 100%
  2. `reach0` -> boolean, indicates if the current observed showup element reaches threshold 0
  3. `edgeIndex` -> indicates the observed edge element index in edge element array, the original value is `windowStart + windowSize - 1`
  4. `showupIndex` -> indicates the observed showup element index in showup element array, the original value is `windowStart + windowSize`
  5. `sectionState` -> **STICKY** or **FLOAT**, indicates the current state of component
  6. `windowStart` -> window start value, initial value is 0 and range is >= `0` and <= `LENGTH - window size`

According to the state machine diagram, there are three types of states:
  1. the normal state, it's normally stable (yellow one)
  2. the state triggered by user scroll behavior (pink one)
  3. the state should be updated internally (blue one)

We will handle each scenario which triggered and started by the scroll event which is **solid** line in the diagram. One entire process should be end to the normal state whose color is *yellow*. From the diagram, we can see one process should have three states, except two edge situations.
1. The process is triggered by scrolling down, starting from the original state and transiting to the normal state directly, without `scroll` state (pink) and `internal` state (blue);
  <div style="text-align:center;">
    <img src="/images/folderflip/start-state-scroll-down.png" alt="Start state scroll down" width="600"/>
  </div>
2. The process is triggered by scrolling up from the final normal state and transites to the normal state directly as well.
  <div style="text-align:center;">
    <img src="/images/folderflip/final-state-scroll-up.png" alt="Final state scroll up" width="400"/>
  </div>

We need to handle these two situations separately.

## With only two IntersectionObservers

Although there are 6 (for example) items in the list, actually we only need two active observers. One is for the edge element, the other for showup element, although these two elements are not fixed. So why wouldn't we just create two observers and update the observer's observed element dynamically to get the correct state.

<iframe src="https://codesandbox.io/embed/folderflip-two-observers-pc9bwi?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="folderflip-two-observers"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

As you see, it does work if we scroll showly and carefully. But if we swipe the page fast, something begins going wrong. Why? Because when we swipe too fast, the observer could not change to the correct element before observing the changing.

## IntersectionObserver observes multiple elements???

OK, the solution with only two observers does not work. But actually one `IntersectionObserver` can observe multiple elements, like this:

```js
// Create a new observer
let observer = new IntersectionObserver(function (entries) {
	entries.forEach(function (entry) {
		console.log(entry.target);
		console.log(entry.isIntersecting);
	});
});

// The elements to observe
let div1 = document.querySelector('#div-1');
let div2 = document.querySelector('#div-2');

// Attach them to the observer
observer.observe(div1);
observer.observe(div2);
```

so would using one observer on multiple elements save some resources? 

The answer is no. Not just because there is no big difference, but also it does not work as we expect. According to this [blog](https://gomakethings.com/options-settings-and-approaches-with-the-vanilla-js-intersection-observer-api/), **only elements that have changed show up in the entries array**. So if the element's state not changed, the state would not be in the parameters of observer's callback function, which means we cannot get the correct state of desired element.

So actually when we change the state by scrolling behavior or internal updating, we need the state of the observed element which can be saved in an array. When we change the observed element, we just read the state from that array.

## Observer Array

So we need to setup an array for every type elment (edge, showup) which saves the value of if the elements reaches the threshold, 0 or 100%. Therefore, we have to create observer for each element. Would it effect the performance? Luckily the answer is also no. According to previous mentioned blog, there is no difference of using many observers with one element each.

> A few years ago, there was [a discussion about the performance implications of using this approach](https://github.com/w3c/IntersectionObserver/issues/81) on the w3c GitHub repository for this specification.    
> The general conclusion was that using many observers with one element each and one observer with many elements should be about equally performant...

```js
const [edgeElementIndex, setEdgeElementIndex] = useState(windowSize - 1);
const [showupElementIndex, setShowupElementIndex] = useState(windowSize);

// save the edge and showup elements, it should be stable
const edgeElements = useMemo(
  () => [].slice.call(contentElements, windowSize - 1, stepLength),
  [contentElements, stepLength]
);

const showupElements = useMemo(
  () => [].slice.call(contentElements, windowSize - 1, stepLength),
  [contentElements, stepLength]
);

// save all the states of edge and showup elements in the array and
// get their states update whenever observers are triggered, init values are false
const [edgeStates, setEdgeStates] = useState([]);
const [showupStates, setShowupStates] = useState([]);

// initial the content elements
useEffect(() => {
  let tags = elementRef.current.getElementsByClassName("FolderFlip-Tag");
  // get the height of the tag
  setTagHeight(tags[0].getBoundingClientRect().height);
  setContentElements(
    elementRef.current.getElementsByClassName("FolderFlip-Content")
  );
}, []);

// the callback function to handle when the folder reaches edge with scrolling down
// keep updating the state according to observers no matter if the element's state is used
const reachEdgeFunc = ([entry], index) => {
  setEdgeStates((v) => {
    let value = [...v];
    value[index] = entry.isIntersecting || entry.boundingClientRect.top < 0;
    return value;
  });
};

const folderShowUpFunc = ([entry], index) => {
  setShowupStates((v) => {
    let value = [...v];
    value[index] = entry.isIntersecting;
    return value;
  });
};

// set up the observer for edge element with threshold 100%
useIntersection(edgeElements, reachEdgeFunc, {
  threshold: 1.0
});

// set up the observer for showup element with threshold 0%
useIntersection(showupElements, folderShowUpFunc, {
  threshold: 0
});
```

We use `useMemo` to save the `edgeElements` and `showupElements` to avoid re-rendering. And create arrays `edgeStates` and `showupStates` to save the states of all the elements. To get the correct observed element's state, we also need `edgeIndex` and `showupIndex`. When certain element reaches the threshold and triggers the callback function, it passes `entry` and the index in state array.

`useIntersection` needs to update as well:

```js
function useIntersection(nodeElements, callbackFunc, options) {
  let observers = useMemo(() => {
    if (typeof IntersectionObserver === "undefined") return;

    return nodeElements.map(
      (_, i) =>
        new IntersectionObserver(
          (entries) => {
            callbackFunc(entries, i);
          },
          {
            threshold: options.threshold
          }
        )
    );
  }, [nodeElements, callbackFunc, options.threshold]);

  useEffect(() => {
    observers.forEach((observer, i) => {
      if (nodeElements[i]) {
        if (observer) observer.observe(nodeElements[i]);
      }
    });

    return () => {
      observers.forEach((observer) => {
        if (observer) observer.disconnect();
      });
    };
  }, [nodeElements, observers]);
}
```

But here comes another problem, it seems too many `useState`, and each of them hangs out with others, to handle the logic, it's better to put them in one function. The state transition could be processed there. How to implement this?

## `useReducer` makes my day

The answer is `useReducer` in React.

`reducer` and `dispatcher` are the concepts from `Redux`, even though we do not use it here, but `useReducer` was introduced to React as well. 

```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

So we can handle all the variables in the `reducer` and update the UI according to the returned `state`. Also use `dispatch` to send the state from observer's callback function.

```js
// the callback function to handle when the folder reaches edge with scrolling down
// keep updating the state according to observers no matter if the element's state is used
const reachEdgeFunc = useCallback(
  ([entry], index) =>
    dispatchFunc({
      type: REDUCER_TYPE.edge,
      payload: {
        index,
        value: entry.isIntersecting || entry.boundingClientRect.top < 0
      }
    }),
  [dispatchFunc]
);

const folderShowUpFunc = useCallback(
  ([entry], index) =>
    dispatchFunc({
      type: REDUCER_TYPE.showup,
      payload: { index, value: entry.isIntersecting }
    }),
  [dispatchFunc]
);
```

Here we use `useCallback` to avoid re-rendering in hooks `useIntersection`. And the payload contains the element index and the state of element. 

The `reducer` takes `state` and `action` as parameters and should be pure, which means with the same input, the output should also not change. Note: Within `StrictMode` of React, the reducer would be called twice with same value.

```js
function FolderFlipReducer(state, action) {
  if (!action) return state;

  // set the value of edge state with given index
  if (action.type === REDUCER_TYPE.edge) {
    state.edgeStates[action.payload.index] = action.payload.value;
  } else if (action.type === REDUCER_TYPE.showup) {
    state.showupStates[action.payload.index] = action.payload.value;
  }

  let edgeIndex = state.edgeIndex,
    showupIndex = state.showupIndex,
    sectionState = state.sectionState,
    windowStart = state.windowStart;

  const reach0 = state.showupStates[state.showupIndex - windowSize + 1],
    reach100 = state.edgeStates[state.edgeIndex - windowSize + 1];

  // if the prev state is initial stable state, just update the section state
  if (!reach0 && reach100 && edgeIndex + 1 === showupIndex) {
    sectionState = SECTION_STATE.float;
    return {
      ...state,
      sectionState
    };
  }

  // if the prev state is final stable state
  if (reach0 && !reach100 && edgeIndex === showupIndex) {
    sectionState = SECTION_STATE.sticky;
    return {
      ...state,
      sectionState
    };
  }

  // all the other four situations would need to be handled under state type scroll
  // handle the pink ones in state machine diagram

  // if edge and showup observed elements are the same, set the state as float
  if (edgeIndex === showupIndex) sectionState = SECTION_STATE.float;

  // otherwises, sticky
  if (edgeIndex + 1 === showupIndex) sectionState = SECTION_STATE.sticky;

  // need to update the edge and showup index in the internal state type

  if (sectionState === SECTION_STATE.float) {
    if (reach0 && reach100) {
      if (windowStart + windowSize < state.stepLength)
        showupIndex = windowStart + windowSize;
    } else if (!reach0 && !reach100) {
      windowStart = state.windowStart > 0 ? state.windowStart - 1 : 0;
      edgeIndex = state.windowStart + windowSize - 2;
    }
  }

  if (sectionState === SECTION_STATE.sticky) {
    if (!reach0 && !reach100) {
      if (windowStart > 0) showupIndex = windowStart + windowSize - 1;
    } else if (reach0 && reach100) {
      edgeIndex = windowStart + windowSize;
      windowStart =
        state.windowStart + windowSize < state.stepLength
          ? state.windowStart + 1
          : state.stepLength - windowSize;
    }
  }

  return {
    ...state,
    windowStart,
    edgeIndex,
    showupIndex,
    sectionState
  };
}
```

So til now, we have explained and presented the solution, the page works quite stable. Below is the full codes and welcome any comments.

<iframe src="https://codesandbox.io/embed/folder-flip-reducer-j8h6xz?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="folder-flip-reducer"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

