title: "iOS学习笔记：iOS多线程的3种方式之GCD"
date: 2015-03-07 10:12:24
tags: 
- iOS学习笔记
- iOS多线程
id: 57
categories: 
---

众所周知：iOS的多线程的创建方式有3种：[NSThread](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSThread_Class/), [NSOperation](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSOperation_class/) 和[GCD(Grand_Central_Dispatch)](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/)。

## 为什么苹果要出3种多线程呢？
答案是创建多线种的需求千变万化，不是所有的方式都能解决需求，所以三者相互共存着。今天我就先来分析GCD的优缺点。  
<!--more-->
## 我的见解
通过阅读苹果的官方文档和参考了同行中的大牛的理解得出一下结论：

### 优点：
* 使用block技术（闭包），使代码看上去十分简洁，使用简单（适合新手）；
* 能自动分配到空闲的处理器内核中，最大限度发挥多核心CPU的性能；
### 缺点：
* 不能管理线程的生命周期，不能满足某些需求例如图片上传的取消（AFNetworking中就是用NSOperation实现的）。
## 总结：
iOS的多线程的这3种方式会一直存在着来满足不同的用户开发需求，三者相辅相成。我们不能一味的说哪种多线程的好与不好，只要找到某个场景下最合适的方法就可以了。

具体的使用我这边就不多讲了，推荐别人写的文章好了：

### [iOS多线程编程之Grand Central Dispatch(GCD)介绍和使用](http://blog.csdn.net/totogo2010/article/details/8016129)

