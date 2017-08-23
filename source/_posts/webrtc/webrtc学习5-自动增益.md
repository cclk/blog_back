---
title: webrtc学习5-自动增益
date: 2017-08-23 10:35:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 一、背景
**AGC**（Auto Gain Control，自动增益控制），较新的webrtc已经把原来的agc模块移动到了一个叫做legacy的文件夹。具体代码路径“webrtc/modules/audio_processing/agc/legacy”。
具有三种模式：
+ kAgcModeAdaptiveAnalog带有模拟音量调节的功能；
+ kAgcModeAdaptiveDigital是可变增益agc，但是不调节系统音量；
+ kAgcModeFixedDigital是固定增益的agc；

具体原理可以参考webrtc源码。

## 二、注意事项
只支持以下采样率和帧长的输入数据：
+ 8K采样率，10或者20ms长度数据，采样数为80； 
+ 16K采样率，10或者20ms长度数据，采样数为160； 
+ 32K采样率，5或者10ms长度数据，采样数为160； 

<!--more-->

### Demo

#### C++ code
``` cpp
#include <webrtc/modules/audio_processing/agc/legacy/gain_control.h>//自动增益

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

void TestAutoGainControl(const char* filePathIn, const char* filePathOut, SampleConfig cfg)
{
    void *agcInst = nullptr;

    FILE *fpIn = fopen(filePathIn, "rb");
    FILE *fpOut = fopen(filePathOut, "wb");

    int samples = GetSamplesPer10ms(cfg.sampleRate);

    char  *frame_in_c = new char[samples * 2];    //读取文件的字节
    short *frame_in_s = new short[samples];       //声音存储short数据

    short *frame_out_s = new short[samples];      //ns后float转short数据

    do
    {
        if (!fpIn || !fpOut)
        {
            std::cout << "open file" << std::endl;
            break;
        }

        agcInst = WebRtcAgc_Create();
        if (!agcInst)
        {
            std::cout << "WebRtcAgc_Create error" << std::endl;
            break;
        }

        int minLevel = 0;
        int maxLevel = 255;
        int agcMode = kAgcModeFixedDigital;
        //kAgcModeAdaptiveAnalog 带有模拟音量调节的功能
        //kAgcModeAdaptiveDigital 是可变增益agc，但是不调节系统音量
        //kAgcModeFixedDigital是固定增益的agc
        if (0 != WebRtcAgc_Init(agcInst, minLevel, maxLevel, agcMode, cfg.sampleRate))
        {
            std::cout << "WebRtcAgc_Init error" << std::endl;
            break;
        }

        WebRtcAgcConfig agcConfig;
        agcConfig.compressionGaindB = 20;
        agcConfig.limiterEnable = 1;
        agcConfig.targetLevelDbfs = 3;
        if (0 != WebRtcAgc_set_config(agcInst, agcConfig))
        {
            std::cout << "WebRtcAgc_set_config error" << std::endl;
            break;
        }

        int spanTick = GetTickCount();
        int micLevelIn = 0;
        int micLevelOut = 0;

        while (1)
        {
            if (samples == fread(frame_in_c, sizeof(char) * 2, samples, fpIn))
            {
                //1
                for (int i = 0; i < samples; ++i)
                {
                    frame_in_s[i] = (frame_in_c[i * 2 + 1] << 8) | (frame_in_c[i * 2] & 0xFF);//两个char型拼成一个short
                }

                //2
                short* const p = frame_in_s;
                const short* const* inNear = &p;

                short* const q = frame_out_s;
                short* const* outframe = &q;

                uint8_t saturationWarning = 0;

                //3
                if (0 != WebRtcAgc_Process(agcInst, inNear, 1, samples, outframe, micLevelIn, &micLevelOut, 0, &saturationWarning))
                {
                    std::cout << "WebRtcAgc_Process error" << std::endl;
                    break;
                }
                micLevelIn = micLevelOut;

                fwrite(frame_out_s, sizeof(short), samples, fpOut);
            }
            else
            {
                break;
            }
        }

        spanTick = GetTickCount() - spanTick;
        std::cout << "agc spanTick:" << spanTick << std::endl;
    } while (0);

    //clean
    if (agcInst) WebRtcAgc_Free(agcInst);
    if (fpIn) fclose(fpIn);
    if (fpOut) fclose(fpOut);

    delete[]frame_in_c;
    delete[]frame_in_s;
    delete[]frame_out_s;
}
```

#### 测试文件
自动增益测试音频 [1C_16bit_32K_lhydd_ns.pcm](http://ooid5jqhw.bkt.clouddn.com/1C_16bit_32K_lhydd_ns.pcm)