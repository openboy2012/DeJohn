title: "iOS学习笔记：NSRunLoop的理解"
date: 2015-03-17 15:00:49
tags: 
- iOS学习笔记
- iOS消息循环
categories: 
---
## 背景
有没有好奇过iOS应用不是像C程序那样执行完main函数以后就退出了，还可以在点击按钮以后弹出一些交互UI，还能通过手势滑动UI(如UIScrollView、UITableView等)，这一切都是NSRunLoop在帮忙。
<!--more-->

## NSRunLoop官方定义
官方文档的定义是[NSRunLoop](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/uid/TP40003725)，其字面意思是“运行回路”，它是一个循环、可以处理事件，是用来管理线程中输入的资源。iOS应用能在启动以后不退出，就是因为它的存在。  
主线程中的NSRunLoop是默认开启的但是处于一种“等待”的状态，当有信息输入时NSRunLoop才会发生响应（消息分发，支持异步），所以主线程不会被卡线程。 

### NSRunLoop的落脚点
每个创建的多线程都会自己创建一个NSRunLoop，但是默认非主线程的RunLoop是没有运行的，需要为RunLoop添加至少一个事件源，然后手动去run它。

### NSRunLoop处理的事件
NSRunLoop能处理的事件有2种：一种是输入源，一种是定时源。  
输入源包括3种：performSelector源，基于端口（Mach port）的源，以及自定义的源；他们都是用来处理异步事件的；  
定时源即NSTimer，一般情况下是用来处理同步事件的。

### NSRunLoop的使用场景
下来是Run Loop的使用场景：  

* 使用port或是自定义的input source来和其他线程进行通信<AFNetworking中有使用>;
* 在线程（非主线程）中使用NSTimer
* 使用 performSelector...系列
``` objc
- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray *)modes;  
- (void)performSelector:(SEL)aSelector withObject:(id)anArgument afterDelay:(NSTimeInterval)delay;  
```
* 使用线程执行周期性工作


## 引用别人的一篇文章觉得不错：
### [- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate方法详解](http://blog.csdn.net/wjsxiaoweige/article/details/38318733)

