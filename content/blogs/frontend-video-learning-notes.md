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

## Concept

This article would only focus on the frontend technologies, maybe I will write something about the backend in the future as well, but let's focus frontend here. So to play a frontend html5 video/audio, we need to get familiar with some concepts first.

### Video format

The video format is a type of file to contain the digital media, like audio, video and other information, subtitles and metadata. It would determine how the data is stored and organized in the file. Differences between different formats includes:
1. Codecs, different formats still have
2. Compression efficiency: different strategies are applied to compress the video files;
3. Quality: Some format are better suited for high-quality video, while some other are designed for low-bandwidth applications;
3. Streaming support: some video formats are only designed specifically for streaming, while others support adaptive bitrate streaming and low-latency playback

The most common formats for video includes MP4, AVI, MPEG and WebM. The most famous format for audio is mp3.

### Video codecs

The video codecs is a software algorithm that compress and decompress the digital media, like audio and video. It can be used to reduce the size of files for efficient storage and transmission. The common video codecs include H.264, H.265, AV1 and VP9.

The relationship between of video format and codecs is closely related. Codecs suppported by formats are different, currently, mp4 is widely used because it supports 

So format is kind of independent from codecs, it does not care what kind of codecs when the video is produced, as long as format supports, you can use any kind of codecs. But considering the reality, most suitable codecs would be chosen with factors such as file size, video quality, compatibility with different devices and platforms, and support for features such as subtitles, closed captions, and streaming protocols. 

Normally, we would choose H.264 for video and AAC for audio.

### Video player

A video player will read the format and codecs of the video from its metadata (also other information, like audio, subtitles). It will decide if the format and codecs are supported. To play the video, the related codecs are required to decode the video or audio. That's why we have to choose the correct video player, because it needs to support the format and codecs that your audience is likely to use.

### Video streaming

#### Live streaming

## Video optimization