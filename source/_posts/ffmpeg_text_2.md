---
title: ffmpeg利用filter渲染文字
date: 2017-05-17 11:50:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - ffmpeg
---

## 背景
ffmpeg渲染视频文字c++类。

## 头文件（VideoDrawText.h）

<!--more-->

``` cpp
#ifndef VideoDrawText_h__
#define VideoDrawText_h__

#include <memory>
#include <string>

struct AVCodec;
struct AVCodecContext;
struct AVPacket;
struct AVFrame;
struct SwsContext;
struct AVFilterGraph;
struct AVFilterContext;
struct AVFilterInOut;

class VideoDrawText
{
public:
	VideoDrawText();
	~VideoDrawText();

	AVFrame *drawText(AVFrame *frameIn, std::string text);

private:
	bool init(int w, int h);
	void uninit();

	std::string GetModulePath();

private:
	AVFrame *m_frame_out;
	unsigned char *m_frame_buffer_out;

	AVFilterGraph *m_filter_graph;
	AVFilterContext *m_buffersrc_ctx;
	AVFilterContext *m_buffersink_ctx;
	AVFilterInOut *m_outputs;
	AVFilterInOut *m_inputs;

	int m_width;
	int m_height;
	std::string m_lastText;
};

#endif // VideoDrawText_h__
```

## 源文件（VideoDrawText.cpp）

