---
title: Singleton & memory order
date: 2017-09-13 09:09:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 背景
先看一段c++单例常见代码：
``` cpp
Singleton* Singleton::instance() 
{
    if (pInstance == nullptr)       //第一次检查
    {     
        Lock lock;
        if (pInstance == nullptr)   //第二次检查
        {    
            pInstance = new Singleton;
        }
    }

    return pInstance;
}
```

在上面的代码中，第一次检查并没有加锁，就避免了每次调用instance()时都要加锁的问题。貌似这个方法很完美了吧，逻辑上无懈可击。

<!--more-->

其实上述设计方式使用到了双重检查锁定模式（DCLP），下面介绍下什么是DCLP。

**双重检查锁定模式（DCLP）**：DCLP的关键之处在于我们观察到的这一现象：调用者在调用instance()时，pInstance在大部分时候都是非空的，因此没必要再次初始化。所以，DCLP在加锁之前先做了一次pInstance是否为空的检查。只有判断结果为真（即pInstance还未初始化），加锁操作才会进行，然后再次检查pInstance是否为空（这就是该模式被命名为双重检查的原因）。第二次检查是必不可少的，因为，正如我们之前的分析，在第一次检验pInstance和加锁之间，可能有另一个线程对pInstance进行初始化。

DCLP前提：`DCLP的执行过程中必须确保机器指令是按一个可接受的顺序执行的。`

## DCLP与指令执行顺序
思考一下初始化pInstance的这行代码:
``` cpp
pInstance = new Singleton;
```
这条语句实际做了三件事情：

+ 步骤1：为Singleton对象分配一片内存
+ 步骤2：构造一个Singleton对象，存入已分配的内存区
+ 步骤3：将pInstance指向这片内存区

这里至关重要的一点是：我们发现编译器并不会被强制按照以上顺序执行！实际上，编译器有时会交换步骤2和步骤3的执行顺序。

优化编译器会仔细地分析并重新排序你的代码，使得程序执行时，在可见行为的限制下，同一时间能做尽可能多的事情。在串行代码中发现并利用这种并行性是重新排列代码并引入乱序执行最重要的原因，但并不是唯一原因，以下几个原因也可能使编译器（和链接器）将指令重新排序：

1. 避免寄存器数据溢出；
2. 保持指令流水线连续；
3. 公共子表达式消除；
4. 降低生成的可执行文件的大小；

让我们先专注于如果编译这么做了，会发生些什么。

请看下面这段代码。我们将pInstance初始化的那行代码分解成我们上文提及的三个步骤来完成，把步骤1（内存分配）和步骤3（指针赋值）写成一条语句，接着写步骤2（构造Singleton对象）。正常人当然不会这么写代码，可是编译器却有可能将我们上文写出的DCLP源码生成出以下形式的等价代码。

``` cpp
Singleton* Singleton::instance()
{
    if (pInstance == 0) 
    {
        Lock lock;

        if (pInstance == 0)
        {
            pInstance =                         //步骤3
            operator new(sizeof(Singleton));    //步骤1
            new (pInstance) Singleton;          //步骤2
        }
    }
    
    return pInstance;
}
```

根据上述转化后的等价代码，我们来考虑以下场景：
+ 线程A进入instance()，检查出pInstance为空，请求加锁，而后执行由步骤1和步骤3组成的语句。之后线程A被挂起。此时，pInstance已为非空指针，但pInstance指向的内存里的Singleton对象还未被构造出来。
+ 线程B进入instance(), 检查出pInstance非空，直接将pInstance返回(return)给调用者。之后，调用者使用该返回指针去访问Singleton对象————显然这个Singleton对象实际上还未被构造出来呢！

只有步骤1和步骤2在步骤3之前执行，DCLP才有效。

因为c++编译器在编译过程中会对代码进行优化，所以实际的代码执行顺序可能被打乱，另外因为CPU有一级二级缓存(cache)，CPU的计算结果并不是及时更新到内存的，所以在多线程环境，不同线程间共享内存数据存在可见性问题，从而导致使用DCLP也存在风险。

## 内存栅栏技术
我们知道的双重检查锁定模式存在风险，那么有没有办法改进呢？ 办法是有，这就是内存栅栏技术(memory fence),也称内存栅障(memory barrier) 。

内存栅栏的作用在于保证内存操作的相对顺序， 但并不保证内存操作的严格时序， 确保第一个线程更新的数据对其他线程可见。 一个 memory fence之前的内存访问操作必定先于其之后的完成。

