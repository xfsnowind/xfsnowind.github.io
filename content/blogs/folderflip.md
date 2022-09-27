---
title: "How to implement a FolderFlip with React"
date: 2022-09-27T11:15:24+08:00
author: "Feng Xue"
tags: ["Javascript", "React", "FolderFlip", "Frontend"]
toc: true
usePageBundles: false
draft: false
---

Haven't updated the blogs for a long time. Just had been struggling on the house work during the whole summer time, painting external and internal wall, new bathroom and etc. But there is still the good news, implemented an interesting frontend component with React, which would inspired by [lifeatspotify](http://lifeatspotify.com/) - borrow the name `FolderFlip`.

The original idea was come up with by the UX designer in our team, she would like to develop a fancy component which can be used to present the company culture. Here is how it looks like:

<div style="text-align:center;">
  <video width="600" height="600" controls>
    <source src="/images/folderflip/screenrecord.mp4" type="video/mp4">
  </video>
</div>

After investing, I found it can be done with the css feature `position: sticky` and javascript's `IntersectionObserver`.

## Tip: `position: sticky`

This is not a new feature, but I rarely used it before because of not fully supported by all the browsers before. But now definitely it's supported by all the main stream browsers. Check [CanIUse](https://caniuse.com/?search=position%3Asticky).

Let's start with creating a list with three items which consists of a title and some simple texts as the content. And before the list, it also has some texts.

<iframe src="https://codesandbox.io/embed/lucid-stitch-y9t8iy?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
  title="folderflip-version1-sticky"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

We can see when we scroll down and the list enters the screen, all the three titles would always be inside of screen with setting the value of `top`, `bottom` and `margin-top`. And display the titles in order according to the item's index in the list.

```js
marginTop: `${tagHeight * idx}px`,
top: `${idx * tagHeight}px`,
bottom: `${(stepLength - idx - 1) * tagHeight}px`
```

Here there is one thing I would like to mention. When we use `position: sticky`, the stickied item would be attached to its parent node, to make all the titles have the same parent node, we use [React's fragment](https://reactjs.org/docs/fragments.html) to compose each item.

```html
<React.Fragment key={"FolderFlipStep" + Title.value + idx}>
  <div id={id}></div>
  <a
    href={"#" + id}
    className="FolderFlip-Tag">
    <span className="FolderFlip-Tag-Number" />
    <h2>{Title.value ?? ""}</h2>
  </a>
  <div className="FolderFlip-Content">
    <span className="FolderFlip-Content-Title">{Title ?? ""}</span>
    <div className="FolderFlip-Content-Container">
      <div>{Ingress}</div>
      <button>{Button}</button>
    </div>
  </div>
</React.Fragment>
```

OK, now it seems we have fixed the most important feature of the component. Nja, kind of. Actually, maybe you have found it when we keep scrolling down (there are some texts under the list as well) and beyond the list, the titles are still sticky and only contents move up. Definitely the title should move together with contents fluently. How do we solve this?

## Floating with IntersectionObserver

It comes the js API `IntersectionObserver`, which observes how the node intersects with the specified master node (defaultly and normally it's the screen) in the non-main process. For detail and description, you can check [Mozilla's doc](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API).

```javascript
let observer = new IntersectionObserver(
  ([entry]) => {
    console.log("reach 100%");
  },
  {
    threshold: 1.0
  }
);

observer.observe(element);
```

In this example, when the node reaches 100% in the screen, it would print out the log.

Therefore, the logic would be simple, when the content of the last item reaches the threshold 100% (taking the screen as master), we would change the items inside the screen from `position: sticky` to `position: relative` to allow the items float.

```javascript
let observer = new IntersectionObserver(
  ([entry]) => {
    setIntersection(
      entry.isIntersecting || entry.boundingClientRect.top < 0
    );
  },
  {
    threshold: 1.0
  }
);

useEffect(() => {
  observer.observe(textRef.current);

  return () => {
    observer.disconnect();
  };
}, [observer]);
```

In the code, the variable `observer` would be created every time when the page renders which would disconnect the observer and observe the same element again in `useEffect`. To avoid this, we can use `useMemo` to reserve the `observer` from rendering and we can pass a `memorized` callback function to deal with the entries. And we can create a new hooks to handle this:

```javascript
function useIntersection (textRef, callbackFunc) {
  let observer = useMemo(() => {
    return new IntersectionObserver(callbackFunc,
      {
        threshold: 1.0
      }
    );
  }, [callbackFunc]);

  useEffect(() => {
    if (textRef?.current) observer.observe(textRef.current);

    return () => {
      observer.disconnect();
    };
  }, [textRef, observer]);
}
```

And we can use this hooks to observe the last item of the list.

```javascript
  const textRef = useRef(null);

  const callbackFunc = useCallback(
    ([entry]) =>
      setIntersection(entry.isIntersecting || entry.boundingClientRect.top < 0),
    []
  );

  useIntersection(textRef, intersectCallbackFunc);

  ...

  return (<React.Fragment>
  <div
    className="FolderFlip-Content"
    ref={stepLength - 1 == idx ? textRef : undefined}
    ></div>
  </React.Fragment>)
```

Notice that the textRef is a React `ref` which would not trigger the execution of `useEffect` when it changes.

<iframe src="https://codesandbox.io/embed/folder-flip-version1-i2g1p2?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
  title="folder-flip-version1"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

Now we can see when we keep scrolling down, the whole items move out of the screen fluently.

## What if more items?

Till now, we have implemented the component. And maybe someone has noticed that we have only three items in the example, what if there are more items, like 6 or more? And actually the title of items would take over the whole screen, the content of the item would only have a very small part of the screen or even cannot show, especially in mobile. How could we fix that?

The solution to this in simple would be that we set maximum value of displayed items in the screen, like 3, no matter how many items we have. And definitely it is more complicated and will be explained in the next article.