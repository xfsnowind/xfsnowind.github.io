---
title: "Learning Notes about Blob, Buffer, Arraybuffer with Image"
date: 2023-02-27T16:27:42+08:00
author: "Feng Xue"
tags: ["Frontend", "Blob", "Learning Notes", "Buffer", "ArrayBuffer", "Image"]
toc: true
usePageBundles: false
draft: true
---

These days when I handle the image in the frontend, there are some concepts make me confusing. So I want to write some notes about the learning notes here.

* [What is Blob, Buffer, ArrayBuffer and Base64 format?](#concept)
* [How to convert them among one another?](#conversion)
* [How can they be used with Image?](#image)

# Concept

## Blob

explain the blob definition, concept

https://developer.mozilla.org/en-US/docs/Web/API/Blob

## ArrayBuffer

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer

## Buffer

## Base64

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
