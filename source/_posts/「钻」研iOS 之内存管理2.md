title: "『钻』研iOS 之 内存管理（二）"
date: 2017-02-06 20:00:29
tags: 
  - Objective-C
  - iOS基础
  - 内存管理
  - Block
  - C++混编
categories:
---

上一讲中我把内存管理的基础、MRC与ARC作了自己的一些理解和经验分享，这一讲我会继续iOS内存管理方面的讲解。

##Block的内存管理
Block有3种内存类型：
`NSGlobalBlock`（全局块）：block内部没有引用任何外部变量的block是全局block（Global Block）；
`NSStackBlock`（栈内存块）：block内部引用了block之外的外部变量的block是栈内存block（Stack Block）；
`NSMallocBlock`（堆内存块）：block内部引用了block之外的外部变量的block并且被copy了一次是堆内存block（Malloc Block）。
<!--more-->
`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行关键代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。
###先看Block内存变化的相关代码
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    //block内存变化示例
    [self blockMemoryChangeExample];
    
    //block关键字变化引用示例
    //[self blockExample];
    
    //block循环引用示例
    //[self cycleReferenceBlockExample];
}

/**
  block内存变化示例 内存的变化顺序为{->[alloc]GlobalBlock、->[alloc]StackBlock->[copy]MallocBlock}, block最初被设定的内存类型为全局内存类型（无外部变量），当block内部出现外部变量时转换为栈内存类型，再当此栈内存block被外部持有（copy）操作时会变成堆内存类型。
 */
- (void)blockMemoryChangeExample {
    void (^ddkitBlockGlobal)() = ^(){
        int b = 18;
        //没有引用外部变量的block是__NSGlobalBlock__
        NSLog(@"this is global block b is %d", b);
    };
    NSLog(@"this block type is %@", ddkitBlockGlobal); //block内部没有使用任何的外部变量，所以是__NSGlobalBlock__
    ddkitBlockGlobal();
    
    int a = 28;
    
    //使用__weak能阻止编译器对block进行copy操作，从而保证了block的内存类型是栈block(__NSStackBlock__)
    __weak void (^ddkitBlockStack)() = ^(){
        int b = a;
        //使用外部变量的block是__NSStackBlock__
        NSLog(@"this is stack block, b is %d", b);
    };
    NSLog(@"this block type is %@", ddkitBlockStack); //block内部使用了的外部变量a，所以是__NSStackBlock__
    ddkitBlockStack();
    
    //block赋值操作
    void (^blockMemoryType)(void) = ddkitBlockGlobal; //对全局内存block copy不会形成堆内存block，内存地址也没有发生改变
    NSLog(@"copy from ddkitBlockGlobal blockMemoryType is %@, ddkitBlockGlobal is %@", blockMemoryType, ddkitBlockGlobal);
    blockMemoryType = ddkitBlockStack; //对栈内存block copy会形成堆内存block, 会产生新的地址
    NSLog(@"copy from ddkitBlockStack blockMemoryType is %@, ddkitBlockStack is %@", blockMemoryType, ddkitBlockStack);
}
```
运行结果如下：
<img src="http://ipa-download.qiniudn.com/D8E67C22-DB73-4BA4-B2E0-FAB50CA77FFB.png" width="400"/>
####总结分析：
Block内存的变化流程图如下：
<img src="http://ipa-download.qiniudn.com/block%E7%9A%84%E5%86%85%E5%AD%98%E5%8F%98%E5%8C%96.jpg" width="300"/>
我的总结：
1.全局内存Block和栈内存Block都是系统自己进行内存管理的，与类的方法的内存管理一致；
2.`__weak`显示调用可以阻止编译器对栈内存Block进行copy，所以`ddkitBlockStack`保留了栈内存类型b；
3.Xcode编译器在ARC环境下给对象的默认关键字是`__strong`，所以`blockMemoryType = ddkitBlockStack`赋值形成了堆内存Block。
###MRC环境下尝试Block的内存引用计数相关的示例代码
```objc
#pragma mark - MRC下测试block相关的retain、release、copy操作
//全局内存Block尝试retain、copy、release等方法
- (void)mrcGlobalBlockExample
{
    NSLog(@"===== MRC GlobalBlock retainCount methods begin =====");
    void (^ddkitBlockGlobal)() = ^(){
        int b = 18;
        //没有引用外部变量的block是__NSGlobalBlock__
        NSLog(@"this is global block b is %d", b);
    };
    ddkitBlockGlobal();
    [ddkitBlockGlobal release]; //release操作对全局内存Block无效
    
    NSLog(@"ddkitBlockGlobal retainCount = %lu", [ddkitBlockGlobal retainCount]);
    void (^ddkitBlockGlobalRetain)() = [ddkitBlockGlobal retain]; //尝试retain方法
    NSLog(@"ddkitBlockGlobal[%@] retainCount = %lu, ddkitBlockGlobalRetain[%@] retainCount = %lu", ddkitBlockGlobal, [ddkitBlockGlobal retainCount], ddkitBlockGlobalRetain, [ddkitBlockGlobalRetain retainCount]);
    
    void (^ddkitBlockGlobalCopy)() = [ddkitBlockGlobal copy]; //尝试copy方法
    NSLog(@"ddkitBlockGlobal[%@] retainCount = %lu, ddkitBlockGlobalCopy[%@] retainCount = %lu", ddkitBlockGlobal, [ddkitBlockGlobal retainCount], ddkitBlockGlobalCopy, [ddkitBlockGlobalCopy retainCount]);
    
    NSLog(@"===== MRC GlobalBlock retainCount methods end =====");

}

//栈内存Block尝试retain、copy、release等方法
- (void)mrcStackBlockExample
{
    NSLog(@"===== MRC StackBlock retainCount methods begin =====");

    int a = 10;
    void (^ddkitBlockGlobal)() = ^(){
        int b = 28 + a;
        //引用外部变量的block是__NSStackBlock__
        NSLog(@"this is stack block b is %d", b);
    };
    ddkitBlockGlobal();
    [ddkitBlockGlobal release]; //release操作对栈内存Block无效
    
    NSLog(@"ddkitBlockGlobal retainCount = %lu", [ddkitBlockGlobal retainCount]);
    void (^ddkitBlockGlobalRetain)() = [ddkitBlockGlobal retain]; //尝试retain方法
    NSLog(@"ddkitBlockGlobal[%@] retainCount = %lu, ddkitBlockGlobalRetain[%@] retainCount = %lu", ddkitBlockGlobal, [ddkitBlockGlobal retainCount], ddkitBlockGlobalRetain, [ddkitBlockGlobalRetain retainCount]);
    
    void (^ddkitBlockGlobalCopy)() = [ddkitBlockGlobal copy]; //尝试copy方法
    NSLog(@"ddkitBlockGlobal[%@] retainCount = %lu, ddkitBlockGlobalCopy[%@] retainCount = %lu", ddkitBlockGlobal, [ddkitBlockGlobal retainCount], ddkitBlockGlobalCopy, [ddkitBlockGlobalCopy retainCount]);
    [ddkitBlockGlobalCopy release]; //release操作对堆内存Block有效，所以下一行注释掉代码出现崩溃。
    //NSLog(@"ddkitBlockGlobalCopy = %@",ddkitBlockGlobalCopy);
    
    NSLog(@"===== MRC StackBlock retainCount methods end =====");
}
```
运行结果如下：
<img src="http://ipa-download.qiniudn.com/9901FB96-1D67-4E30-B09B-BFDB963193F4.png" width="400"/>
####总结分析：
1.全局内存Block和栈内存Block的copy、release、retain方法都是无效的，指针地址和retainCount都不会发生变化。
2.只有栈内存Block通过copy方法能变成堆内存Block（retain方法无效，retain方法对所有内存类型的block都起不到引用计数+1的作用），然后通过release方法可以释放堆内存Block。
 
