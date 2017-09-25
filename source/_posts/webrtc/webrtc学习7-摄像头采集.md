---
title: webrtc学习7-摄像头采集
date: 2017-09-25 14:44:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 背景
webrtc的video\_capture模块，为我们在不同端设备上采集视频提供了一个跨平台封装的视频采集功能，现webrtc的video\_capture模块支持android、ios、linux、mac和windows各操作平台下的视频采集。我们可以在不同端设备上开发视频直播的功能，也可以使用该模块进行视频采集。

windows 61版本的实现源码路径为webrtc\modules\video_capture\windows，其中提供了两套采集方案，分别为ds（direct show）和mf（media foundation），但mf方案实现是空的。

## Sample
样例路径如下：
``` cpp
webrtc\modules\video_capture\test\video_capture_unittest.cc
webrtc\modules\video_capture\test\video_capture_unittest.mm
```
其中后缀为mm对应mac系统。

<!--more-->

## windows使用注意事项
+ 需要在包含头文件前增加宏定义WEBRTC_WIN
+ 出现std::min等定义冲突，需要增加宏定义NOMINMAX

## Native Demo
``` cpp
#include <memory>
#include <iostream>
#include <time.h>

#define NOMINMAX
#define WEBRTC_WIN
#include <webrtc/modules/video_capture/video_capture.h>
#include <webrtc/modules/video_capture/video_capture_factory.h>

using namespace webrtc;

#pragma comment(lib, "winmm.lib")
#pragma comment(lib, "Strmiids.lib")//dshow

class TestVideoCaptureCallback : public rtc::VideoSinkInterface<webrtc::VideoFrame>
{
public:
    TestVideoCaptureCallback()
    {
    }

    ~TestVideoCaptureCallback()
    {
    }

    void OnFrame(const webrtc::VideoFrame& videoFrame) override
    {
        std::cout << "OnFrame w*h(" << videoFrame.width() << "*" << videoFrame.height() << ")" << std::endl;

        static bool first = false;
        static FILE* pf = nullptr;
        if (!first)
        {
            first = true;
            char path[1024] = { 0 };
            sprintf(path, "i420_%dx%d.yuv", videoFrame.width(), videoFrame.height());
            pf = fopen(path, "wb+");
        }

        rtc::scoped_refptr<webrtc::VideoFrameBuffer> buffer = videoFrame.video_frame_buffer();
        rtc::scoped_refptr<const I420BufferInterface> i420 = buffer->GetI420();

        if (pf)
        {
            int size = videoFrame.width() * videoFrame.height();
            fwrite(i420->DataY(), 1, size, pf);
            fwrite(i420->DataU(), 1, size / 4, pf);
            fwrite(i420->DataV(), 1, size / 4, pf);
        }
    }
};

int TestWebrtcVideoCapture()
{
    std::unique_ptr<VideoCaptureModule::DeviceInfo> device_info(VideoCaptureFactory::CreateDeviceInfo());
    uint32_t nums = device_info->NumberOfDevices();
    if (0 == nums)
    {
        std::cout << "video capture devices is 0." << std::endl;
        return 0;
    }

    //get first devices
    std::string my_unique_name;
    for (uint32_t i = 0; i < nums; ++i)
    {
        char device_name[256];
        char unique_name[256];
        uint32_t ret = device_info->GetDeviceName(i, device_name, 256, unique_name, 256);
        if (0 != ret)
        {
            std::cout << "GetDeviceName faild index:" << i << "|ret:" << ret;
            continue;
        }

        //remember
        std::cout << "==================>device_name:" << device_name << "|unique_name:" << unique_name << std::endl;
        if (my_unique_name.empty())
        {
            my_unique_name = unique_name;
        }

        //Capability
        int32_t cabs = device_info->NumberOfCapabilities(unique_name);
        for (int32_t j = 0; j < cabs; ++j)
        {
            VideoCaptureCapability capability;
            device_info->GetCapability(unique_name, j, capability);
            std::cout << "w*h(" << capability.width << "*" << capability.height << ")"
                << "|VideoType:" << static_cast<int>(capability.videoType)
                << "|maxFPS:" << capability.maxFPS << std::endl;
        }
    }

    if (my_unique_name.empty())
    {
        std::cout << "video capture devices unique name is error." << std::endl;
        return 0;
    }

    //start
    TestVideoCaptureCallback videoback;
    rtc::scoped_refptr<VideoCaptureModule> module = VideoCaptureFactory::Create(my_unique_name.c_str());
    module->RegisterCaptureDataCallback(&videoback);

    VideoCaptureCapability settings;
    settings.width = 1280;
    settings.height = 768;
    settings.maxFPS = 25;
    settings.videoType = webrtc::VideoType::kUnknown;
    uint32_t ret = module->StartCapture(settings);
    if (0 != ret)
    {
        std::cout << "StartCapture faild:" << ret;
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
    module->StopCapture();//StartCapture
    return 0;
}

int main()
{
    TestWebrtcVideoCapture();
    return 0;
}
```