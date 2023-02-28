---
title: "Learning Notes about Blob, Buffer, Arraybuffer with Image"
date: 2023-02-27T16:27:42+08:00
author: "Feng Xue"
tags: ["Frontend", "Blob", "Learning Notes", "Buffer", "ArrayBuffer", "Image"]
toc: true
usePageBundles: false
draft: true
---

These days when I handle the image in the frontend, there are some concepts make me confusing. So I want to write some notes about the learning processing here.

* [What is Blob, Buffer, ArrayBuffer and Base64 format?](#concept)
* [How to convert them among one another?](#conversion)
* [How can they be used with Image?](#image)

# Concept

## ArrayBuffer

`ArrayBuffer` is an build-in object in Javascript that represents a generic, fixed-length binary data buffer. It can be used to hold raw binary data that can be accessed with types like `Int8Array`, `Unit8Arrray`, `Float32Array` and etc. ArrayBuffer is normally used to work with binary data, such as binary files and sending files over network.

But you cannot manipulate the ArrayBuffer data directly, instead, you need a typed array object or `DataView` object to read or write the content of the buffer. 

```js
const buffer = new ArrayBuffer(8);
const view = new DataView(buffer);
view.setInt16(0, 42);
view.setFloat32(2, Math.PI);
console.log(new Uint8Array(buffer));
```

You can check the detail [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)

## Blob

Blob is the abbreviation of `Binary Large OBject`. From its name, we can see it's used to save the raw data in the format of binary. And it can also be converted ReadableStream and represent some unusual data like `File`. It's similar to `ArrayBuffer`. However, `Blob` can represent the text and image files. A `Blob` object can contain the content of data and a MIME type which indicates the type of file.

```js
const text = 'Hello, world!';
const blob = new Blob([text], { type: 'text/plain' });
console.log(blob); // output: Blob { size: 13, type: "text/plain" }
```

### Blob Url

As I mentioned before, Blob can be used to represent the image file in the combination with `URL.createObjectURL` method to create a URL that points to the data contained in the `Blob` object.

```js
const imageBlob = fetch('https://example.com/image.png').then(response => response.blob());
const imageUrl = URL.createObjectURL(imageBlob);
const img = document.createElement('img');
img.src = imageUrl; // blob:XXX...
```

## Buffer

`Buffer` is another object in `Node.js` that represents binary data. It's similar to `ArrayBuffer`, but it's designed for Node.js and has some additional features.

Buffer is an object in Node.js that represents a binary data buffer. It is similar to ArrayBuffer, but it is designed for use in Node.js and has some additional functionality. For example, Buffer has methods for encoding and decoding strings, and for copying and slicing buffers.

## Base64

Base64 is a binary-to-text encoding scheme to represent binary data in an ASCII string format. It's commonly used to encode binary data, like image, audio and video files so that they can be transmitted over network.

Base64 is a binary-to-text encoding scheme that represents binary data in an ASCII string format. It is commonly used to encode binary data such as images, audio files, and video files so that they can be transmitted over networks that are designed to handle only text data.

The Base64 encoding algorithm takes a sequence of binary data and converts it to a sequence of printable characters using a fixed set of 64 characters, typically consisting of uppercase and lowercase letters, digits, and two additional symbols such as '+' and '/'. The resulting string is larger than the original binary data because each block of 3 bytes is represented as 4 characters.



# Conversion

## Blob <-> ArrayBuffer

## Blob <-> base64

## Blob <-> Buffer

## ArrayBuffer <-> Base64

## ArrayBuffer <-> Buffer

# Image

How are these formats applied when we transfer, display and save the image?

## From Url

with given url, we can display it directly in the html page by assign it to the `src` of a `img` element.

### Blob Url

https://stackoverflow.com/questions/30864573/what-is-a-blob-url-and-why-it-is-used

https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL

## Local file

Type `File`

Blob


```js
async function imageUrlToBlobUrl(url: string) {
    const blob = await (await fetch(url)).blob()
    const blobUrl = window.URL.createObjectURL(blob)
    return blobUrl
}
```