###blockExample的示例代码
并点击UI中的Crash按钮（[demo](https://github.com/openboy2012/DDCategory)示例）发生崩溃(连续点击几次)：
```objc

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    //block内存变化示例
    //[self blockMemoryChangeExample];
    
    //block关键字变化引用示例
    [self blockExample];
    
    //block循环引用示例
    //[self cycleReferenceBlockExample];
}

- (IBAction)clickCrashButton:(id)sender
{
    self.isUseCrashCode = YES;
    [self blockExample];
}

- (void)blockExample {
     int a = 49;
    
//    __weak //使用__weak也能阻止编译器对block进行copy操作，而且更加安全，这里使用__unsafe_unretained只是试验关键字的效果
    __unsafe_unretained //此处加上__unsafe_unretained是因为在ARC环境下可以阻止Clang编译器对block进行copy操作，从而保持block的内存类型是栈类型
    void (^ddkitBlockNoCopy)() = ^(){
        int b = a;
        /*
         因为引用了block之外的外部变量，所以是栈block
         这里运行到了打印ddkitBlockTwo的时侯崩溃，因为使用了__unsafe_unretained关键让ARC环境下不对block进行copy操作。
         结合打印的日志，这里证明了2点：
         1.__unsafe_unretained确实在ARC的环境下让block赋值操作时不自动copy，从而保证了block的内存类型是栈block(__NSStackBlock__);
         2.__unsafe_unretained确实不安全，在运行到}结束时，指针对象已经被清理，但指针还是保存着之前的地址，再访问时造成了EXC_BAD_ACCESS
         */
        #warning 因为ddkitBlockTwo是局部变量，不存在循环引用的问题，可以放心地在{}引用self
        if (self.isUseCrashCode) {
            NSLog(@"this block type is %@ b is %d", ddkitBlockNoCopy, b);
        } else {
            NSLog(@"this is stack block, b is %d", b);
        }
    };
    NSLog(@"this block type is %@", ddkitBlockNoCopy); //这里能正确打印block的类型，此处为__NSStackBlock__
    ddkitBlockNoCopy();
    
    //在ARC环境下，默认关键字是__strong，会对block进行copy操作，所以block的内存类型会变成堆类型；同理在MRC环境下需要显示调用copy方法来把GlobalBlock或者StackBlock转换成MallocBlock
    void (^ddkitBlockMalloc)() = ^(){
        int b = a + a;
        //引用了外部变量，并且把这个block赋值给了ddkitBlockTwo(有copy操作)，所以是__NSMallocBlock__
        NSLog(@"this block type is %@ b is %d", ddkitBlockMalloc, b); //因为ddkitBlockThree是局部变量，在ARC环境下到了{}之外就会被清理，所以会打印null空指针
    };
    NSLog(@"this block type is %@", ddkitBlockMalloc);
    ddkitBlockMalloc();
}
```
代码执行的结果如下：
<img src="http://ipa-download.qiniudn.com/955B9457-ADF5-4898-99E4-4B1A1E70837A.png" width="400"/>
####总结分析：
代码运行到了打印ddkitBlockNoCopy的时侯发生崩溃--`EXC_BAD_ACCESS`：因为ddkitBlockNoCopy使用了`__unsafe_unretained`关键字，所以在ARC环境下没有对ddkitBlockNoCopy进行copy操作，从而ddkitBlockNoCopy保持了栈内存Block。当程序运行到『}』结束时，ddkitBlockNoCopy指针内存已经被清理，但指针还是保存着地址（指针没有置nil），接着在Block内部访问ddkitBlockNoCopy指针时造成了`EXC_BAD_ACCESS`。
结论：
`__unsafe_unretained`确实不安全，没有及时把指针置成nil是非常有风险的事情。我们需要`__unsafe_unretained`替换成`__weak`，这样程序才是安全的。

