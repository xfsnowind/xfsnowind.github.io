---
title: "Frontend Video Learning Notes"
date: 2023-02-17T14:20:52+08:00
author: "Feng Xue"
tags: ["Frontend", "Video", "Learning Notes", "Live Streaming"]
toc: true
usePageBundles: false
draft: true
---

These days I am interested at the Frontend video technologies, so did some investigation and want to write learning notes here.

This article would only focus on the frontend technologies, maybe I will write something about the backend in the future maybe, but let's focus frontend here. So to play a frontend html5 video/audio, we need to get familiar with some concepts first.

OK, let's begin from the start point, video. What technologies or concept are involved when a video is played

## Video codecs

The video codecs is a software algorithm that compress and decompress the digital media, like audio and video. It can be used to reduce the size of files for efficient storage and transmission. The common video codecs include H.264, H.265, AV1 and VP9.

A simple example about the necessary of video codecs is a 30 minute video with high definition (1920 x 1080) in full code (4 bytes per pixel) would need about 447.9 GB of storage. With encoding or compression, not only the storage, but also network band-width which is more expensive are largely saved.

But different codecs have different strategies to encode the videos, like lossless and lossy.
1. Lossless compression would encode the data without discard any information. 
2. Lossy compression would discard some unnecessary data and reduce quality when possible, while easier to store and transfer. And this is the name of the game for web compression.

Here are the most common codecs being used on the web.
* AV1 - AOMedia Video 1	
* AVC (H.264) - Advanced Video Coding
* H.263 - H.263 Video
* HEVC (H.265) - High Efficiency Video Coding
* MP4V-ES - MPEG-4 Video Elemental Stream
* MPEG-1 - MPEG-1 Part 2 Visual
* MPEG-2 - MPEG-2 Part 2 Visual
* Theora - Theora
* VP8 - Video Processor 8
* VP9 - Video Processor 9

Normally, H.264 for video and AAC for audio are mostly used.

