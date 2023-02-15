---
title: "Draw a flexible line with React Konva"
date: 2023-02-15T16:20:25+08:00
author: "Feng Xue"
tags: ["Typescript", "Frontend", "React", "React konva", "konva", ]
toc: true
usePageBundles: false
draft: false
---

These days I need to implement a component to draw a line on the given image and this line should be scalable, draggable and limited within the image. So here are the steps to implement it.

## Konva

Definitely we need `canvas` to complete it, but instead of using `canvas` directly, it's better to use some mature library, like [`Konva`](https://konvajs.org/) - an HTML5 Canvas JavaScript framework that extends the 2d context by enabling canvas interactivity for desktop and mobile applications.

## Setup the canvas

Let's set up the canvas with Konva. We assign the node's width as canvas's width. But we would not like to scale the image, so we need to get the image's height.

```javascript
const stageRef = useRef<HTMLDivElement | null>();
// calculate the width of parent's node as canvas's width
const stageWidth = stageRef?.current?.offsetWidth || 400;
return (
  <div
    ref={stageRef}
    style={{
      width: '100%',
      height: '100%',
    }}
  >
    <Stage
      width={stageWidth}
      height={imgHeight} // we need to get the related image height
    >
      <Layer>
        //...
      </Layer>
    </Stage>
  </div>
)
```

## Display the image

First we need to display the image as the background. Since Konva's `Image` do not accept string as input, we need to generate an image html element. Give the image source string, set the image instance src when it loads.

```js
const useLineCrossingImage = ({ imgSrc }: { imgSrc: string }) => {
  const [image, setImage] = useState<HTMLImageElement | undefined>()

  // load image with given base64 string src
  useEffect(() => {
    const imageInstance: HTMLImageElement = new window.Image()
    const updateImage = () => {
      setImage(imageInstance)
    }
    imageInstance.src = imgSrc
    imageInstance.addEventListener('load', updateImage)
    return () => {
      imageInstance.removeEventListener('load', updateImage)
    }
  }, [imgSrc])

  return <Group>
    <Image
      image={image}
      onMouseEnter={(e) => updateMouseCursor(e, 'crosshair')}
    />
  </Group>
}
```

The image should fit the parent's width with original ratio, so we need the input of width and calculate the height based on it and the ratio.

```js
const useLineCrossingImage = ({ imgSrc, width }: { imgSrc: string; width: number }) => {
  const [image, setImage] = useState<HTMLImageElement | undefined>()
  const [imgHeight, setImgHeight] = useState<number>(DEFAULT_WIDTH_HEIGHT)

  // load image with given base64 string src
  useEffect(() => {
    const imageInstance: HTMLImageElement = new window.Image()
    const updateImage = () => {
      // calculate the related height with width and not changing ratio
      const height = (width / imageInstance.width) * imageInstance.height
      imageInstance.width = width
      imageInstance.height = height
      setImgHeight(height)
      setImage(imageInstance)
    }
    imageInstance.src = imgSr
    imageInstance.addEventListener('load', updateImage)
    return () => {
      imageInstance.removeEventListener('load', updateImage)
    }
  }, [imgSrc, width])

  return {
    imgHeight,
    instance: (
      <Group>
        <Image
          image={image}
          onMouseEnter={(e) => updateMouseCursor(e, 'crosshair')}
        />
      </Group>
    ),
  }
}
```

So we return the image's instance and height which would be used in the `Stage`.

## Draw line

So the background image is done now, let's draw the line on it.

First we need to set the start and end point with Konva's type `Vector2d`

```typescript
export interface Vector2d {
  x: number;
  y: number;
}

//...


const [startPoint, setStartPoint] = useState<Vector2d | null>(null);
const [endPoint, setEndPoint] = useState<Vector2d | null>(null);
```

When we click on the image, the start point should be set, and the end point would be set when we release the mouse after dragging. So a mouse down and mouse up event should be setup.