###Block的循环引用
```objc
//interface 
typedef void(^ CycleReferenceBlock)(void);

@property (nonatomic, copy) CycleReferenceBlock block; //声明block成员变量，ARC环境下，用strong与copy关键字都一样，都会对block内存进行copy

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    //block内存变化示例
    //[self blockMemoryChangeExample];
    
    //block关键字变化引用示例
    //[self blockExample];
    
    //block循环引用示例
    [self cycleReferenceBlockExample];
}

//implementation
- (IBAction)cycleReferenceBlockExample
{
    if (self.switchCycleReferen.on) {
        __weak __typeof(self) bSelf = self; //因为block是self的成员变量，会造成循环引用的问题，所以在block外部先__weak一次打破循环引用
        self.block = ^() {
            NSLog(@"this is cycle reference in blocks, result is %d, cycle reference is broken, this viewContoller can dealloc", bSelf.isUseCrashCode);
        };
        self.block();
    } else {
        self.block = ^() {
            NSLog(@"this is cycle reference in blocks, result is %d, cycle reference is happened, this viewContoller can't dealloc", self.isUseCrashCode);
        };
        self.block();
    }
}
```
代码运行的结果如下：
<img src="http://ipa-download.qiniudn.com/DA106C4E-C48B-4717-AE22-F33E2901977A.png" width="400"/>
`结论`：只有堆内存的block才`有可能`发生循环引用，本栗子中cycleReferenceBlock被self进行copy持有，然后如果在block内部引用self的话就存在了block持有self、self持有block的典型Block循环引用问题，所以当前的UIViewController无法释放。
在block外部先把self进行`__weak`方法把self弱引用成weakSelf，在block内部使用弱引用的weakSelf时就能正常释放UIViewController了。

