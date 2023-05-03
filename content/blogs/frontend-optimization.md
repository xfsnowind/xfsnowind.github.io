---
title: "Frontend Optimization"
date: 2023-03-12T10:48:51+08:00
author: "Feng Xue"
tags: ["Frontend", "Optimization"]
toc: true
usePageBundles: false
draft: true
---

This one is a learning note about the optimization of frontend. 

Frontend optimization is always a big topic in the frontend technology. Fluent user experience can increase user's 

# Cache

## Local Cache

Normally we can save our transaction data in the frontend cache with `localStorage`, `sessionStorage` or `IndexDB`. For example, we can save some cookie or some data changed not that frequently in the cache to save the time for request.

### IndexDB

## Memory Cache

When we visit a page or the resources, sometimes one resource would be used multiply times, it would be save in the browser's cache. And browser recognizes them by their url, so if we want to always get the latest file, we can assign an unique (hash) string to the file name.

![Memory Cache](/images/optimization/memory-cache.png "Memory Cache")

In the image, we can see some js and image files are cached in the memory. And we can check the `Disable cache` in the browser's inspector, but it will disable both browser cache and http cache.

## Cache API

This cache api would provide the capbility to communicate with the cache in the browser

```js
// sw.js
self.addEventListener('fetch', function (e) {
  e.respondWith(
    caches.match(e.request).then(function (cache) {
      return cache || fetch(e.request);
    }).catch(function (err) {
      console.log(err);
      return fetch(e.request);
    })
  );
});
```

## Http Cache

### Disk cache

Some requests would be cached in the disk with the related HTTP headers `Expires` and `Cache-Control`.  `Expires` would be set a expired time, browser would compare the current time with it to judge if it's expired. If not, it would fetch the cache from disk directly. `Cache-Control` would set a `max-age` value to control the period. If it's not over the time, it would read the cache from disk. 

### Heuristic caching

One way to check cache is the Http tag `Last-Modified`. The server would return a value of `Last-Modified` for the first request, and browser would contain this value as `If-Modified-Since`. If the server finds the value has not changed, it would return the 304, otherwise 200.

The other way is to check `ETag`. The server would return `ETag` for the first request, and the following requests would contain this value in the header `If-None-Match`.
