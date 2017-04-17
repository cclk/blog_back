---
title: c++单一头文件全局变量导出
date: 2017-04-17 09:11:32
categories:
reward: false
tags:
     - blog
     - c++
---

## 场景
当我们需要提供一些简单的接口时，可能只需要提供单一头文件即可（不需要静、动态库）。但如果需要存在一些全局变量需要定义时则不好处理，本文提供一种方法，通过模版初始化的特性进行导出单一头文件的全局变量。下面介绍一些现有静、动态库定义全局变量的方法。

### extern
extern可置于变量或者函数前，以表示变量或者函数的定义在别的文件中，提示编译器遇到此变量或函数时，在其它模块中寻找其定义。另外，extern也可用来进行链接指定。

### 防重复定义

``` cpp
#ifndef XXX
#define XXX
#endif
```

这类条件编译是为了防止同一个.c文件包含同一个头文件多次。

要明白每一个.c文件最后都会编译生成对应的.obj文件的。所以两个.c文件对应的两个.obj文件都会有定义的那个全局变量的，链接的时候，链接器就会发现有定义了两个同名变量，于是就报multiple definition错误。
正确的做法是：是其中一个.c文件定义这个变量,在另外一个.c文件用

``` cpp
extern int g_var;
```

声明，这就可以在两个.c都使用这个变量了。

### 结论
由于我们的需求是导出单一的头文件，就无法使用extern方法；使用防重复定义的方法后，导出后被多个cpp引用的话也是会报编译错误，所以上述方法无法解决我们的问题。

## 模版
我们可以根据模版的特性，编译时才会生成确定对象，用以解决这个问题。

### 模板实例化
+ 编译器使用模板，通过更换模板参数来创建数据类型。这个过程就是模板实例化(Instantiation)。
+ 从模板类创建得到的类型称之为特例(specialization)。
+ 模板实例化取决于编译器能够找到可用代码来创建特例(称之为实例化要素，point of instantiation)。
+ 要创建特例，编译器不但要看到模板的声明，还要看到模板的定义。
+ 模板实例化过程是迟钝的，即只能用函数的定义来实现实例化。

### 实现
``` cpp
template<typename T>
class GlobalVar{
public:
	static T VAR;
};
```

我们可以在一个编译单元中使用如下代码为变量赋值:
``` cpp
template<typename T> T GlobalVar<T>::VAR = nullptr;
```

而在另一个编译单元中读取该变量：
``` cpp
std::cout << GlobalVar<XXX>::VAR << std::endl;
```

### 扩展
上述实现只是单一参数，如果需要多个参数，可以增加模版的参数个数来进行。
``` cpp
template<typename T, int n>
class GlobalVar{
public:
	static T VAR;
};

template<typename T, int n> T GlobalVar<T, 1>::VAR = 12345;
std::cout << GlobalVar<int, 1>::VAR << std::endl;
```

### 详细解释

在这里，我们利用了C++模板的一个特性——编译器保证具有相同模板参数的模板只实例化一次。也就是说，具有相同模板参数的GlobalVar中的静态变量var只被实例化一次。此时，以上方法定义的全局变量在视觉上，不大像全局变量。但是大家可以使用宏定义等技巧来把它变得更像一点。