##Objective-C与C++混编（CF对象）
本人有过OC与C++对象混编的经验，所以把bridge的相关内存管理问题简单描述下：
首先，ARC环境只适用Objective-C/Swift对象，C++对象（包含苹果的CF对象）都需要开发者手动内存管理。
先来讲解下ARC环境下OC与C++混编时用到的关键字：
`__bridge`：CF对象（C++对象）桥接OC对象，没有牵扯到对象所有权（主要是内存管理的权限）交接；
这就意味着OC创建的对象要在OC端释放，C++创建的对象要在C++端释放，两者对象互不影响。
`__bridge_transfer`：常用在将CF对象（C++对象）转换成OC对象时，将CF对象（C++对象）的所有权交给OC对象，此时ARC就能自动管理该内存；（作用同CFBridgingRelease()）
使用这个关键字以后C++创建的对象不需要在C++端释放，ARC会对其进行内存管理，如果C++端把内存释放，OC端会出现`EXC_BAD_ACCESS`。
`__bridge_retained`：（与__bridge_transfer相反）常用在将OC对象转换成CF对象时，将OC对象的所有权交给CF对象来管理；(作用同CFBridgingRetain()) 
使用这个关键字以后OC的对象会被C++来管理，如果OC对象提前释放了，会造成C++端`EXC_BAD_ACCESS`。

