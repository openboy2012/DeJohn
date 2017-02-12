title: "『钻』研iOS 之 内存管理（一）"
date: 2017-02-05 09:20:33
tags: 
  - Objective-C
  - iOS基础
  - 内存管理
  - ARC & MRC
categories:
---

iOS内存管理在我这6年工作经验过程中的变化可谓是翻天覆地，由于ARC（Automatic Refenerce Counting）的出现，大大简化了iOS开发者的内存管理优化工作。ARC不是垃圾回收机制，虽然开发者不用像以前那样刻意关心内存管理问题，但也不是意味着我们不需要了解iOS的内存管理。
## 堆（Heap）与栈(Stack)
讲到内存，不得不先讲下堆栈，堆和栈的定义大家自己[百度百科](http://baike.baidu.com/link?url=2DK5Zuh6uTuATG_7IKOH8kNayWss2L_pOchRmk1mh6iYM5wo60g_QU1S4IoopLMQzy8f937n7U6_cwQSgZvRXnXo-b0QOEE2mfI3Ebo-u4S)，本文不再展开，我只阐述自己的总结：
栈（操作系统）：由操作系统（编译器）自动分配释放，存放函数的参数值，局部变量的值、常量（int、bool）等。其操作方式类似于数据结构中的栈(Stack)。
堆（操作系统）：一般由程序员分配释放，一般的指针对象创建都存放在堆区，若程序员不释放，程序结束时可能由OS回收，分配方式类似于链表（Queue）。
<!--more-->
本文主要围绕着堆区（Heap）的内存管理进行讲解。
## Reference Counting
先讲「Reference Counting」引用计数，不管是MRC或者ARC都会有这个Reference Counting。Objective-C的内存管理方式采用的是保留计数（retainCount）的方式来保证内存的可用性，内存初始化（alloc&init）的时候retainCount为1，当内存被其他指针引用（retain）一次以后该内存的retainCount会+1，当引用的这块内存的这个指针被释放（release）一次以后该内存的retainCount会-1，当retainCount=0时内存会被标记为可回收，会执行dealloc方法进行真正的资源释放销毁操作。
如果内存在不再使用的时候retainCount没有变成0（指针已经置成nil，但当时分配的对象没有release的情况下，形成了野指针），那就是内存泄露（Memory leak）；
如果内存在使用的时候retainCount已经为0（指针的地址还是存在，但这个地址已经被release的情况下）的时候，那就是内存溢出（EXC_BAD_ACCESS）。
引用计数的概念在很多语言中都有体现：比如C++的智能指针：std::shared_ptr。还有更多[Reference Counting](https://en.wikipedia.org/wiki/Reference_counting)的说明请看[Wiki](https://en.wikipedia.org/wiki/Reference_counting)。
## MRC时代（Before 2012）
`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行关键代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。

MRC（Mannual Reference Counting）手动引用计数内存管理，就是Objective-C对象的内存的创建和释放需要开发者手动管理。
先看一段非在NSAutoReleasePool的代码栗子(UI元素相关的「简单」代码)：
```objc
//interface
@property (nonatomic, retain) UIButton *btnMRC; //MRC用retain修饰对象，保持内存持有

//implementation
- (void)dealloc
{
    NSLog(@"_btnMRC.retainCount is %lu", _btnMRC.retainCount);
    //[_btnMRC release]; //这里release需要注意，如果构建的指针对象是通过实例方法创建如alloc、new等，这里需要release一次，否则
    NSLog(@"_btnMRC.retainCount is %lu", _btnMRC.retainCount);
    [super dealloc];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    NSLog(@"_btnMRC.retainCount is %lu", _btnMRC.retainCount);
    _btnMRC = [UIButton buttonWithType:UIButtonTypeCustom]; //类方法创建的对象在ARC的情况下会默认加上autorelease,所以在dealloc不需要再手动释放
    //_btnMRC = [[UIButton alloc] init]; //实例方法创建的对象，需要dealloc的时候进行一次release释放
    NSLog(@"_btnMRC.retainCount is %lu", _btnMRC.retainCount);
    [self.view addSubview:_btnMRC]; //addSubView方法会retainCount+1
    NSLog(@"_btnMRC.retainCount is %lu", _btnMRC.retainCount);
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    [_btnMRC setFrame:CGRectMake(100, 100, 100.0, 20)];
    [_btnMRC setTitle:@"MRC Button" forState:UIControlStateNormal];
    _btnMRC.backgroundColor = [UIColor redColor];
}

- (void)viewDidUnload
{
#warning iOS 6.0以前如果收到memory warning的警告，系统会先viewDidUnload来处理view的释放，如果不在viewDidUnload里处理释放操作，然后系统会重新加载viewDidLoad方法，如果那里的UI内存没有处理得很好，很容易造成野指针的内存泄露。iOS 6.0以后已经废弃了这个方法，跟进ARC的进度
//    [self.btnMRC release];
    [super viewDidUnload];
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
#warning iOS 6.0 以后在这里处理非UI资源的释放内存
}
```
PS:这里我使用的是内部变量的指针（带_），从Xcode 4.4版本开始已经可以不显示调用@synthesize来合成get/set方法了，在.m文件内部可以直接用_xxx替代@property声明了变量xxx。这个也是Clang编译器的功劳。

这么多代码只完成了一个简单Button的手动引用计数内存管理，那么在一个复杂界面的UIViewController中有很多的UI元素和其他成员变量，每个对象都需要这么细心地去处理内存管理，这是一件多么恐怖的事情。

MRC时代的UIViewController实现文件随随便便都是好几千行代码，不管是阅读还是维护，都是让人心里抓狂的。

再来看个NSAutoReleasePool的代码栗子:
```objc
- (IBAction)testAutoReleasePool
{
    NSLog(@"====== autoreleasepool method begin ======"); //增加打印分割标识
    NSAutoreleasePool *pool = nil; //声明一个显式的NSAutoReleasePool
    if (self.switchShowAutoReleasePool.on) {//是否初始化显式的NSAutoReleasePool
         pool = [[NSAutoreleasePool alloc] init]; //显式创建一个NSAutoReleasePool
    }
    for (int i = 0; i < 5; i++) {
        if (self.switchAutoRelease.on) {
            __unused DDObject *object = [[[DDObject alloc] init] autorelease]; //有autorelease关键字方法，会自动加标识到NSAutoReleasePool里去：如果有显式创建的pool存在，该对象会在显式的Pool里被标识，当Pool释放时该对象；如果没有显示的Pool，会加到系统自动创建的NSAutoReleasePool（隐式的Pool）里去，跟NSRunLoop有关，该Pool会在当次RunLoop结束时执行release。通常是『{}』的}的时候触发，看打印结果可知。

        } else {
            DDObject *object = [[DDObject alloc] init];
            [object release]; //创建以后就release释放
        }
    }
    if (self.switchShowAutoReleasePool.on) {//是否初始化显式的NSAutoReleasePool
        [pool release];
    }
    NSLog(@"====== autoreleasepool method end ======");  //增加打印分割标识
}
```
以上代码的运行结果如下：
<img src="http://ipa-download.qiniudn.com/F70C808F-2445-4E7D-A219-3E1120CD105E.png" width="714"/>

`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。

NSAutoReleasePool的显式调用会改变对象的生命周期，原本应该在方法块外部执行dealloc的对象会被提前执行，所以NSAutoReleasePool以及autorelease关键字方法的不合理使用，都会造成一些莫名奇妙的问题，如果你不了解iOS内存管理，在那时候真的很难写出漂亮的代码出来。

下面来解释下出现的关键字方法：
`alloc&init`：通常情况下2个方法是同时出现的，初始化一个对象，分配内存空间，所以当前对象的retainCount为1；   
`retain`：对象的retainCount+1；   
`release`：对象的retainCount-1；   
`copy`:对象的retainCount+1；   
`addSubview`：仅限于UI的元素对象。同retain的效果，retainCount会+1；同理
`removeFromSuperView`方法调用的时候会retainCount-1；   
removeFromSuperView方法通常由系统自动完成;
`autorelease`：会把对象加入到就近的NSAutoReleasePool（显式优先），当Pool释放时对该对象进行一次release操作。
`dealloc`：当对象的retainCount为0的时候会调用这个方法，用来销毁对象的内部处理。

### dealloc的线程小知识
dealloc方法的执行线程是对象最后一次release的线程，这里就会存在一个问题：如果在dealloc里发生了非常耗时的操作，就会出现主线程卡住的情况，通常我们会重载UIViewController的dealloc方法（基本上都是主线程）来释放一些资源，比如通知（NSNotification）的移除、C++的跨平台库的析构等。如果C++的跨平台库的析构出现了耗时操作，很有可能会卡我们的主线程，所以使用的时候要格外的注意。

PS:写这么多代码我只是想表达之前手动管理有多么地复杂，iOS开发的入门门槛也比现在高很多。最重要的事，我只是想表达：我确确实实是从那个时代过来了，真真实实得写了这么多年的代码>_<

好在苹果的工程师们早早得注意到了这个问题，设计了这个[Clang](https://en.wikipedia.org/wiki/Clang)编译器以及ARC的内存管理方式，替开发者来处理内存管理的事情，大大促进了iOS的开发效率，也大大降低了iOS开发的门槛。

## ARC时代(2012~至今)
ARC（Automatic Reference Counting）自动引用计数内存管理，通过编译器（Clang Complier），本质上还是会使用到retain、release等关键字方法，只是不是开发者手动添加，而是编译器在编译过程中添加retain、release等关键字方法到相应的代码行。

<img src="https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Art/ARC_Illustration.jpg", width='751'/>

还是先来看看之前的代码在ARC环境下是怎么样的：
```objc
//interface
@property (nonatomic, strong) UIButton *btnARC; //ARC环境下用strong保持强引用关系

//implementation
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    /* 
     ARC环境下，Clang编译器会对_btnARC进行retain方法，所以开发者无需显示调用retain方法，而且Clang编译器已经在ARC环境下把retain方法标记为不可用。
     */
    _btnARC = [UIButton buttonWithType:UIButtonTypeCustom]; //ARC情况下不需要特别关心内存管理，开发门槛大大降低。
    //_btnARC = [[UIButton alloc] init];
    [self.view addSubview:_btnARC];
    .
    .
    .
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    [_btnARC setFrame:CGRectMake(100, 100, 100.0, 20)];
    [_btnARC setTitle:@"ARC Button" forState:UIControlStateNormal];
    _btnARC.backgroundColor = [UIColor redColor];
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}
```
啥？只有么点？对，确实只要这些，编译器已经帮开发者加了上内存管理的方法(retain/release)代码，所以ARC下无需特别关注iOS的内存管理。只需要了解以下关键字即可：
`__strong/strong`：强引用，与MRC下的retain对应，在编译阶段加上的是retain方法，会对引用计数作增加；  
`__weak/weak`：弱引用，引用计数不发生变化，某些特定情况下与MRC的assign对应；  
`copy`：一般情况下是浅拷贝，某些特定条件下是深拷贝（下一讲我会仔细讲解『深拷贝与浅拷贝』）；   
`__autoreleasing`：ARC环境下标识autorelease方法关键字，在编译阶段加上的是autorelease方法；   
`__unsafe_unretained`：弱引用，引用计数不发生变化，与weak的区别是这个处理在内存对象被释放以后不会对指针地址置成nil，weak会在释放对象以后把指针地址置成nil。

再来看下AutoReleasePool在ARC的情况下的代码实现：
```objc
- (IBAction)testAutoReleasePool
{
    NSLog(@"====== autoreleasepool method begin ======"); //增加打印分割标识
    
    if (self.switchShowAutoReleasePool.on) {//是否初始化显式的@autoreleasepool
        @autoreleasepool { //创建显示autorelease pool，与NSAutoReleasePool的效果一样
            [self doForMethod]; //执行for循环方法
        }
    } else {
        [self doForMethod];
    }
    
    NSLog(@"====== autoreleasepool method end ======");  //增加打印分割标识
}

- (void)doForMethod {
    for (int i = 0; i < 5; i++) {
        if (self.switchAutoRelease.on) { //是否使用__autoreleasing关键字
            __unused __autoreleasing DDObject *object = [[DDObject alloc] init]; //有__autoreleasing关键字方法,与MRC下的autorelease效果一样
        } else {
            __unused DDObject *object = [[DDObject alloc] init]; //编译器会自动加上release
        }
    }
}
```
`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行关键代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。

执行这段代码的打印效果与MRC是一模一样的（同样参考MRC运行的结果图）。只是`@autoreleasepool{}`调用更加简单，从代码和运行结果中也可以得出结论：默认状态下编译器（Clang Complier）对Objective-C的对象只会加上release的关键字方法（打印信息是alloc与dealloc交叉的那种情况），相对来说编译器并没有想象中的那么智能。
在高级编程情况下还是需要开发者合理地使用`@autoreleasepool`和`__autoreleasing`关键字来控制对象的生命周期。
如何合理？就是需要明白iOS的内存管理的精髓--真正了解MRC的工作原理。

从AutoReleasePool的栗子可以分析出来，ARC与MRC内存管理的底层实现其实没有什么变化，只是苹果的工程师们在ARC环境的设计理念下花了大量精力把内存管理的工作交给了编译器来处理，简化开发内存管理的工作。 

### assign与weak的区别
`assign`：ARC&MRC环境下通用，通常修饰的是常量，比如int、bool等；在MRC环境下，@property时可以修饰（id<delegate>对象）来防止循环引用、可以修饰IBOutlet出来的UI元素对象、当然也可以修饰不想对retainCount作增加的对象（引用计数不发生变化）；
`weak`：只能在ARC环境下使用，weak只能修饰OC对象（包含delegate），不能修饰常量或者其他非OC对象。

下一讲我会讲解『block内存管理』、『C++混编的内存管理』和『深拷贝与浅拷贝』。 

参考资料：
[堆栈](http://baike.baidu.com/link?url=2DK5Zuh6uTuATG_7IKOH8kNayWss2L_pOchRmk1mh6iYM5wo60g_QU1S4IoopLMQzy8f937n7U6_cwQSgZvRXnXo-b0QOEE2mfI3Ebo-u4S)
[Automatic Reference Counting](https://en.wikipedia.org/wiki/Automatic_Reference_Counting)
[Transitioning to ARC Release Notes](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html) 
[Clang](https://en.wikipedia.org/wiki/Clang)


