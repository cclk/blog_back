---
title: c++模块日志设计
date: 2017-04-17 11:40:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
平时在导出静态、动态库时，库内部需要写日志，如果直接依赖基础日志库，会存在基础日志库更新后，所有依赖模块都需要更新的情况。

## 设计方法
设计简单的只提供头文件的日志库，回调给上层应用，因为接口功能会很单一，稳定后基本不会更新，减少了对log库的依赖项。

## 实现

### mlog_impl.hpp
<!--more-->

``` cpp
#ifndef mlog_impl_h__
#define mlog_impl_h__

#include <sstream>
#include <iostream>
#include <functional>

#define MLOG_NAME_SPACE_START	namespace mlog {
#define MLOG_NAME_SPACE_END		};


MLOG_NAME_SPACE_START

enum MLogSeverity
{
	MLOG_DEBUG = 0,
	MLOG_INFO = 1,
	MLOG_WARNING = 2,
	MLOG_ERROR = 3,
	MLOG_FATAL = 4
};

typedef std::function <void(const std::string& file, const int& line, const std::string& function, const int& severity, const std::string& context)> MLogCallBack;

template<typename T>
class GlobalVar{
public:
	static T VAR;
};

class LogMessage
{
public:
	LogMessage(const char* file, int line, const char* function, MLogSeverity severity, mlog::MLogCallBack func)
		: m_file(file)
		, m_line(line)
		, m_function(function)
		, m_severity(severity)
		, m_func(func)
	{

	}

	~LogMessage()
	{
		if (nullptr != m_func)
		{
			m_func(m_file, m_line, m_function, m_severity, m_stream.str());
		}
	}

	std::ostringstream &stream() { return m_stream; }

private:
	std::ostringstream m_stream;

	const char *m_file;
	int m_line;
	const char *m_function;
	MLogSeverity m_severity;
	mlog::MLogCallBack m_func;
};


MLOG_NAME_SPACE_END


#endif // mlog_impl_h__

```

### mlog.hpp

``` cpp
#ifndef mlog_h__
#define mlog_h__

#include "mlog_impl.hpp"

MLOG_NAME_SPACE_START
template<typename T> T GlobalVar<T>::VAR = nullptr;
static void SetMlogCallBack(MLogCallBack func) { GlobalVar<MLogCallBack>::VAR = func; }
MLOG_NAME_SPACE_END

#define LOG_DEBUG mlog::LogMessage(__FILE__, __LINE__, __FUNCTION__, mlog::MLOG_DEBUG, mlog::GlobalVar<mlog::MLogCallBack>::VAR).stream()
#define LOG_INFO  mlog::LogMessage(__FILE__, __LINE__, __FUNCTION__, mlog::MLOG_INFO,  mlog::GlobalVar<mlog::MLogCallBack>::VAR).stream()
#define LOG_WARN  mlog::LogMessage(__FILE__, __LINE__, __FUNCTION__, mlog::MLOG_WARNING, mlog::GlobalVar<mlog::MLogCallBack>::VAR).stream()
#define LOG_ERROR mlog::LogMessage(__FILE__, __LINE__, __FUNCTION__, mlog::MLOG_ERROR, mlog::GlobalVar<mlog::MLogCallBack>::VAR).stream()
#define LOG_FATAL mlog::LogMessage(__FILE__, __LINE__, __FUNCTION__, mlog::MLOG_FATAL, mlog::GlobalVar<mlog::MLogCallBack>::VAR).stream()


#endif // mlog_h__

```