##深拷贝与浅拷贝
`深拷贝`是内容拷贝，产生了新的指针，内存生命周期重新开始； 
`浅拷贝`是指针拷贝，retainCount+1。
###系统对象的可变与不可变对象
`immutable不可变对象`（NSString、NSDictionary、NSArrary、NSSet等） 
`mutable可变对象` (NSMutableString, NSMutableDictionary, NSMutableArray, NSMutableSet等) 
###集合与非集合对象
在非集合类对象中：对immutable对象进行copy操作，是指针复制，mutableCopy操作时内容复制；对mutable对象进行copy和mutableCopy都是内容复制。用代码简单表示如下： 
```objc
	[immutableObject copy] // 浅复制
	[immutableObject mutableCopy] //深复制
	[mutableObject copy] //深复制
	[mutableObject mutableCopy] //深复制
```
在集合类对象中，对immutable对象进行copy，是指针复制，mutableCopy是内容复制；对mutable对象进行copy和mutableCopy都是内容复制。但是：集合对象的内容复制仅限于对象本身，对象元素仍然是指针复制。用代码简单表示如下： 
```objc
	[immutableObject copy] // 浅复制
	[immutableObject mutableCopy] //单层深复制
	[mutableObject copy] //单层深复制
	[mutableObject mutableCopy] //单层深复制 
```
代码示例：
```objc
- (void)copyExample
{
    _dict = [[NSDictionary alloc] initWithObjectsAndKeys:@"1",@"key",nil];
    NSLog(@"dict retainCount is %lu", _dict.retainCount);
    NSDictionary *dict2 = [self.dict copy];
    NSLog(@"dict2 retainCount is %lu, dict retainCount is %lu", dict2.retainCount, self.dict.retainCount);
    NSDictionary *mutableCopyDict = [self.dict mutableCopy];
    NSLog(@"mutableCopyDict retainCount is %lu, dict retainCount is %lu", mutableCopyDict.retainCount, self.dict.retainCount);
    NSLog(@"dict addess is %p dict2 address is %p mutableCopyDict address is %p", self.dict, dict2, mutableCopyDict);
    [_dict release];
    [_dict release]; //因为immutable类型对象copy是指针拷贝，没有产生新的指针，只是在原来的指针retainCount+1，所以可以对_dict进行2次release而不发生崩溃，第二次[_dict release]与[dict2 release]等价
    [mutableCopyDict release]; //内容复制产生了新的指针，需要手动释放才能避免内存泄漏。
    
    _mDict = [[NSMutableDictionary alloc] initWithObjectsAndKeys:@"1",@"key",nil];
    NSLog(@"mDict retainCount is %lu", _mDict.retainCount);
    NSDictionary *dict3 = [self.mDict copy];
    NSLog(@"dict3 retainCount is %lu, mDict retainCount is %lu", dict3.retainCount, self.mDict.retainCount);
    NSDictionary *dict4 = [self.mDict mutableCopy];
    NSLog(@"dict4 retainCount is %lu, mDict retainCount is %lu", dict4.retainCount, self.mDict.retainCount);
    NSLog(@"mDict addess is %p dict3 address is %p, dict4 address is %p", self.mDict, dict3, dict4);
    [_mDict release];
    [dict4 release]; //内容复制产生了新的指针，需要手动释放才能避免内存泄漏。
    [_mDict release]; //因为mutable类型对象copy以后产生了新的指针，对mDict进行释放是发生崩溃
}
```
运行结果：
<img src="http://ipa-download.qiniudn.com/DFDA366F-625D-4874-9A57-7A8B1EA0B5FB.png" width="400"/>
`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行关键代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。

结合实例代码和运行结果总结：
`immutable`对象`copy`以后引用计数+1,`mutableCopy`产生新的指针地址，内存管理重新开始；
`mutable`对象`copy`与`mutableCopy`都产生了新的指针地址，内存管理重新开始。

###为什么我们@property一个NSString使用copy修饰？
因为`NSString`是一个`immutable对象`，使用copy是指针复制（与retain效果一样），同时`NSString`可以指向`NSMutableString`创建的指针，使用copy会对`NSMutableString`的指针进行内容拷贝（产生新的指针），这时如果`NSMutableString`的指针内容即使发生了改变，也不会影响到`NSString`的指针内容。
这个原理也适用NSDictionary、NSSet和NSArray等系统对象。我们可以根据自己的需求来选择`strong`或者`copy`关键字来修饰相应的属性。

###关于自定义对象的copy
上文中提到的对象都是系统级的对象，自定义对象要实现`copy`方法就要实现NSCopying协议。如果对没有实现`NSCopying`协议（主要是`copyWithZone：`方法的实现）的对象进行copy操作会发生崩溃，报`*** Terminating app due to uncaught exception 'NSInvalidArgumentException', 'reason: '-[XXX copyWithZone:]: unrecognized selector sent to instance 0xXXXXXX'`。

参考资料：
[Working with Blocks](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)
[Toll-Free Bridged Types](https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/tollFreeBridgedTypes.html#//apple_ref/doc/uid/TP40010677-SW1)
[深拷贝与浅拷贝](https://www.zybuluo.com/MicroCai/note/50592)