以下是使用内存栅栏技术来实现DCLP的伪代码
``` cpp
Singleton* Singleton::instance() 
{
    Singleton* tmp = m_instance;
    ...                     
    
    // 插入内存栅栏指令
    if (tmp == nullptr) 
    {
        Lock lock;

        tmp = m_instance;
        if (tmp == nullptr) 
        {
            tmp = new Singleton; // 语句1
            ...     

            // 插入内存栅栏指令,确保语句2执行时，tmp指向的对象已经完成初始化构造函数
            m_instance = tmp;//语句2            
        }
    }

    return tmp;
}
```

这里，我们可以看到：在m\_instance指针为NULL时，我们做了一次锁定，这个锁定确保创建该对象的线程对m\_instance 的操作对其他线程可见。在创建线程内部构造块中，m_instance被再一次检查，以确保该线程仅创建了一份对象副本。

## atomic
上节的代码使用内存栅栏锁定技术可以很方便地实现双重检查锁定。但是看着实在有点麻烦，在C++11中更好的实现方式是直接使用原子操作。

``` cpp
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::instance() 
{
    Singleton* tmp = m_instance.load(std::memory_order_acquire);

    if (tmp == nullptr) 
    {
        std::lock_guard<std::mutex> lock(m_mutex);

        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) 
        {
            tmp = new Singleton;
            m_instance.store(tmp, std::memory_order_release);
        }
    }

    return tmp;
}
```

如果你对memory\_order的概念还是不太清楚，那么就使用C++顺序一致的原子操作，所有std::atomic的操作如果不带参数默认都是std::memory\_order\_seq\_cst,即顺序的原子操作（sequentially consistent）,简称SC,使用（SC）原子操作库，整个函数执行指令都将保证顺序执行,这是一种最保守的内存模型策略。 

下面的代码就是使用SC原子操作实现双重检查锁定：

``` cpp
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance()
 {
    Singleton* tmp = m_instance.load();

    if (tmp == nullptr) 
    {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load();

        if (tmp == nullptr) 
        {
            tmp = new Singleton;
            m_instance.store(tmp);
        }
    }

    return tmp;
}
```

## call_once
call_one保证函数fn只被执行一次，如果有多个线程同时执行函数fn调用，则只有一个活动线程(active call)会执行函数，其他的线程在这个线程执行返回之前会处于”passive execution”(被动执行状态)—不会直接返回，直到活动线程对fn调用结束才返回。对于所有调用函数fn的并发线程的数据可见性都是同步的(一致的)。 

如果活动线程在执行fn时抛出异常，则会从处于”passive execution”状态的线程中挑一个线程成为活动线程继续执行fn,依此类推。 

一旦活动线程返回,所有”passive execution”状态的线程也返回,不会成为活动线程。

由上面的说明，我们可以确信call_once完全满足对多线程状态下对数据可见性的要求。 
所以利用call_once再结合lambda表达式,前面几节那么多复杂代码，在这里千言万语凝聚为一句话：

``` cpp
Singleton* Singleton::m_instance;
Singleton* Singleton::instance() 
{
    static std::once_flag oc;//用于call_once的局部静态变量
    std::call_once(oc, [&] {  m_instance = new Singleton();});
    return m_instance;
}
```

## 多线程与static
可以看出上面的代码相比较之前的示例代码来说已经相当的简洁了，但是在C++memory model中对static local variable，说道：The initialization of such a variable is defined to occur the first time control passes through its declaration; for multiple threads calling the function, this means there’s the potential for a race condition to define first.

因此，我们将会得到一份最简洁也是效率最高的单例模式的C++11实现：
(vc2015简单测试通过)

``` cpp
Singleton& Singleton::instance() 
{
    static Singleton instance;
    return instance;
}
```

## 日常应用中可能会踩的坑

线程A代码：
``` cpp
int flag = 0;
int params = 0;
params = 1;
flag = 1;
```

有线程B引用线程A中的数据：
``` cpp
if(1 == flag)
{
    //using params;
    ....
}
```
由于编译器会进行优化，不能确保params = 1的赋值操作在flag = 1之前进行。


## 参考
[当我们在谈论 memory order 的时候，我们在谈论什么](https://cloud.tencent.com/community/article/163162)

[C++和双重检查锁定模式(DCLP)的风险](http://blog.jobbole.com/86392/)

[c++11单实例(singleton)初始化的几种方法](http://blog.csdn.net/10km/article/details/49777749)

[漫谈C++11多线程内存模型](http://blog.csdn.net/cszhouwei/article/details/11730559)