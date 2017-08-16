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

## 使用注意事项

### 音频格式
+ 音频处理的时候webrtc一次仅能处理10ms数据,小于10ms的数据不要传入，因为即使是传入小于10ms的数据最后传入也是按照10ms的数据传出，此时会出现问题。另外支持采样率也只有8K，16K,32K三种，不论是降噪模块，或者是回声消除增益等等均是如此。
+ 对于8000采样率，16bit的音频数据，10ms的时间采样点就是80个，一个采样点16bit也就是两个字节，那么需要传入WebRtcNsx_Process的数据就是160字节。对于8000和16000采样率的音频数据在使用时可以不管高频部分，只需要传入低频数据即可，但是对于32K采样率的数据就必须通过滤波接口将数据分为高频和低频传入，传入降噪后再组合成音频数据。大于32K的音频文件就必须要通过重采样接口降频到对应的采样率再处理，在demo源码里面有对应的接口使用者可以去查。


### 接口类型
比如回声消除接口
``` cpp
int32_t WebRtcAec_Process(void* aecInst,
                          const float* const* nearend,
                          size_t num_bands,
                          float* const* out,
                          size_t nrOfSamples,
                          int16_t msInSndCardBuf,
                          int32_t skew);
```

+ webrtc接口使用类型为float类型，音频数据类型可能是short类型，网络传输类型可能是char类型，需要进行转换。
+ webrtc接口为const float\* const\*，float\* const\*之类的，不是很常见，需要转换。

#### char转short, short转float
``` cpp
#define  NN 160

char far_frame_c[NN * 2] = { 0 };
short far_frame_s[NN] = { 0 };

FILE *fp_far = fopen("far.pcm", "rb");
fread(far_frame_c, sizeof(char) * 2, NN, fp_far);

for (int i = 0; i < NN; ++i)
{
    far_frame_s[i] = (far_frame_c[i * 2 + 1] << 8) | (far_frame_c[i * 2] & 0xFF);//两个char型拼成一个short

    far_frame_f[i] = far_frame_s[i];//转float型接口需要
}
```

#### float转short

``` cpp
#define  NN 160

short out_frame_s[NN] = { 0 };
float out_frame_f[NN] = { 0 };

for (int i = 0; i < NN; ++i)
{
    out_frame_s[i] = out_frame_f[i];
}
```

#### float\*转const float\* const\*
``` cpp
#define  NN 160

float near_frame_f[NN] = { 0 };

float* const p = near_frame_f;
const float* const* nearend = &p;
```

#### float\*转float\* const\*
``` cpp
#define  NN 160

float out_frame_f[NN] = { 0 };

float* const q = out_frame_f;
float* const* out = &q;
```