---
title: ffmpeg渲染文字字体路径问题
date: 2017-05-17 11:40:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - ffmpeg
---

## 背景
windows下想在视频上写字， 使用Drawtext滤镜，但路径是个问题：

``` cpp
drawtext="fontfile=/usr/share/fonts/truetype/freefont/FreeSerif.ttf: text='Test Text'"
```

<!--more-->

+ 以上是官网的使用方法， 参数间用冒号分隔， 但是在windows下fontfile在C:\WINDOWS\FONTS\目录下， 导致使用时总是cannot load font “C":impossible to find a matching font"。
+ 原因是fontfile使用的路径为linux风格。不适用于windows，windows中有冒号且使用反斜杠。
+ 查看ffmpeg源代码，avfilter_graph_parse_ptr->parse_filter->create_filter->avfilter_init_str，参数在windows下使用时， 到冒号就自动截断了， 所以fontfile总是加载失败。
+ 目前的解决方法是拷贝字体文件到执行文件目录下，直接使用当前文件解决，中文的话需要字体支持；最好在调用前，设置工作目录为当前目录。