```js

  const [value , setValue] = useState<ImageLineCrossingFormType>()

  const [isDuringNewLine, setIsDuringNewLine] = useState<boolean>(false);

  const handleMouseDown = (e: Konva.KonvaEventObject<MouseEvent>) => {
    const target = e?.target;

    // Draw a new line again if click on the image not the Group
    if (target.getClassName() === "Image") {
      const stage = target?.getStage();
      if (stage && stage.getPointerPosition()) {
        setIsDuringNewLine(true);
        setStartPoint(stage.getPointerPosition());
        // remove previous end point when start a new line
        setEndPoint(null);
      }
    }
  };

  const handleMouseUp = (e: Konva.KonvaEventObject<MouseEvent>) => {
    const target = e?.target;
    // NOTE: finish the line only when the target is image
    if (target.getClassName() === "Image" && isDuringNewLine) {
      const stage = target?.getStage();
      if (stage && stage.getPointerPosition()) {
        const endValue = stage.getPointerPosition();
        setIsDuringNewLine(false);
        setEndPoint(endValue);
        // save the value to the form
        console.log({
          x1: (startPoint?.x ?? 0) / stageWidth,
          y1: (startPoint?.y ?? 0) / imgHeight,
          x2: (endValue?.x ?? DEFAULT_WIDTH_HEIGHT) / stageWidth,
          y2: (endValue?.y ?? DEFAULT_WIDTH_HEIGHT) / imgHeight,
        });
      }
    }
  };

  // calculate the width of parent's node as canvas's width
  const stageWidth = stageRef?.current?.offsetWidth || DEFAULT_WIDTH_HEIGHT;

  // with given width, calculate the related height without changing ratio of image
  // and get the image canvas instance
  const { imgHeight, instance: ImgInstance } = useLineCrossingImage({
    imgSrc,
    width: stageWidth,
  });


  return (
    <Stage
      width={stageWidth}
      height={imgHeight}
      onMouseDown={handleMouseDown}
      onMouseUp={handleMouseUp}
    >
      //...
    </Stage>
  )
```

To make sure we can draw a new line, we need to make sure the item clicked is the image. And during the drawing, we should lock this process and set the end point only after the start point being set. To achieve that, `isDuringNewLine` is used to indicate this procedure.

```js
const target = e?.target;
// NOTE: finish the line only when the target is image
if (target.getClassName() === "Image" && isDuringNewLine) 
```

To display the line, we are gonna use the konva's class [`Line`](https://konvajs.org/api/Konva.Line.html) with only two points. To distinguish the line from other objects, set the mouse cursor as `grab`.

```js
<Line
  points={[ startPoint.x, startPoint.y, endPoint.x, endPoint.y ]}
  stroke="green"
  strokeWidth={6}
  onMouseEnter={(e) => updateMouseCursor(e, 'grab')} // use grab cursor for line
/>
```

## Scale points

