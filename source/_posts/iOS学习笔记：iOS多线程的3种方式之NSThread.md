title: "iOS学习笔记：iOS多线程的3种方式之NSThread"
date: 2015-03-06 22:03:48
tags: 
- iOS学习笔记
- iOS多线程
categories: 
---

众所周知：iOS的多线程的创建方式有3种：[NSThread](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSThread_Class/), [NSOperation](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/) 和[GCD(Grand_Central_Dispatch)](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/)。

## 为什么苹果要出3种多线程呢？
答案是创建多线种的需求千变万化，不是所有的方式都能解决需求，所以三者相互共存着。今天我就先来分析NSThread的优缺点。  
<!--more-->

## 我的见解
通过阅读苹果的官方文档和参考了同行中的大牛的理解得出以下结论：

优点：

* NSThread 是轻量级的（大家公认的，Apple最早的多线程技术）；  
* 可以管理生命周期，NSThread 是一个对象，必然可以通过相应的方法来管理它的生命周期，通过init方法创建线程实例，start方法让线程真正跑起来，cancel方法让线程取消，每个方法都能让我们合理操作。（ps：其实很多人认为这是它的缺点，使用起来确实复杂了点，但是当你有相应的需求时你会发现这才是你要的最合适的多线程方式，所以看问题是要多角度来分析的，在某些场景下优点与缺点会互换）。  

缺点：  

* NSThread需要程序员自己处理其生命周期，数据同步问题（数据锁等），势必会影响了开发效率（ps：我以上的结论是站在不同使用角度的看法）;
* 线程同步以及数据加锁这些操作会影响到线程运行时的性能。  

## 具体的使用我这边就不多讲了，推荐别人写的文章好了：
### [iOS多线程编程之NSThread的使用](http://blog.csdn.net/totogo2010/article/details/8010231)

