---
title: c++线程安全的map
date: 2017-04-17 10:54:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
std::map不是线程安全的，平时在多线程处理时，需要加锁，比较麻烦，可以简单封装，使之支持多线程安全。

## 实现
需要手动释放资源，建议传入智能指针或者对象，这样就不需要关心资源释放问题。

<!--more-->

``` cpp
#ifndef UTILS_MAP_HPP__
#define UTILS_MAP_HPP__

#include <map>
#include <memory>
#include <mutex>

namespace utils
{

//thread safe map, need to free data youself
template<typename TKey, typename TValue>
class map
{
public:
	map() 
	{
	}

	virtual ~map() 
	{ 
		std::lock_guard<std::mutex> locker(m_mutexMap);
		m_map.clear(); 
	}

	bool insert(const TKey &key, const TValue &value, bool cover = false)
	{
		std::lock_guard<std::mutex> locker(m_mutexMap);

		auto find = m_map.find(key);
		if (find != m_map.end() && cover)
		{
			m_map.erase(find);
		}

		auto result = m_map.insert(std::pair<TKey, TValue>(key, value));
		return result.second;
	}

	void remove(const TKey &key)
	{
		std::lock_guard<std::mutex> locker(m_mutexMap);

		auto find = m_map.find(key);
		if (find != m_map.end())
		{
			m_map.erase(find);
		}
	}

	bool lookup(const TKey &key, TValue &value)
	{
		std::lock_guard<std::mutex> locker(m_mutexMap);

		auto find = m_map.find(key);
		if (find != m_map.end())
		{
			value = (*find).second;
			return true;
		}
		else
		{
			return false;
		}
	}

	int size()
	{
		std::lock_guard<std::mutex> locker(m_mutexMap);
		return m_map.size();
	}

public:
	std::mutex m_mutexMap;
	std::map<TKey, TValue> m_map;
};

}

#endif // UTILS_MAP_HPP__

```

## 使用示例
``` cpp
#include <utils/map.hpp>
#include <iostream>

class TestMapClass
{
public:
	TestMapClass(int i)
	{
		a = i;
	}

	~TestMapClass()
	{

	}

private:
	int a;
};


int main(int argc, char* argv[])
{
	//point
	{
		TestMapClass *p1 = new TestMapClass(1);
		TestMapClass *p2 = new TestMapClass(2);

		utils::map<std::string, TestMapClass*> MapTest;
		bool ret = MapTest.insert("1", p1);
		ret = MapTest.insert("1", p2);
		ret = MapTest.insert("2", p2);

		TestMapClass *lRet = NULL;
		MapTest.remove("2");
		ret = MapTest.lookup("2", lRet);
		ret = MapTest.lookup("1", lRet);
	}

	//ptr
	{
		std::shared_ptr<TestMapClass> p1(new TestMapClass(1));
		std::shared_ptr<TestMapClass> p2(new TestMapClass(2));

		utils::map<std::string, std::shared_ptr<TestMapClass> > MapTest;
		bool ret = MapTest.insert("1", p1);
		ret = MapTest.insert("1", p2);
		ret = MapTest.insert("2", p2);

		std::shared_ptr<TestMapClass> lRet;
		MapTest.remove("2");
		ret = MapTest.lookup("2", lRet);
		ret = MapTest.lookup("1", lRet);
	}


	return 0;
}
```