The line's start and end points should also be draggable to reset their values. We can draw two circle objects around the points with Konva's class [`Circle`](https://konvajs.org/api/Konva.Circle.html).

```js
<Circle
  x={startPoint.x}
  y={startPoint.y}
  draggable // circle can be dragged to extend the line
  onMouseEnter={(e) => updateMouseCursor(e, 'pointer')} // the cursor is pointer
  fill="white"
  stroke="green"
  strokeWidth={3}
  radius={6}
/>
```

The circles can be dragged now, but they are not attached to the line, when we move the circle points, the line's points are not updated. So we need to set them as a Group.

## Group

When we want to transform multiple shapes together with the same operation, [`Group`](https://konvajs.org/api/Konva.Group.html) can be applied. And one thing needs to be careful, the position of the whole Group is absolute to the [`Stage`](https://konvajs.org/api/Konva.Stage.html), while all the positions within the Group would be related to the Group.

Let's name the start point of `Group` as `groupAbsoluteStart`. The relative position of end point should also be applied within the Group.

```js
<Group
  draggable
  // NOTE: The Group x/y should use the absolute position, we use it as start point
  x={groupAbsoluteStart.x}
  y={groupAbsoluteStart.y}
>
  <Line
    points={[ // NOTE: the node inside of group should use relative position
      0,
      0,
      groupAbsoluteEnd.x - groupAbsoluteStart.x,
      groupAbsoluteEnd.y - groupAbsoluteStart.y,
    ]}
    stroke="green"
    strokeWidth={6}
    onMouseEnter={(e) => updateMouseCursor(e, 'grab')} // use grab cursor for line
  />
  {groupAbsoluteStart && (
    <Circle
      // NOTE: the start point of start circle should always have the static relative position to the Group
      x={0}
      y={0}
      draggable // circle can be dragged to extend the line
      onMouseEnter={(e) => updateMouseCursor(e, 'pointer')} // the cursor is pointer
      fill="white"
      stroke="green"
      strokeWidth={3}
      radius={6}
    />
  )}
  {groupAbsoluteEnd && (
    <Circle
      // NOTE: use the relative position inside of the Group
      x={groupAbsoluteEnd.x - groupAbsoluteStart.x}
      y={groupAbsoluteEnd.y - groupAbsoluteStart.y}
      draggable
      onMouseEnter={(e) => updateMouseCursor(e, 'pointer')}
      fill="white"
      stroke="green"
      strokeWidth={3}
      radius={6}
    />
  )}
</Group>
```

To get the points' positions, we can obtain the absolute positions of the Group on the end of dragging with event `onDragEnd`. 
1. save the start position on dragging start;
2. on the end of dragging, start position can be obtained through target's method `getAbsolutePosition`;
3. calculate the length of the line according to the previous start/end points
4. calculate the new end points' positions
5. save the new start and end positions

```js
const [savedStartPoint, setSavedStartPoint] = useState<Vector2d | null>(null)

<Group
  draggable
  // NOTE: The Group x/y should use the absolute position, we use it as start point
  x={groupAbsoluteStart.x}
  y={groupAbsoluteStart.y}
  onDragStart={(e) => {
    const target = e?.currentTarget
    // 1. Save the start point to calculate moved distance when drag ends
    setSavedStartPoint({ x: target.x(), y: target.y() })
  }}
  onDragEnd={(e) => {
    const target = e?.currentTarget
    // 2. get the absolute of the group and save it as start point
    const { x, y } = target.getAbsolutePosition()
    // 3, 4. get the new end point based on the moved distance and previous end point
    const newEndPointX = x - (savedStartPoint?.x ?? 0) + groupAbsoluteEnd.x
    const newEndPointY = y - (savedStartPoint?.y ?? 0) + groupAbsoluteEnd.y
    // 5. after we finish the drag, need to update the start and end points for the future actions
    setGroupAbsoluteStart({ x, y })
    setGroupAbsoluteEnd({ x: newEndPointX, y: newEndPointY })

    // calculate the values with form's format
    SET_VALUE_WITH_NAME({ x1: x, y1: y, x2: newEndPointX, y2: newEndPointY })
  }}
/>
```

For the start/end circle points, when we drag them, the line is already attached to the circle, but the related points' values should also be updated.

To avoid triggering the drag event of `Group`, instead of calling html event's `stopPropagation`, we should set `cancelBubble` of event as `true` on the `onDragEnd` event. Check the official doc [here](https://konvajs.org/docs/events/Cancel_Propagation.html). And note that this `cancelBubble` must be done on the `onDragEnd` event.

NOTE: the relative position of the circle point would be changed, to keep the position consistent, we manually set its relative position as 0 (e.g for the start point)

```js
// Start circle point
onDragEnd={(e: Konva.KonvaEventObject<MouseEvent>) => {
  // NOTE: MUST set the cancelBubble on the drag end event
  e.cancelBubble = true

  const target = e.target
  // get the absolute position of the start circle and save it to the form
  const { x, y } = target.getAbsolutePosition()
  // Save the value to form when ends instead of during dragging
  SET_VALUE_WITH_NAME({
    x1: x / stageWidth,
    y1: y / stageHeight,
    x2: groupAbsoluteEnd.x / stageWidth,
    y2: groupAbsoluteEnd.y / stageHeight,
  })
}}
onDragMove={(e: Konva.KonvaEventObject<MouseEvent>) => {
  const target = e.target
  // NOTE: keep the circle relative position always being 0
  target.x(0)
  target.y(0)
}}
```

## Limitation

Til now, the basic feature has been implemented, drag line, circle to new position, draw a new line. But there is no border to the line, the line and its points can be dragged out of the image. To implement this, we need to check points' position during dragging. And the situation is different when the start point is left/right or higher/lower to the end.

```js
const limitValue = (xValue: number, maxValue: number, minValue = 0) =>
  Math.max(minValue, Math.min(maxValue, xValue))

<Group
  ...
  onDragMove={(e) => {
    const target = e?.currentTarget
    const { x, y } = target.getAbsolutePosition()
    // limit the move area
    const xMovedDistance = groupAbsoluteEnd.x - groupAbsoluteStart.x
    const yMovedDistance = groupAbsoluteEnd.y - groupAbsoluteStart.y

    // the range changes when the start is behind or before end
    if (xMovedDistance > 0) {
      target.x(limitValue(x, stageWidth - xMovedDistance))
    } else {
      target.x(limitValue(x, stageWidth, 0 - xMovedDistance))
    }

    if (yMovedDistance > 0) {
      target.y(limitValue(y, stageHeight - yMovedDistance))
    } else {
      target.y(limitValue(y, stageHeight, 0 - yMovedDistance))
    }
  }}
/>
```

For the circle point, we need to calculate its absolute position and limit it within the border as well.

## Summary

So this is what we want now, you can check the result on the under example and the [Codes](https://github.com/xfsnowind/react-konva-draw-line) here.

<iframe src="https://xfsnowind.github.io/react-konva-draw-line/" width='620px' height='400px' ></iframe>
