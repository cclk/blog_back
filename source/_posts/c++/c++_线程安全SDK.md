---
title: c++开发线程安全的SDK
date: 2017-11-14 14:37:32
categories:
reward: false
tags:
     - blog
     - c++
     - sdk
---

## 背景
平时在封装SDK接口给上层应用调用时，理论上我们希望该SDK是可以在多线程环境下运行的，那样就可以避免上层应用多线程乱入的问题。

但如何设计一个多线程安全的SDK呢，想到的常规做法可能是加锁。使用加锁的方式，简单的接口可能比较容易实现，但如果是复杂的类导出，用锁就比较麻烦了，因为要考虑到每个接口的性能，又要考虑接口递归可能导致死锁，并且锁的维护比较麻烦。

之前在学习webrtc时，发现有一些很奇怪宏，然后就查了一下这些宏的作用。原来webrtc是在接口部分通过一个proxy代理线程来进行真正的调用， 它将来自任意线程的API调用， 转向SDK内部的工作线程，这样，在应用的角度看来， 这个SDK是线程安全并且支持线程乱入的， 但是内部又保持了简单有序的操作。

<!--more-->

## 实现
参考上篇文章<<基于c++11的线程池>>，我们可以应用一个代理线程来将sdk的调用转移到SDK内部的工作线程内。比如我们导出一个class sdk。

### 内部实现类
这个类原型一般为我们类内部实现的，会调用其他模块等。这里有2个方法newjob、deljob。
``` cpp
class sdkimpl
{
public:
    int newjob(int b)
    {
        deljob();

        a = new int;
        *a = b;

        return *a;
    }

    void deljob()
    {
        if (a)
        {
            delete a;
            a = nullptr;
        }
    }

private:
    int *a = nullptr;
};
```

### 导出类
这个类原型一般为导出类的真正接口，会在头文件内隐藏真正的实现。newjob和deljob是通常默认做法，safe_newjob和safe_deljob是线程安全的。
``` cpp
class __declspec(dllexport) sdk
{
public:
    sdk()
        : _pool(1)
    {
        _impl = new sdkimpl;
    }

    int newjob(int b)
    {
        return _impl->newjob(b);
    }

    void deljob()
    {
        _impl->deljob();
    }

    int safe_newjob(int b)
    {
        auto ret = _pool.commit(std::bind(&sdkimpl::newjob, _impl, b));
        return ret.get();
    }

    void safe_deljob()
    {
        auto ret = _pool.commit(std::bind(&sdkimpl::deljob, _impl));
        ret.get();
    }

private:
    sdkimpl *_impl;
    threadpool _pool;
};
```

### 测试
这里有2个线程不同在调用sdk的2个接口。
``` cpp
sdk *proxy = nullptr;

void newthread()
{
    while (true)
    {
        //proxy->newjob(100);
        proxy->safe_newjob(100);
    }
}

void delthread()
{
    while (true)
    {
        //proxy->deljob();
        proxy->safe_deljob();
    }
}

int main(int argc, char *argv[])
{
    proxy = new sdk;
    std::thread thnew(newthread);
    std::thread thdel(delthread);

    thnew.join();
    thdel.join();
    return 0;
}
```

## 参考
[WebRTC的PROXY-如何解决应用中的线程乱入](http://blog.csdn.net/volvet/article/details/52345025)