Mozilla also has a very good doc to explain the codecs [here](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Video_codecs) and wowza gives [more detail](https://www.wowza.com/blog/video-codecs-encoding) about codecs on web as well

## Video format

Once compressed, the result of audio and video would need to be packed into a file or container, which is the video format. And the video format is a type of file to contain the digital media, like audio, video and other information, subtitles and metadata. It would determine how the data is stored and organized in the file. Differences between different formats includes:
1. Codecs, different formats still have
2. Compression efficiency: different strategies are applied to compress the video files;
3. Quality: Some format are better suited for high-quality video, while some other are designed for low-bandwidth applications;
4. Streaming support: some video formats are only designed specifically for streaming, while others support adaptive bitrate streaming and low-latency playback

The relationship between of video format and codecs is closely related. Codecs suppported by formats are different, currently, mp4 is widely used.

So format is kind of independent from codecs, it does not care what kind of codecs when the video is produced, as long as format supports, you can use any kind of codecs. But considering the reality, most suitable codecs would be chosen with factors such as file size, video quality, compatibility with different devices and platforms, and support for features such as subtitles, closed captions, and streaming protocols. 


Here is the list of popular formats supported codecs:

| MP4(MPEG-4)                                                                                             | MKV                                                                                                                              | WEBM                                          | MPEG                                    | AVI                                                                                   |
| ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------- |
| <ul><li>H.264(AVC)</li><li>H.265(HEVC)</li><li>AV1</li><li>MP4V-ES</li><li>MPEG-2</li><li>VP9</li></ul> | <ul><li>H.264(AVC)</li><li>H.265(HEVC)</li><li>Xvid</li><li>DivX</li><li>MPEG-2</li><li>MPEG-4</li><li>VP8</li><li>VP9</li></ul> | <ul><li>AV1</li><li>VP8</li><li>VP9</li></ul> | <ul><li>MPEG-1</li><li>MPEG-2</li></ul> | <ul><li>Xvid</li><li>DivX</li><li>MPEG-4 SP</li><li>MPEG-4 AVC</li><li>VC-1</li></ul> |


The most common formats for video includes MP4, AVI, MPEG and WebM. The most famous format for audio is mp3.


## Video player

So the codecs and format of video file are explained, then we need a player to play it now.

A video player will read the format and codecs of the video from its metadata (also other information, like audio, subtitles). It will decide if the format and codecs are supported. To play the video, the related codecs are required to decode the video or audio. That's why we have to choose the correct video player, because it needs to support the format and codecs that your audience is likely to use. 

### Web Video player

For frontend, we have several options to play video:
1. HTML5 native video player: It is built into most modern web browsers, also supporting live streaming. It offers basic playback functionalities and can be customized with HTML, css and js.
2. Video.js: It is an open source HTML5 video player, which is highly customizable and supports a wide range of video formats. It also offers a plugin system that helps developers to add additional functioanlity to the plaoyer, such as advertising and analystics.
3. hls.js: it works directly on the H5 video element by implementing an HLS client and supports HLS
4. dash.js: it provides DASH playback in any browser that support Media Source Extension (MSE) and is one of the best adaptive streaming algorithms.

Here is the difference between H5 player and video.js:

![Difference between H5 player and video.js](/images/video/video.js-vs-h5-player.png "Difference between H5 player and video.js")

There are also some other commercial players, like Flowplayer, JW player, etc. We will not discuss them here.

Some other crucial features are also need to look out for H5 video player:
* DRM - Digital Rights management. It's used to encrypt and protect your video content from unauthorized users.
* Ad Insertion - From commercial aspect, player needs to support to insert the ad before (preroll), in the middle (midroll) and after (postroll) the video.
* Subtitles - it should allow to display the different language subtitles
* Analystics - it's valuable to measure the viewership, engagement levels and other aspect of the viewer to help author get the statistic.

## Video streaming (Video on Demand - VOD)

So we have explained the basic concept of the video and player. And it can be played locally now, but nowadays it's so common to play the video online. So we need a streaming protocol to delivery data over the internet. They can sit on the Application, Presentation and Session layers.

There are multiple protocols or solutions used:
* Progressive download
* HLS (HTTP Live Streaming)
* DASH (Dynamic Adaptive Streaming over HTTP)
* RTMP (Real-Time Messaging Protocol)
* RTSP (Real-Time Streaming Protocol)
* WebRTC (Web Real-Time Communication)
* websocket

### Traditional Stateful Streaming Protocols

RTSP (Real-Time Streaming Protocol) and RTMP (Real-Time Messaging Protocol) were created by Adobe and played by the Adobe Flash player. Even though Flash was announced of death by Adobe, RTMP and RTSP are still used for video and audio transmission for fast video delivery. Many broadcasters choose to transport live streams to their media server using RTMP. RTMP and RTSP keep latency at around 5 seconds or less.

### HTTP-Based Adaptive Streaming Protocols

HLS (supported by iOS and Android devices) and MPEG-DASH (supported by Android devices) are the most common protocols used nowadays. And these streaming over HTTP are technically not "Streams", rather are progressive downloads sent via web servers. HTTP-based protocols are stateless and can cause 10-45 seconds in latency. To reduce the latency, there are also low-latency version to HLS and DASH.

![Difference between HLS and DASH](/images/video/hls-dash.png "Difference between HLS and DASH")

Nowadays, different environments would provide different bandwidths for video delivery and could effected by different reasons. So to satisfy the user experience of a variety of devices and connection speeds, instead of creating one stream with one bitrate, HTTP-based protocols allows you create multiple streams at different bitrates and resolutions. So for ourstanding bandwidth and processing power, user can watch the high-quality streams, while user can also enjoy the lower quality video with limited speed and power instead of interruptions. And the resolution would be updated when the network speed changes.

## Live streaming

The difference of video streaming from live streaming is live streaming sending the live content generated by user's camera instead of fixed the video files. And normally live streaming has higher requirement for casting the live video to multiple users simultaneously. 

### BackPressure

The backpressure of live streaming problem in frontend can occur when the amount of data being sent from the server exceeds the capacity of the client to receive and process that data. This can lead to buffering and delays in the live stream, which can negatively impact the user experience. 

Http-based streaming protocols are designed to handle the backpressure due to their use of adaptive bitrate streaming, which allows the server to adjust the quality of the video sent according to the available bandwidth and the capability of the client device. In addition, Http-based streaming protocols work with http caching, allow the client to cache the video segments and reducing the amount of data. However, it's still possible for backpressure to occur with http-based streaming protocols if the server is under heavy load.

### Content Delivery Network

When deliverying the video content to the remote users, it may cause the latency and buffering if user is far away from the server, so a Content Delivery Network is required. To provide CDN, here are the general steps to implement:

1. Create the source stream: This can be done by a live encoder or a streaming software to send the content to the streaming server;
2. Configure the CDN - to replicate and distribute the content to multiple destinations;
3. Set up the streaming server: The streaming server is needed to receive and distribute the content;
4. Connect the streaming server to the CDN: Applying the pull or pushing mechanism to obtain the content from streaming server to CDN;

### Apple platform

Live streaming for iOS devices with HLS or DASH needs an index file with format M3U8. It is the format that is supported by the native video player in iOS, which is the AVPlayer.

### WebSocket

Websocket is a realtime protocol that enables client-server bi-directional communication over a persisitent, single-socket connection. This is largely used in the applications that require real time communication and cooperations, such as Google doc. And comparing the Http-based protocol's long polling, websocket is based on event-driven, this can help to reduce latency and improve the performance of the live stream.

But using WebSocket for live streaming [have the problem of backpressure](https://developer.chrome.com/en/articles/websocketstream/), the client cannot consume the data sent from server. So **How to solve this?** One solution would be pushing the data to the buffer when available. But what if the buffer is full? So another solution would be `speed up` the live streaming. If the current time is behind the duration, we can increase the speed based on the lag time.

### WebRTC

WebRTC is a open-sourced framework to enable the real-time communication between to web and mobile application in a peer-to-peer fashion.

The difference between WebRTC and WebSocket is 

1. WebSocket is a client-server protocol while WebRTC is a peer-to-peer protocol and offers capabilities for browsers and mobiles.
2. WebSocket works only over TCP, WebRTC is primarily used over UDP (although it can work over TCP as well).
3. WebRTC is primarily designed for streaming audio and video content. But Websocket is better suitable for text data, although it can also be used for video transmission.
4. 

### Other aspects

Some additional considerations are needed when simulcasting:

1. Bandwidth and storage requirement: the network and storage infrastructure should be able to handle the huge network load
2. Latency: configuration should be optimized to minimize the delay and latency
3. Content protection: make sure that you have the proper licenses and content protection mechanisms to prevent unauthorized access and distribution
