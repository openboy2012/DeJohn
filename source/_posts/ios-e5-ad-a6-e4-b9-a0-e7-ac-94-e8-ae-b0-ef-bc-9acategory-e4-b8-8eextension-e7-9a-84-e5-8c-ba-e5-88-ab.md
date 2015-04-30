title: "iOS学习笔记：Category与Extension的区别"
date: 2015-03-11 17:18:49
tags:
id: 63
categories: iOS学习笔记
---

很早就知道了Category和Extension了，虽然平常使用的频率还是可以的，但真正的区别也不是很清楚；

于是乎查阅了相关资料并且结合自己的实践得出以下结论：

Category: 字面意思是类别。

* 主要用来扩展方法，并且适用于subclass;
* 只能添加readonly的属性, 如果要添加readwrite属性必须在runtime过程中用objc_setAssociatedObject()和objc_getAssociatedObject()方法来实现属性的get与set方法;
Extension:字面意思是扩展。

* 同样可以扩展方法和属性，但局限于原始类；
* 声明的方法必须在@implemention中实现，不然编译器会报warning;
在extension中可以定义可写的属性，公有可读、私有可写的属性(Publicly-Readable, Privately-Writeable Properties)一般这样实现。

