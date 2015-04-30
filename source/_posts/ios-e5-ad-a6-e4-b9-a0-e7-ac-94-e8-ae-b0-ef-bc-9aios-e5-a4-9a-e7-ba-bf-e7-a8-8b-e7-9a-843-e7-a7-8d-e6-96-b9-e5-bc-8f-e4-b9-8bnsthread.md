title: "iOS学习笔记：iOS多线程的3种方式之NSThread"
date: 2015-03-06 22:03:48
tags: iOS学习笔记
id: 43
categories: iOS学习笔记
---

众所周知：iOS的多线程的创建方式有3种：[NSThread](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSThread_Class/), [NSOperation](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/) 和[GCD(Grand_Central_Dispatch)](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/)。

##为什么苹果要出3种多线程呢？
答案是创建多线种的需求千变万化，不是所有的方式都能解决需求，所以三者相互共存着。今天我就先来分析NSThread的优缺点。  

##我的见解
通过阅读苹果的官方文档和参考了同行中的大牛的理解得出一下结论：

优点：

* NSThread 是轻量级的（大家公认的，Apple最早的多线程技术）；  
* 可以管理生命周期，NSThread 是一个对象，必然可以通过代码来管理它的生命周期（ps: 其实很多人认为这是它的缺点，使用起来复杂了需求，但是如果你有相应的需求的时候你会发现这才是你要的最好的多线程方式，看问题是需要多面的，在某些场景下优点与缺点会互换）。  

缺点：

* NSThread需要程序员自己处理其生命周期，数据同步问题（数据锁等），势必会影响了开发效率（ps:我以上的结论是站在不同的角度去说）;
* 线程同步以及数据加锁这些使用会影响性能的开销。  

##具体的使用我这边就不多讲了，推荐别人写的文章好了：
###[ iOS多线程编程之NSThread的使用](http://blog.csdn.net/totogo2010/article/details/8010231)

