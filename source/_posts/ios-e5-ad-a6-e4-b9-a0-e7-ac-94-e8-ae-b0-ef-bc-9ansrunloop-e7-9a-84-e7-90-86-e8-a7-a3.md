title: "iOS学习笔记：NSRunLoop的理解"
date: 2015-03-17 15:00:49
tags: 
- iOS学习笔记
- iOS开发
id: 41
categories: 
- iOS学习笔记 
- iOS开发
---

官方文档的定义是[NSRunLoop](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/uid/TP40003725)是用来管理线程中输入的资源，它是一个循环、可以处理事件。

每个创建的多线程都会自己创建一个NSRunLoop。

主线程中的NSRunLoop是默认开启的但是处于一种“等待”的状态（而不像一些命令行程序一样运行一次就结束了），当有信息输入时NSRunLoop才会发生响应，所以主线程不会被卡线程。

在多线程中使用就必须手动管理了NSRunLoop了。

下来是Run Loop的使用场合：
* 使用port或是自定义的input source来和其他线程进行通信<AFNetworking中有使用>;
* 在线程（非主线程）中使用timer
* 使用 performSelector...系列
```
- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray *)modes;  
- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay;  
```
* 使用线程执行周期性工作
##引用别人的一篇文章觉得不错：

###[ - (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate 方法 详解 ](http://blog.csdn.net/wjsxiaoweige/article/details/38318733)

