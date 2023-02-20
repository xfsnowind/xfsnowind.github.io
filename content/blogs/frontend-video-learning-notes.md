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

| MP4(MPEG-4)                                                                                                             | MKV                                                                                                                  | WEBM                              | MPEG                                                                     | AVI                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|-----------------------------------|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
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

There are also some other commercial players, like Flowplayer, JW player, etc.

an HTML5 player is a JavaScript package that uses the Media Source Extensions API, or MSE. This API, combined with Encrypted Media Extensions (EME, for features like enhanced security and DRM), and VTTCue (for subtitles) allows developers to use JavaScript to override how browsers handle video tags and improve streaming video delivery. 

## Video streaming (Video on Demand - VOD)

RTMP, WebRTC, HLS, DASH, progressive download


There are two main protocols for video streaming, HLS (Http live streaming) and DASH (Dynamic Adaptive Streaming over HTTP), 

![Difference between HLS and DASH](/images/video/hls-dash.png "Difference between HLS and DASH")

## Live streaming

https://www.wowza.com/blog/streaming-protocols
