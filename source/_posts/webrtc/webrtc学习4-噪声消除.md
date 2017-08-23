---
title: webrtc学习4-噪声消除
date: 2017-08-23 10:21:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 一、背景
**NS**（Noise Suppression，噪声抑制），webrtc的噪声处理模块源码在“webrtc/modules/audio_processing/ns”内，包括两种方式：
+ noise_suppression.h 去噪浮点算法
+ noise_suppression_x.h 去噪定点算法,可以在性能较低的嵌入式设备上使用

具体去噪原理可以参考webrtc源码。

## 二、注意事项
+ webrtc默认接口都是只支持输入10ms的采样数据，并且只支持8000，16000，32000的采样率，非上述类型采样率，需要重采样后才能进行处理。
+ 最新61版本，去噪模块支持输入32k的采样率，但采样个数为160，与上述不符合，需要进一步研究。

<!--more-->

### Demo

#### C++ code
``` cpp
#include <webrtc/modules/audio_processing/ns/noise_suppression.h>   //去噪浮点算法

struct SampleConfig
{
    int channels = 1;       //采样声道数
    int sampleRate = 44100; //采样频率
    int bitsPerSample = 16; //采样位数
};

int GetSamplesPer10ms(int fs)
{
    switch (fs)
    {
    case 8000:
        return 80;

    case 16000:
    case 32000:
        return 160;

    default:
        return -1;
    }
}

/*
**@fs 采样率
**@mode 设置噪声抑制的级别 0: Mild-轻微, 1: Medium-中等 , 2: Aggressive-积极的
*/
void TestNoiseSuppression(const char* filePathIn, const char* filePathOut, int mode, SampleConfig cfg)
{
    NsHandle *nsInst = nullptr;

    FILE *fpIn = fopen(filePathIn, "rb");
    FILE *fpOut = fopen(filePathOut, "wb");

    int samples = GetSamplesPer10ms(cfg.sampleRate);

    char  *frame_in_c = new char[samples * 2];    //读取文件的字节
    short *frame_in_s = new short[samples];       //声音存储short数据
    float *frame_in_f = new float[samples];       //声音存储float数据

    short *frame_out_s = new short[samples];      //ns后float转short数据
    float *frame_out_f = new float[samples];      //ns后返回得到的float数据

    do
    {
        if (!fpIn || !fpOut)
        {
            std::cout << "open file" << std::endl;
            break;
        }

        nsInst = WebRtcNs_Create();
        if (!nsInst)
        {
            std::cout << "WebRtcNs_Create error" << std::endl;
            break;
        }

        if (0 != WebRtcNs_Init(nsInst, cfg.sampleRate))
        {
            std::cout << "WebRtcNs_Init error" << std::endl;
            break;
        }

        //设置噪声抑制的级别 0: Mild-轻微, 1: Medium-中等 , 2: Aggressive-积极的
        if (0 != WebRtcNs_set_policy(nsInst, mode))
        {
            std::cout << "WebRtcNs_set_policy error" << std::endl;
            break;
        }

        int spanTick = GetTickCount();

        while (1)
        {
            if (samples == fread(frame_in_c, sizeof(char) * 2, samples, fpIn))
            {
                //1
                for (int i = 0; i < samples; ++i)
                {
                    frame_in_s[i] = (frame_in_c[i * 2 + 1] << 8) | (frame_in_c[i * 2] & 0xFF);//两个char型拼成一个short
                    frame_in_f[i] = frame_in_s[i];//转float型接口需要
                }

                //2
                float* const p = frame_in_f;
                const float* const* spframe = &p;

                float* const q = frame_out_f;
                float* const* outframe = &q;

                //3
                WebRtcNs_Analyze(nsInst, frame_in_f);
                WebRtcNs_Process(nsInst, spframe, 1, outframe);
                for (int i = 0; i < samples; ++i)
                {
                    frame_out_s[i] = frame_out_f[i];
                }
                fwrite(frame_out_s, sizeof(short), samples, fpOut);
            }
            else
            {
                break;
            }
        }

        spanTick = GetTickCount() - spanTick;
        std::cout << "ns spanTick:" << spanTick << std::endl;
    } while (0);

    //clean
    if (nsInst) WebRtcNs_Free(nsInst);
    if (fpIn) fclose(fpIn);
    if (fpOut) fclose(fpOut);

    delete[]frame_in_c;
    delete[]frame_in_s;
    delete[]frame_in_f;
    delete[]frame_out_s;
    delete[]frame_out_f;
}
```

#### 测试文件
噪声测试音频 [1C_16bit_32K_lhydd.pcm](http://ooid5jqhw.bkt.clouddn.com/1C_16bit_32K_lhydd.pcm)