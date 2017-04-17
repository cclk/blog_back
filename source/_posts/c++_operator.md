---
title: c++判断类是否可用
date: 2017-04-17 09:59:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
在c++ 中，有时候我们需要一个自定义类型能够支持 if(obj) 和 if(!obj) 之类的语法，也就是说
``` cpp
A obj;
if(obj)
{
  //do something
}

if(!obj)
{
  //do something
}
```

这个需要在智能指针的实现中尤其明显，因为它可以保证与原生C++ 指针在用法上的一致性。明显的解决方法是重载 operator bool() 转换，但是这样问题太多，Effective C++ 里面有讨论。还有一个办法是重载 operator ! ，但是这样我们就不得不用 if(!!obj) 这样丑陋的语法来表达 if(obj) 。

<!--more-->

## 解决办法
参考boost
``` cpp
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
```

## 解读
``` cpp
typedef void(*unspecified_bool_type)()
```
这一句申明了一个指向本类成员变量的指针的类型，声明一个指向类成员变量的指针类型的格式：类型 类名::*指针类型，不明白可以参考下指向类成员函数的指针。通过这句代码得到的信息有：
+ unspecified_bool_type是个类型，而不是变量，由typedef得知。
+ unspecified_bool_type是该类的成员变量的类型，该成员变量的类型是void *。

``` cpp
operator unspecified_bool_type() const
{
  return m_var == NULL ? 0 : unspecified_bool_true;
}
```
operator关键字除了操作符重载外，还有另外一种用法，那就是隐式类型转换，格式如下:
``` cpp
operator type() {
}
```
执行if (ptr)时候便会执行operator unspecified_bool_type() const，转嫁成判断unspecified_bool_type类型的指针是否为空。