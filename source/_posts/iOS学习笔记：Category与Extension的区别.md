title: "iOS学习笔记：Category与Extension的区别"
date: 2015-03-11 17:18:49
tags: 
- iOS学习笔记
categories: 
---

## 前言
作为一个有4年以上iOS开发经验的开者者来说，使用Category和Extension的场景应该是数不胜数的，也不知道从Xcode的哪个版本开始，创建一个新的UIViewController类会默认添加一个该类的Extension，而我也习惯于在这个Extension上添加一些不公开的属性。
<!--more-->
## 我的分析
仔细查阅了相关资料并且结合自己的实践得出以下结论：

Category：字面意思是类别。

* 主要用来扩展方法，并且适用于subclass;
* 只能添加readonly的属性, 如果要添加readwrite属性必须在runtime过程中用objc_setAssociatedObject()和objc_getAssociatedObject()方法来实现属性的get与set方法;  

Extension：字面意思是扩展。

* 同样可以扩展方法和属性，但局限于原始类；
* 声明的方法必须在@implemention中实现，不然编译器会报warning;
在extension中可以定义可写的属性，公有可读、私有可写的属性(Publicly-Readable, Privately-Writeable Properties)一般这样实现。  
## 总结
综上所述，我们通常要封装一些公共方法的时候我们可以考虑使用Category的方式。  
如果我们想在原始类上面增加一些不公开的方法、属性（私有方法、属性）时可以新建一个Extension来解决问题。

