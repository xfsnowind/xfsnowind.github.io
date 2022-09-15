---
title: "Folderflip Version2"
date: 2022-09-09T13:13:55+02:00
author: "Feng Xue"
draft: true
usePageBundles: false
---

## How to implement a FolderFlip - 2

### The Problem

### Clear the logic first

![Folder Flip State machin diagram](/images/folderflip/FolderFlip-flat.jpg "State machine diagram")


### With only two IntersectionObservers

<iframe src="https://codesandbox.io/embed/folderflip-two-observers-pc9bwi?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="folderflip-two-observers"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

### IntersectionObserver observe multiple elements???

https://gomakethings.com/options-settings-and-approaches-with-the-vanilla-js-intersection-observer-api/

### Observer Array

<iframe src="https://codesandbox.io/embed/folder-flip-ejzdjy?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="folder-flip"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

### useReducer makes my day

<iframe src="https://codesandbox.io/embed/folder-flip-reducer-j8h6xz?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="folder-flip-reducer"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>
