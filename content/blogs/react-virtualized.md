---
title: "How to implement Virtualized Grid/List in React"
date: 2022-12-03T22:22:49+08:00
author: "Feng Xue"
tags: ["Javascript", "React", "virtualized", "Frontend"]
toc: true
usePageBundles: false
draft: false
---

Around 2018, one of my colleague was working on creating a list component which only renders a limited amount of items in the list instead of the whole one to improve the performance if the list is in large or huge scale. I was always interested at how he did that, but I did not do any investment, just an idea. Then once I was asked how to implement such thing during an interview, it remindered me. I wrote it to my learning plan blog. After I began to work in the new company, I found all the lists in the product already used this idea with the library [react-virtualized](https://github.com/bvaughn/react-virtualized) and its optimized version [react-window](https://github.com/bvaughn/react-window). Finaly, I decided to learn this thing -- **virtualized**, figure out how it is implemented.

Nowadays `virtualized` has become a kind of standard for all the grid/list library, it's used in most frameworks or libraries,like mui, react-table. It helps improve the performance and user experience of large, complex, and data-intensive applications built with React. It does this by rendering only the items that are currently visible on the screen, and virtualizing the rest of the items, which allows the application to handle large datasets without negatively impacting performance or the user experienceSo here I would share my research about how it is implemented. Since I am more familiar with React, so I will use React as the framework.

## Grid/List

Normally, both `List` and `Grid` are virtualized. But since `List` is actually an one-dimension `Grid`, so let's take `Grid` as an example.

## Workflow 

To implement this feature, we need to implement in two parts: javascript and html.
With javascript, we need to calculate the start/end indexes of the visible elements. And for html, we need to paint them.

### Javascript - logic

OK, let's clear the logic firstly.
Let's imagine we have a 1000x1000 grid, only 20x20 are rendered in the table no matter how it scrolls. So to render only the visible items in the long list/grid during scrolling, it must be related to the scroll event. We need to
1. calculate the scroll offset of the whole component when scroll
2. calculate the start and end index of vertical and horizontal elements based on offset
3. generate the visible element based on the indexes

#### Scroll offset

Apparently, we need a callback event function to bind to the scroll event of the root element, calculating the offsets in vertical and horizontal directions. It can be obtained from node's property `scrollLeft` and `scrollTop`.

```javascript
const [verticalScroll, setVerticalScroll] = React.useState(0);
const [horizontalScroll, setHorizontalScroll] = React.useState(0);

// set up scroll event to update the offset of top and left
const onScroll = useCallback((event: UIEvent<HTMLDivElement>) => {
  const target = event.target as HTMLDivElement;
  const leftOffset = Math.max(0, target.scrollLeft);
  const topOffset = Math.max(0, target.scrollTop);

  setVerticalScroll(topOffset);
  setHorizontalScroll(leftOffset);
}, []);
```

### Start/end index

OK, the offsets are here now. Naturally, the start and end indexes are easy to calculate with the size of cell and the window from input.

```js
const useIndexForDimensions = ({
  offset,
  cellDimension,
  windowDimension,
}: DimensionsType) => {
  const startIndex = Math.floor(offset / cellDimension);
  const endIndex = Math.ceil((offset + windowDimension) / cellDimension);
  return [startIndex, endIndex];
};

...

// calculate the start and end row and column based on the offset
const [verticalStartIdx, verticalEndIdx] = useIndexForDimensions({
  offset: verticalScroll,
  cellDimension: cellHeight,
  windowDimension: inputWindowHeight,
});

const [horizontalStartIdx, horizontalEndIdx] = useIndexForDimensions({
  offset: horizontalScroll,
  cellDimension: cellWidth,
  windowDimension: inputWindowWidth,
});
```

### Grid cell

After getting the index, we can just render the element within the range. Just simply `slice` the data array and pass the width and height to the cell element.

```jsx
const useScrollItem = ({
  verticalStartIdx,
  verticalEndIdx,
  horizontalStartIdx,
  horizontalEndIdx,
  cellWidth,
  cellHeight,
  data,
}: ScrollItemType) =>
  useMemo(() => {
    return data.slice(verticalStartIdx, verticalEndIdx).map((row, i) => {
      const rowChildren = row
        .slice(horizontalStartIdx, horizontalEndIdx)
        .map((_, j) => {
          const vIdx = i + verticalStartIdx;
          const hIdx = j + horizontalStartIdx;
          let background = (vIdx + hIdx) % 2 === 1 ? "grey" : "white";
          return (
            <div
              key={"row-" + vIdx + "-column-" + hIdx}
              style={{
                background,
                color: "black",
                display: "flex",
                justifyContent: "center",
                alignItems: "center",
                width: cellWidth + "px",
                height: cellHeight + "px",
              }}
            >
              {vIdx}, {hIdx}
            </div>
          );
        });

      return (
        <div key={"row-" + i} style={{ display: "flex" }} >
          {rowChildren}
        </div>
      );
    });
  }, [
    verticalStartIdx,
    verticalEndIdx,
    horizontalStartIdx,
    horizontalEndIdx,
    cellWidth,
    cellHeight,
    data,
  ]);

```

### Html part

So the logic part is finished. We also need to render it correctly in the html file. At first, to limit the component in the given size, we need a root element to set the width and height.

```html
<div
  onScroll={onScroll}
  style={{
    width: `${inputWindowWidth}px`,
    height: `${inputWindowHeight}px`,
    overflow: "auto",
    position: "relative",
  }}
>
```

You can see we bind the `onScroll` callback function on this root element, and also set the `overflow` as `auto` to allow the children elements scrollable.

Since we do not paint all the elements in the DOM, we must have something to meet two requirements at the same time. 
1. We need a child element with big enough size to make the root element scrollable. And its size should allow the visible element display correctly.
2. This child element has no text to display, only has size.

```html
<div style={{
    width: `${cellWidth * data[0].length}px`,
    height: `${cellHeight * data.length}px`,
  }}
>
```

So here the width would be cell width multiple the length of the row and height would be the same. When we scroll the page, actually we are scrolling this non-text element.

Finally, we need to the parent node to display the visible elements. This is the core part, because when the previous invisible element scrolls, this child element would also have offset. To make sure it displays inside of the window, we need to do `transform` to it with the offset values calculated from the first step in javascript part.

```html
<div
  style={{
    position: "absolute",
    transform: `translate(${horizontalScroll}px, ${verticalScroll}px)`,
    display: "flex",
    flexDirection: "column",
  }}
>

```

## Deploy to Github pages

You can check the source code [xfsnowind/react-virtualized-experiment](https://github.com/xfsnowind/react-virtualized-experiment), I also have deployed it to my blog [Here](https://xfsnowind.github.io/react-virtualized-experiment/).

Actually, I have already deployed my own blog website by Hugo in Github Pages, then how could I deploy this app to a subpage of the website without effecting Hugo. Check [here](https://github.com/gitname/react-gh-pages) to deploy and [here](https://dev.to/dyarleniber/setting-up-a-ci-cd-workflow-on-github-actions-for-a-react-app-with-github-pages-and-codecov-4hnp) to add command to github actions.