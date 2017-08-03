---
title: c++双队列
date: 2017-04-17 10:41:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
平时在做一些数据处理中，会遇到一个读线程，一个写线程的情形，为了方便使用，可以简单封装一下线程安全的队列。

## 线程安全的队列

<!--more-->

``` cpp
#ifndef double_deque_h__
#define double_deque_h__

#include <deque>
#include <mutex>

template<typename val_type>
class utils_deque
{
public:
	utils_deque() {}
	~utils_deque() 
	{ 
		std::lock_guard<std::mutex> locker(m_mutex);

		while (!m_deque.empty())
		{
			val_type* var = m_deque.front();
			if (var)
			{
				delete var;
				var = NULL;
			}
			m_deque.pop_front();
		}
	}

	void push(val_type *var)
	{
		std::lock_guard<std::mutex> locker(m_mutex);

		m_deque.push_back(var);
	}

	val_type *pop()
	{
		std::lock_guard<std::mutex> locker(m_mutex);

		if (m_deque.empty())
		{
			return nullptr;
		}
		else
		{
			val_type* var = m_deque.front();
			m_deque.pop_front();
			return var;
		}
	}

	size_t size()
	{
		return m_deque.size();
	}

private:
	std::mutex m_mutex;
	std::deque<val_type* > m_deque;
};
#endif // double_deque_h__

```

为了方便自动释放空间，设计只支持传入指针对象。

## 双队列
但平时实际运用中，可能的需求是一个队列用于读，一个队列用于写；读队列用完之后会放入写队列；为了简化外部使用和自动管理，可以使用模版双队列。

``` cpp
#ifndef double_deque_h__
#define double_deque_h__

#include <deque>
#include <mutex>

template<typename val_type>
class utils_deque_rw
{
public:
	utils_deque_rw(utils_deque<val_type> *read, utils_deque<val_type> *write)
		: m_pop(read)
		, m_push(write)
		, m_var(NULL)
	{
		if (m_pop)
		{
			m_var = m_pop->pop();
		}
	}

	~utils_deque_rw()
	{
		if (m_push && m_var)
		{
			m_push->push(m_var);
		}
	}

	val_type* get()
	{
		return m_var;
	}

	//获取双队列总大小
	int size()
	{
		if (m_pop && m_push)
		{
			if (m_var)
			{
				return m_pop->size() + m_push->size() + 1;
			}
			else
			{
				return m_pop->size() + m_push->size();
			}
		}
		else
		{
			return -1;
		}
	}

	//重载-> 方便可以直接访问对象数据
	val_type* operator->()
	{
		return m_var;
	}

	//重载A a;中if(a)
	typedef void(*unspecified_bool_type)();
	static void unspecified_bool_true() {}
	operator unspecified_bool_type() const
	{
		return m_var == NULL ? 0 : unspecified_bool_true;
	}

	//重载A a;中if(!a)
	bool operator!() const
	{
		return m_var == NULL;
	}

private:
	utils_deque<val_type> *m_pop;   //出队列,需要读取的数据队列
	utils_deque<val_type> *m_push;  //入队列,数据用完之后放入的空闲队列

	val_type* m_var;//获取到的对象数据
};

#endif // double_deque_h__
```

## 使用示例
### 定义
``` cpp
#define CACHE_SIZE 10

struct data
{

}

utils_deque<data> m_used;
utils_deque<data> m_idle;
```

### 初始化
``` cpp
for (size_t i = 0; i < CACHE_SIZE; ++i)
{
    data *var = new data();
    m_idle.push(var);
}
```

### 读写
``` cpp
utils_deque_rw<data> rw(&m_idle, &m_used);
if (!rw)
{
  //do something
}
else
{
  //do something
}
```