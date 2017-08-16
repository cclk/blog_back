---
title: webrtc学习3-回声消除
date: 2017-08-16 09:32:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - webrtc
---

## 一、回声消除
从通讯回音产生的原因看，可以分为声学回音（Acoustic Echo）和线路回音（Line Echo），相应的回声消除技术就叫声学回声消除（Acoustic Echo Cancellation，AEC）和线路回声消除（Line Echo Cancellation, LEC）。
+ 声学回音是由于在免提或者会议应用中，扬声器的声音多次反馈到麦克风引起的（比较好理解）；
+ 线路回音是由于物理电子线路的二四线匹配耦合引起的（比较难理解）。

<!--more-->

## 二、产生原因
回音的产生主要有两种原因

### 由于空间声学反射产生的声学回音

如下图：

![Alt text](http://ooid5jqhw.bkt.clouddn.com/echo1.jpg)

图中的男子说话，语音信号（speech1）传到女士所在的房间，由于空间的反射，形成回音speech1(Echo)重新从麦克风输入，同时叠加了女士的语音信号（speech2）。此时男子将会听到女士的声音叠加了自己的声音，影响了正常的通话质量。此时在女士所在房间应用回音抵消模块，可以抵消掉男子的回音，让男子只听到女士的声音。

### 由于2-4线转换引入的线路回音

如下图：

![Alt text](http://ooid5jqhw.bkt.clouddn.com/echo2.jpg)

在ADSL Modem和交换机上都存在2-4线转换的电路，由于电路存在不匹配的问题，会有一部分的信号被反馈回来，形成了回音。如果在交换机侧不加回音抵消功能，打电话的人就会自己听到自己的声音。

## 三、AEC回声消除
看下面的AEC声学回声消除框图

![Alt text](http://ooid5jqhw.bkt.clouddn.com/echo3.jpg)

其中，我们可以得到两个信号：一个是蓝色和红色混合的信号1，也就是实际需要发送的speech和实际不需要的echo混合而成的语音流；另一个就是虚线的信号2，也就是原始的引起回音的语音。那大家会说，哦，原来回声消除这么简单，直接从混合信号1里面把把这个虚线的2减掉不就行了？请注意，拿到的这个虚线信号2和回音echo是有差异的，直接相减会使语音面目全非。我们把混合信号1叫做近端信号ne，虚线信号2叫做远端参考信号fe，如果没有fe这个信号，回声消除就是不可能完成的任务，就像“巧妇难为无米之炊”。

虽然参考信号fe和echo不完全一样，存在差异，但是二者是高度相关的，这也是echo称之为回音的原因。至少，回音的语义和参考信号是一样的，也还听得懂，但是如果你说一句，马上又听到自己的话回来一句，那是比较难受的。既然fe和echo高度相关，echo又是fe引起的，我们可以把echo表示为fe的数学函数：echo=F（fe）。函数F被称之为回音路径。在声学回声消除里面，函数F表示声音在墙壁，天花板等表面多次反射的物理过程；在线路回声消除里面，函数F表示电子线路的二四线匹配耦合过程。很显然，我们下面要做的工作就是求解函数F。得到函数F就可以从fe计算得到echo，然后从混合信号1里面减掉echo就实现了回声消除。
 
尽管回声消除是非常复杂的技术，但我们可以简单的描述这种处理方法：
+ 房间A的音频会议系统接收到房间B中的声音
+ 声音被采样，这一采样被称为回声消除参考
+ 随后声音被送到房间A的音箱和声学回声消除器中
+ 房间B的声音和房间A的声音一起被房间A的话筒拾取
+ 声音被送到声学回声消除器中，与原始的采样进行比较，移除房间B的声音

### 自适应滤波器（数学太渣，看不懂，有兴趣的翻参考材料，略）

## 四、webrtc aec
webrtc的回声抵消(aec、aecm)算法主要包括以下几个重要模块：
+ 回声时延估计 
+ NLMS(归一化最小均方自适应算法) 
+ NLP（非线性滤波） 
+ CNG(舒适噪声产生）

### 回声时延估计
![](http://ooid5jqhw.bkt.clouddn.com/echo4.jpg)

这张图很多东西可以无视，我们重点看T0，T1，T2三项。
+ T0代表着声音从扬声器传到麦克风的时间，这个时间可以忽略，因为一般来说话筒和扬声器之间距离不会太远，考虑到声音340米每秒的速度，这个时间都不会超过1毫秒。
+ T1代表远处传到你这来的声音，这个声音被传递到回声消除远端接口（WebRtcAec_BufferFarend）的到播放出来的时间。一般来说接收到的音频数据传入这个接口的时候也就是上层传入扬声器的时刻，所以可以理解成该声音防到播放队列中开始计时，到播放出来的时间。
+ T2代表一段声音被扬声器采集到，然后到被送到近端处理函数（WebRtcAec_Process）的时刻，由于声音被采集到马上会做回声消除处理，所以这个时间可以理解成麦克风采集到声音开始计时，然后到你的代码拿到音频PCM数据所用的时间。
+ delay=T0+T1+T2，其实也就是T1+T2。

一般来说，一个设备如果能找到合适的delay，那么这个设备再做回声消除处理就和降噪增益一样几乎没什么难度了。如iPhone的固定delay是60ms。

### NLMS（归一化最小均方自适应算法）
+ LMS/NLMS/AP/RLS等都是经典的自适应滤波算法，此处只对webrtc中使用的NLMS算法做简略介绍。
+ 设远端信号为x(n),近段信号为d(n),W(n),则误差信号e(n)=d(n)-w'(n)x(n)  (此处‘表示转秩），NLMS对滤波器的系数更新使用变步长方法，即步长u=u0/(gamma+x'(n) * x(n))。其中u0为更新步长因子，gamma是稳定因子，则滤波器系数更新方程为 W(n+1)=W(n)+u*e(n)*x(n);  NLMS比传统LMS算法复杂度略高，但收敛速度明显加快。LMS/NLMS性能差于AP和RLS算法。
+ webrtc使用了分段块频域自适应滤波(PBFDAF)算法，这也是自适应滤波器的常用算法。
+ 自适应滤波的更多资料可以参考simon haykin 的《自适应滤波器原理》。

### NLP（非线性滤波）
webrtc采用了维纳滤波器。此处只给出传递函数的表达式，设估计的语音信号的功率谱为Ps(w)，噪声信号的功率谱为Pn(w)，则滤波器的传递函数为H(w)=Ps(w)/(Ps(w)+Pn(w))。

### CNG(舒适噪声产生）
webrtc采用的舒适噪声生成器比较简单，首先生成在[0 ,1 ]上均匀分布的随机噪声矩阵，再用噪声的功率谱开方后去调制噪声的幅度。

### 应用场景
webrtc AEC算法是属于分段快频域自适应滤波算法，Partioned block frequeney domain adaPtive filter(PBFDAF)。具体可以参考Paez Borrallo J M and Otero M G

使用该AEC算法要注意两点：
+ 延时要小，因为算法默认滤波器长度是分为12块，每块64点，按照8000采样率，也就是12*8ms=96ms的数据，而且超过这个长度是处理不了的。
+ 延时抖动要小，因为算法是默认10块也计算一次参考数据的位置（即滤波器能量最大的那一块），所以如果抖动很大的话找参考数据时不准确的，这样回声就消除不掉了。

### Demo

#### C++ code
``` cpp
#include <webrtc/modules/audio_processing/aec/echo_cancellation.h>

using namespace webrtc;

#define NN 160

int webrtcAecTest1()
{
#define  NN 160
    char far_frame_c[NN * 2];
    char near_frame_c[NN * 2];

    short far_frame_s[NN];
    short near_frame_s[NN];
    short out_frame_s[NN];

    void *aecmInst = NULL;
    FILE *fp_far = fopen("speaker.pcm", "rb");
    FILE *fp_near = fopen("micin.pcm", "rb");
    FILE *fp_out = fopen("out1.pcm", "wb");

    float far_frame_f[NN];
    float near_frame_f[NN];
    float out_frame_f[NN];

    float;

    do
    {
        if (!fp_far || !fp_near || !fp_out)
        {
            printf("WebRtcAecTest open file err \n");
            break;
        }

        aecmInst = WebRtcAec_Create();
        WebRtcAec_Init(aecmInst, 8000, 8000);

        AecConfig config;
        config.nlpMode = kAecNlpConservative;
        WebRtcAec_set_config(aecmInst, config);

        while (1)
        {
            if (NN == fread(far_frame_c, sizeof(char) * 2, NN, fp_far))
            {
                //1
                for (int i = 0; i < NN; ++i)
                {
                    far_frame_s[i] = (far_frame_c[i * 2 + 1] << 8) | (far_frame_c[i * 2] & 0xFF);//两个char型拼成一个short
                    far_frame_f[i] = far_frame_s[i];//转float型接口需要
                }
                WebRtcAec_BufferFarend(aecmInst, far_frame_f, NN);//对参考声音(回声)的处理

                //2
                fread(near_frame_c, sizeof(char) * 2, NN, fp_near);
                for (int i = 0; i < NN; ++i)
                {
                    near_frame_s[i] = (near_frame_c[i * 2 + 1] << 8) | (near_frame_c[i * 2] & 0xFF);
                    near_frame_f[i] = near_frame_s[i];
                }

                float* const p = near_frame_f;
                const float* const* nearend = &p;

                float* const q = out_frame_f;
                float* const* out = &q;

                //3
                WebRtcAec_Process(aecmInst, nearend, 1, out, NN, 40, 0);//回声消除
                for (int i = 0; i < NN; ++i)
                {
                    out_frame_s[i] = out_frame_f[i];
                }
                fwrite(out_frame_s, sizeof(short), NN, fp_out);
            }
            else
            {
                break;
            }
        }
    } while (0);

    fclose(fp_far);
    fclose(fp_near);
    fclose(fp_out);
    WebRtcAec_Free(aecmInst);
    return 0;
}
```

#### 测试文件
麦克风输入音频，包括回音 [micin.pcm](http://ooid5jqhw.bkt.clouddn.com/speaker.pcm)

回音参考音频 [speaker.pcm](http://ooid5jqhw.bkt.clouddn.com/micin.pcm)

## 五、直播应用方案

根据直播应用场景，有两种可能需要回声消除的情况：
+ 场景A：主播端具有麦克风输入，并且音箱外放声音，需要过滤掉音响的回音；
+ 场景B：主播端同某些客户端在同一房间，并且客户端在音响外放；

### 场景A
可以利用webrtc的aec模块进行回声消除，在windows端，需要计算出音频输出到音响，麦克风采集到音频的时间间隔。
但实际应用中，一般主播端是无外放功能，回声消除的作用不是特别广泛。

### 场景B
很难计算每个客户端到采集端的delay时间，需要一些ntp之类的时间同步过程，相对复杂，技术难较高，效果也不会特别明显；实际应用过程中，主播端应该是在一个安静的环境中，应用范围低。

综上两种场景，结合直播的实际情况，在直播应用中加入回声消除，适用面窄，不适合加入回声消除方案。

## 六、参考
[解密回声消除技术之一（理论篇）](http://silversand.blog.51cto.com/820613/166095/)

[维基百科-回音消除](https://zh.wikipedia.org/wiki/%E5%9B%9E%E9%9F%B3%E6%B6%88%E9%99%A4)

[单独编译和使用webrtc音频回声消除模块(附完整源码+测试音频文件)](http://www.cnblogs.com/mod109/p/5827918.html)