``` cpp
#include "VideoDrawText.h"
#include <windows.h>

extern "C"
{
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>
#include <libavdevice/avdevice.h>
#include <libavutil/common.h>
#include <libavutil/avstring.h>
#include <libavutil/bprint.h>
#include <libavutil/time.h>
#include <libavutil/timestamp.h>
#include <libavutil/pixdesc.h>
#include <libavutil/avassert.h>
#include <libavutil/intreadwrite.h>
#include <libavutil/avutil.h>
#include <libavutil/imgutils.h>
#include <libavutil/pixfmt.h>
#include <libavfilter/buffersink.h>
#include <libavfilter/avfilter.h>
#include <libavutil/eval.h>
#include <libavutil/parseutils.h>
#include <libavfilter/avfiltergraph.h>
#include <libavfilter/buffersink.h>
#include <libavfilter/buffersrc.h>
#include <libavutil/opt.h>
#include <libavutil/imgutils.h>
};

void my_logoutput(void* ptr, int level, const char* fmt, va_list vl) 
{
#ifdef _DEBUG
	FILE *fp = fopen("my_log.txt", "a+");
	if (fp) 
	{
		vfprintf(fp, fmt, vl);
		fflush(fp);
		fclose(fp);
	}
#endif
}

VideoDrawText::VideoDrawText()
	: m_frame_out(nullptr)
	, m_frame_buffer_out(nullptr)
	, m_filter_graph(nullptr)
	, m_buffersrc_ctx(nullptr)
	, m_buffersink_ctx(nullptr)
	, m_outputs(nullptr)
	, m_inputs(nullptr)
	, m_width(0)
	, m_height(0)
{
	avfilter_register_all();
	av_log_set_callback(my_logoutput);
}

VideoDrawText::~VideoDrawText()
{
	uninit();
}

AVFrame * VideoDrawText::drawText(AVFrame *frameIn, std::string text)
{
	if (nullptr == frameIn)
	{
		return nullptr;
	}

	//set path
	std::string appPath = GetModulePath();
	SetCurrentDirectoryA(appPath.c_str());

	if (m_width != frameIn->width || m_height != frameIn->height || m_lastText != text)
	{
		//init
		if (!init(frameIn->width, frameIn->height))
		{
			std::cout << "init faild";
			return nullptr;
		}

		m_width = frameIn->width;
		m_height = frameIn->height;

		//input
		int fontSize = frameIn->width / 40;
		char filter_descr[1024] = { 0 };// "drawtext=text='hello world':x=100:y=100:fontsize=100:fontfile=FreeSans.ttf";
		sprintf(filter_descr, "drawtext=text='%s':x=50:y=50:fontsize=%d:fontcolor=green:fontfile=FreeSans.ttf", text.c_str(), fontSize);

		int ret = avfilter_graph_parse_ptr(m_filter_graph, filter_descr, &m_inputs, &m_outputs, NULL);
		if (ret < 0)
		{
			std::cout << "avfilter_graph_parse_ptr faild:" << ret;
			return nullptr;
		}

		ret = avfilter_graph_config(m_filter_graph, NULL);
		if (ret < 0)
		{
			std::cout << "avfilter_graph_config faild:" << ret;
			return nullptr;
		}
	}

	//output
	av_frame_unref(m_frame_out);
	int ret = av_buffersrc_add_frame(m_buffersrc_ctx, frameIn);
	if (ret < 0)
	{
		std::cout << "av_buffersrc_add_frame faild:" << ret;
		return nullptr;
	}

	/* pull filtered pictures from the filtergraph */
	ret = av_buffersink_get_frame(m_buffersink_ctx, m_frame_out);
	if (ret < 0)
	{
		std::cout << "av_buffersink_get_frame faild:" << ret;
		return nullptr;
	}

	m_lastText = text;
	return m_frame_out;
}

bool VideoDrawText::init(int width, int height)
{
	uninit();

	m_filter_graph = avfilter_graph_alloc();

	/* buffer video source: the decoded frames from the decoder will be inserted here. */
	char args[512] = { 0 };
	snprintf(args, sizeof(args), 
		"video_size=%dx%d:pix_fmt=%d:time_base=%d/%d:pixel_aspect=%d/%d",
		width, height, AV_PIX_FMT_YUV420P,
		1, 25, 1, 1);

	AVFilter *buffersrc = avfilter_get_by_name("buffer");

	int ret = avfilter_graph_create_filter(&m_buffersrc_ctx, buffersrc, "in", args, NULL, m_filter_graph);
	if (ret < 0) 
	{
		std::cout << "avfilter_graph_create_filter faild:" << ret;
		return false;
	}

	/* buffer video sink: to terminate the filter chain. */
	AVFilter *buffersink = avfilter_get_by_name("buffersink");
	enum AVPixelFormat pix_fmts[] = { AV_PIX_FMT_YUV420P, AV_PIX_FMT_NONE };
	AVBufferSinkParams *buffersink_params = av_buffersink_params_alloc();
	buffersink_params->pixel_fmts = pix_fmts;

	ret = avfilter_graph_create_filter(&m_buffersink_ctx, buffersink, "out", NULL, buffersink_params, m_filter_graph);
	av_free(buffersink_params);
	if (ret < 0)
	{
		std::cout << "avfilter_graph_create_filter faild:" << ret;
		return false;
	}

	/* Endpoints for the filter graph. */
	m_outputs = avfilter_inout_alloc();
	m_inputs = avfilter_inout_alloc();

	m_outputs->name = av_strdup("in");
	m_outputs->filter_ctx = m_buffersrc_ctx;
	m_outputs->pad_idx = 0;
	m_outputs->next = NULL;

	m_inputs->name = av_strdup("out");
	m_inputs->filter_ctx = m_buffersink_ctx;
	m_inputs->pad_idx = 0;
	m_inputs->next = NULL;

	m_frame_out = av_frame_alloc();
	m_frame_buffer_out = (unsigned char *)av_malloc(av_image_get_buffer_size(AV_PIX_FMT_YUV420P, width, height, 1));
	av_image_fill_arrays(m_frame_out->data, m_frame_out->linesize, m_frame_buffer_out, AV_PIX_FMT_YUV420P, width, height, 1);

	return true;
}

void VideoDrawText::uninit()
{
	if (m_frame_buffer_out)
	{
		av_free(m_frame_buffer_out);
		m_frame_buffer_out = nullptr;
	}

	if (m_frame_out)
	{
		av_frame_free(&m_frame_out);
		m_frame_out = nullptr;
	}

	if (m_inputs)
	{
		avfilter_inout_free(&m_inputs);
		m_inputs = nullptr;
	}

	if (m_outputs)
	{
		avfilter_inout_free(&m_outputs);
		m_outputs = nullptr;
	}

	
	if (m_filter_graph)
	{
		avfilter_graph_free(&m_filter_graph);
		m_filter_graph = nullptr;
	}
}

std::string VideoDrawText::GetModulePath()
{
	std::string _appPath;
#ifdef _WIN32
	char szAppPath[MAX_PATH] = { 0 };
	GetModuleFileNameA(NULL, szAppPath, MAX_PATH);
	(strrchr(szAppPath, '\\'))[0] = 0;		//结尾无斜杠
											//(strrchr(szAppPath, '\\'))[1] = 0;	// 结尾有斜杠
	_appPath = szAppPath;
#else
	char szAppPath[1024] = { 0 };
	int rslt = readlink("/proc/self/exe", szAppPath, 1023);
	if (rslt < 0 || (rslt >= 1023))
	{
		_appPath = "";
	}
	else
	{
		szAppPath[rslt] = '\0';
		for (int i = rslt; i >= 0; i--)
		{
			if (szAppPath[i] == '/')
			{
				szAppPath[i] = '\0';

				_appPath = szAppPath;
				break;
			}
		}
	}
#endif

	return _appPath;
}

``` 