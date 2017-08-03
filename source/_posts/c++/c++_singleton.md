---
title: c++通用单例模版类
date: 2017-04-17 10:57:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
c++平时在开发过程中，需要用到单例模式比较多，如果每个都需要去实现，比较麻烦，可实现一个通用的单例模版类。

## 实现

<!--more-->

``` cpp
#ifndef cc_singleton_h__
#define cc_singleton_h__

#include <mutex>
#include <memory>

namespace utils
{

template <typename T>
class singleton
{
public:
	// 创建单例实例
	template<typename ...Args>
	static std::shared_ptr<T> initial(Args&& ...args)
	{
		std::call_once(m_flag, [&] {m_instance = std::make_shared<T>(std::forward<Args>(args)...); });
		return m_instance;
	}

	// 获取单例
	static std::shared_ptr<T> get()
	{
		return m_instance;
	}

private:
	singleton() = default;
	~singleton() = default;
	singleton(const singleton &) = delete;
	singleton(singleton &&) = delete;
	singleton &operator=(const singleton &) = delete;
	singleton &operator=(singleton &&) = delete;

private:
	static std::shared_ptr<T> m_instance;
	static std::once_flag m_flag;
};

template<typename T> std::once_flag singleton<T>::m_flag;
template<typename T> std::shared_ptr<T> singleton<T>::m_instance;

}

#endif // cc_singleton_h__

```

## 使用示例
``` cpp
#include <utils/singleton.hpp>
#include <iostream>

class SingleTest
{
public:
	SingleTest(int x, double y) :m_iX(x), m_dY(y){}
	double sum(){ return m_iX + m_dY; }

private:
	int m_iX;
	double m_dY;
};

class SingleTestB
{
public:
	SingleTestB() :m_iX(0), m_dY(5.0){}
	double sum(){ return m_iX + m_dY; }

private:
	int m_iX;
	double m_dY;
};

int main(int argc, char* argv[])
{
	int a = 1;
	double b = 3.14;
	utils::singleton<SingleTest>::initial(a, b);
	utils::singleton<SingleTest>::initial(2, 3.14);
	std::cout << utils::singleton<SingleTest>::get()->sum() << std::endl;

	utils::singleton<SingleTestB>::initial();
	std::cout << utils::singleton<SingleTestB>::get()->sum() << std::endl;

	return 0;
}
```