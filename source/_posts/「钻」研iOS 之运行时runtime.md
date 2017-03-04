---
title: 『钻』研iOS 之运行时runtime
date: 2017-02-12 16:20:24
tags:
---

## 何为iOS的Runtime?
苹果是这样定义的：`The Objective-C runtime is a runtime library that provides support for the dynamic properties of the Objective-C language, and as such is linked to by all Objective-C apps. `，大概的意思是OC runtime是一个运行时的库，能给Objective-C提供动态特性的支持，苹果的所有基于Objective-C开发的app都会依赖于这个库。

```objc
[array insertObject:foo atIndex:5];
//上面的方法在runtime环境下会变成这样：
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);
``` 
PS：通过这个Clang命令：`clang -readwrite-objc XXX.m -o XXX.cpp`，可以了解更多OC转C++的代码信息。

可以这么说，运行时库是Objective-C的运行基础，运行时通过面向对象的特性创造了Objective-C这编程语言。运行时本身的设计也是消息传递（`objc_msgSend`）和消息转发（`objc_msg`）。

## 消息传递 & 消息转发
消息传递的过程：
iOS的任何方法调用都是一个消息传递过程。runtime首先通过isa指针找到相应的对象（Class）或者实例(instanced object)，然后遍历methodLists，当列表中找到方法时向该方法发送执行的消息，并且通过cache缓存使用过的method。
下面来看下objc_class的结构体：
```objc

```

## 动态特性

`dyld`：是 the dynamic link editor 的缩写，它是苹果的动态链接器。
<img src="" widht=""/>

## 运行时三要素

```objc
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#if SUPPORT_PACKED_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif


#if SUPPORT_INDEXED_ISA

# if  __ARM_ARCH_7K__ >= 2

#   define ISA_INDEX_IS_NPI      1
#   define ISA_INDEX_MASK        0x0001FFFC
#   define ISA_INDEX_SHIFT       2
#   define ISA_INDEX_BITS        15
#   define ISA_INDEX_COUNT       (1 << ISA_INDEX_BITS)
#   define ISA_INDEX_MAGIC_MASK  0x001E0001
#   define ISA_INDEX_MAGIC_VALUE 0x001C0001
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t indexcls          : 15;
        uintptr_t magic             : 4;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 7;
#       define RC_ONE   (1ULL<<25)
#       define RC_HALF  (1ULL<<6)
    };

# else
#   error unknown architecture for indexed isa
# endif

// SUPPORT_INDEXED_ISA
#endif

};

```

* isa：isa_t对象指向对象本身的指针，类指向类本身的指针。
* @selector SEL 选择器，会先在类里注册这个方法
* IMP 实现指针

## Method Swizzling
method_swizzling本质是交换IMP的实现

+load
+initialization


## 参考资料
[Objective-C Runtime](https://developer.apple.com/reference/objectivec/objective_c_runtime?language=objc)
[Type Encoding](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)


