---
title: webrtc学习2-音频预处理模块
date: 2017-08-14 15:38:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## VoiceEngine 
WebRtc中VoiceEngine(**VoE**)可以完成大部分的VOIP相关任务，包括采集、自动增益、噪声消除、回声抑制、编解码、RTP传输。

## APM
对于非webrtc的项目，如果需要用到webrtc中的音频算法处理模块，可以使用仅次于VoE层级的模块APM（Audio Preprocessing Module），一个纯粹的音频预处理单元。

### 源码
APM的整体编译需要WebRTC源码目录下的如下资源：
+ common_audio 整个目录
+ modules 目录（不包含 video 部分）
+ system_wrappers 整个目录
+ 位于 WebRTC 源码根目录下的 common_types.h | common.h | typedefs.h 三个头文件。

<!--more-->

### 模块
APM包括以下几个核心算法模块：

| 简写        | 英文           | 中文  |
| ------------- |:-------------:| -----:|
| **AEC** | Acoustic Echo Canceller | 声学回声消除 |
| **AECM** | Acoustic Echo Canceller for Mobile | 声学回声消除for Mobile |
| **VAD** | Voice Activity Detection | 静音检测 |
| **AGC** | Auto Gain Control | 自动增益控制 |
| **NS** | Noise Suppression | 噪声抑制 |