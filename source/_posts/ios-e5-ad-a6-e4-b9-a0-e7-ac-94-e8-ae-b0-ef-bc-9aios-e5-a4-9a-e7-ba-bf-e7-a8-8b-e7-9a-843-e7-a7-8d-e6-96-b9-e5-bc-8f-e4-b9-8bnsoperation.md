title: "iOS学习笔记：iOS多线程的3种方式之NSOperation"
date: 2015-03-06 22:42:31
tags:
id: 50
categories: iOS学习笔记
---

众所周知：iOS的多线程的创建方式有3种：[NSThread](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSThread_Class/), [NSOperation](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/) 和[GCD(Grand_Central_Dispatch)](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/)。

##为什么苹果要出3种多线程呢？
答案是创建多线种的需求千变万化，不是所有的方式都能解决需求，所以三者相互共存着。今天我就先来分析NSOperation的优缺点。  

##我的见解
通过阅读苹果的官方文档和参考了同行中的大牛的理解得出一下结论：

优点：
*  NSOperation不需要关心线程管理，数据同步的事情，可以把精力放在自己需要执行的操作上；
*  NSOperation可以监控多线程的运行状态，随时可以结束任务；
*  NSOperation支持KVO；
缺点：

* 比GCD更高级的抽象，造成了性能上的劣势；
* 对于新人来说，使用起来比GCD相对复杂；

##具体的使用我这边就不多讲了，推荐别人写的文章好了:
###[iOS多线程编程之NSOperation和NSOperationQueue的使用 ](http://blog.csdn.net/totogo2010/article/details/8013316)


