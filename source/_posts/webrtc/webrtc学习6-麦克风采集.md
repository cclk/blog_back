---
title: webrtc学习6-麦克风采集
date: 2017-09-21 14:57:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 一、背景
webrtc的本地音频的采集由AudioDeviceModule接口统一封装。

AudioDeviceModule是个大而全的接口，具体包括：枚举音频采集设备（Record）和播放设备（Playout）、设置当前的采集设备/播放设备、开始/停止音频的采集/播放、设置音频增益控制开关（AGC）等。

AudioTransport是个关键的对外接口，负责音频数据的传入（调用NeedMorePlayData方法，供Playout使用）和输出（调用RecordedDataIsAvailable方法，数据由Record采集操作产生）。

## 二、windows实现
61版本的webrtc麦克风采集在windows上的实现代码是在下面文件内：
``` cpp
webrtc\modules\audio_device\win\audio_device_core_win.h
webrtc\modules\audio_device\win\audio_device_core_win.cc
```
内部实现方式为**WASAPI**：WASAPI的全称是Windows Audio Session API（Windows音频会话API），是从Windows Vista之后引入的UAA（Universal Audio Architecture）音频架构所属的API。WASAPI在Windows Vista、Windows 7、Windows Server 2008 R2系统中所使用。WASAPI允许传输未经修改的比特流到音频设备，从而避开SRC（Sample Rate Conversion，取样率转换器）的干扰。对于Windows XP来说，与WASAPI类似的通道为ASIO。

只支持Vista SP1及以上版本。

<!--more-->

## 三、Native Demo
``` cpp
#include <webrtc/modules/audio_device/include/audio_device.h>

#include <iostream>
#include <time.h>

#ifdef _WIN32
#include <windows.h>
#pragma comment(lib, "winmm.lib")
#pragma comment(lib, "Msdmo.lib")
#pragma comment(lib, "Dmoguids.lib")
#pragma comment(lib, "wmcodecdspuuid.lib")
#endif

using namespace webrtc;

class AudioTransportAPI : public AudioTransport
{
public:
    AudioTransportAPI(const rtc::scoped_refptr<AudioDeviceModule>& audioDevice) {}
    ~AudioTransportAPI() override {}

    int32_t RecordedDataIsAvailable(const void* audioSamples,
                                    const size_t nSamples,
                                    const size_t nBytesPerSample,
                                    const size_t nChannels,
                                    const uint32_t sampleRate,
                                    const uint32_t totalDelay,
                                    const int32_t clockSkew,
                                    const uint32_t currentMicLevel,
                                    const bool keyPressed,
                                    uint32_t& newMicLevel) override
    {
        char temp[1024] = { 0 };
        sprintf(temp, "audio-%dx%dx%d", sampleRate, nBytesPerSample * 8 / nChannels, nChannels);
        static FILE* pf_ = fopen(temp, "wb+");

        if (nChannels == 1)
        {
            // mono
        }
        else if ((nChannels == 2) && (nBytesPerSample == 2))
        {
            // stereo but only using one channel
        }
        else if ((nChannels == 2) && (nBytesPerSample == 4))
        {
            fwrite(audioSamples, 1, nSamples * nBytesPerSample, pf_);
        }
        else
        {
            // stereo
        }

        return 0;
    }

    int32_t NeedMorePlayData(const size_t nSamples,
                             const size_t nBytesPerSample,
                             const size_t nChannels,
                             const uint32_t sampleRate,
                             void* audioSamples,
                             size_t& nSamplesOut,
                             int64_t* elapsed_time_ms,
                             int64_t* ntp_time_ms) override
    {
        return 0;
    }

    void PushCaptureData(int voe_channel,
                         const void* audio_data,
                         int bits_per_sample,
                         int sample_rate,
                         size_t number_of_channels,
                         size_t number_of_frames) override {}

    void PullRenderData(int bits_per_sample,
                        int sample_rate,
                        size_t number_of_channels,
                        size_t number_of_frames,
                        void* audio_data,
                        int64_t* elapsed_time_ms,
                        int64_t* ntp_time_ms) override {}
};

int TestWebrtcAudioCapture()
{
    // Windows:
    //      if (WEBRTC_WINDOWS_CORE_AUDIO_BUILD)
    //          user can select only the default (Core)
    rtc::scoped_refptr<AudioDeviceModule> audio = webrtc::AudioDeviceModule::Create(0, AudioDeviceModule::kPlatformDefaultAudio);
    if (!audio)
    {
        std::cout << "Create device faild:" << std::endl;
        return 0;
    }

    int32_t ret = audio->Init();
    if (0 != ret)
    {
        std::cout << "audio->Init faild:" << ret << std::endl;
        return 0;
    }

    int num = audio->RecordingDevices();
    if (0 == num)
    {
        std::cout << "Audio input devices is 0" << std::endl;
        return 0;
    }

    std::cout << "Audio input devices:" << num << std::endl;
    for (int i = 0; i < num; i++)
    {
        char name[webrtc::kAdmMaxDeviceNameSize] = { 0 };
        char guid[webrtc::kAdmMaxGuidSize] = { 0 };

        int ret = audio->RecordingDeviceName(i, name, guid);
        if (ret != -1)
        {
            std::cout << "Index:" << i << " | name:" << name << " | guid:" << guid << std::endl;
        }
    }

    ret = audio->SetRecordingDevice(0);
    if (0 != ret)
    {
        std::cout << "audio->SetRecordingDevice faild:" << ret << std::endl;
        return 0;
    }

    ret = audio->SetAGC(true);
    if (0 != ret)
    {
        std::cout << "audio->SetAGC faild:" << ret << std::endl;
        return 0;
    }

    ret = audio->InitRecording();
    if (0 != ret)
    {
        std::cout << "audio->InitRecording faild:" << ret << std::endl;
        return 0;
    }

    AudioTransportAPI callback(audio);
    ret = audio->RegisterAudioCallback(&callback);
    if (0 != ret)
    {
        std::cout << "audio->RegisterAudioCallback faild:" << ret << std::endl;
        return 0;
    }

    ret = audio->StartRecording();
    if (0 != ret)
    {
        std::cout << "audio->StartRecording faild:" << ret << std::endl;
        return 0;
    }

    //wait
    time_t tBegin = time(nullptr);
    while (true)
    {
        time_t tEnd = time(nullptr);
        if ((tEnd - tBegin) > 10)
        {
            break;
        }

        Sleep(10);
    }

    //clean
    audio->StopRecording();//StartRecording
    audio->Terminate();//Init
    return 0;
}

int main()
{
    TestWebrtcAudioCapture();
    return 0;
}
```
