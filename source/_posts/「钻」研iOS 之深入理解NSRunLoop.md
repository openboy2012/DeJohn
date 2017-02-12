title: 『钻』研iOS 之深入理解NSRunLoop
date: 2017-02-12 13:43:41
tags:
  - Objective-C
  - iOS基础
  - RunLoop
  - iOS事件响应
---

差不多2年前，我对[NSRunLoop](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/uid/TP40003725)有过应用层面上的理解，现在我们通过苹果开源在github上的[swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation)来深入了解下苹果是如何实现的这个CFRunLoop(NSRunLoop是对CFRunLoop的封装)。
当然，还是2年前的那句话：
`NSRunLoop`其英文释义一样，是运行一个无限循环，她是跟线程一起存在的。在主线程中NSRunLoop是默认启动的；在多线程中NSRunLoop默认不是启动的，需要开发者手动运行才能启动。 

## RunLoop的概念
RunLoop本质上是一个[Event Loop](https://en.wikipedia.org/wiki/Event_loop)，实现的是一个`do-while`循环，主要用来处理事件消息。
<!--more-->
苹果设计的高明之处：是进入do-while循环之后不会导致死循环，因为`mach_port`的存在，会让这个RunLoop在某个事件处睡眠，事件循环就暂停，再在需要的时候通过`mach_port`唤醒RunLoop，事件循环就会继续处理。

## \_\_CFRunLoop & \_\_CFRunLoopMode
源码中的结构体定义如下：
```objc
struct __CFRunLoop {
    ...
    pthread_mutex_t _lock;	//对象锁，线程保护
    ...
    CFMutableSetRef _commonModes;    //这个Set主要CommonModes类型的名字，能保证不重复名字的mode
    CFMutableSetRef _commonModeItems; //commonModes里可以插入不同Mode Items（可包含Source、Timer和Observer类型）
    CFRunLoopModeRef _currentMode; //当前执行的mode
    CFMutableSetRef _modes; //runLoop的modes，每次执行循环只能执行其中的一个
    ...
};

struct __CFRunLoopMode {
    ...
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */ //对象锁，保证线程安全
    ...
    CFMutableSetRef _sources0; //source0类型的CFRunLoopSource的set
    CFMutableSetRef _sources1; //source1类型的CFRunLoopSource的set
    CFMutableArrayRef _observers; //observer数组
    CFMutableArrayRef _timers; //timer数组
    ...
};
```
由结构体定义可知`__CFRunLoop`内部有_commonModes、_commonModeItems和_modes2个SET集合：
+ `commonModes`（SET类型）只存储Mode的别名；
+ `commonModeItems`（SET类型）存储的`__CFRunLoopMode`的结构体；
	`__CFRunLoopMode`内部定义了这个4个集合类的实例：
	* source0（SET类型）：用来存储`__CFRunLoopSource`的结构体对象；
	* source1（SET类型）：用来存储`__CFRunLoopSource`的结构体对象，该对象在runLoop睡眠时被唤醒；
	* timer（Array类型）：用来存储`__CFRunLoopTimer`的结构体对象，   
	* observer（Array类型）：用来存储`__CFRunLoopObserver`的结构体对象；   
+ `modes`：runLoop执行的mode，每次循环只能执行其中的一个Mode。

说明一个`__CFRunLoop`中可以插入多个Mode，每个Mode又可以包含多个source事件、多个timer实例和多个观察者(observer)对象，Source、Timer和Observer统称Mode Item。

#### 至于source为何选用set类型，而timer与observer选用的array类型？
猜测：`SET集合`既能保证数据的唯一性（hash值比较），又能快速索引（比如NSObject cancel延时事件，需要快速准确得找到cancel的source这种场景）。
## \_\_CFRunLoopSource & \_\_CFRunLoopTimer & \_\_CFRunLoopObserver
```objc
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order; /* immutable */
    CFMutableBagRef _runLoops;
    union { //联合体能保证在同一个时间内不会同时使用version0和version1,2个数据结构体中只有一个会存在，符合runLoop的设计
        CFRunLoopSourceContext version0; /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1; /* immutable, except invalidation */
    } _context;
};

typedef struct {
    CFIndex version;
    void * info;
    const void *(*retain)(const void *info);
    void (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
    Boolean (*equal)(const void *info1, const void *info2);
    CFHashCode (*hash)(const void *info);
    //定时处理，适用`performSelector:withObject:afterDelay:`
    void (*schedule)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    //取消处理，适用`cancelPreviousPerformRequestsWithTarget:selector:object:` 
    void (*cancel)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void (*perform)(void *info);
} CFRunLoopSourceContext; //iOS的performSelector系列方法会用这个Context存储函数指针，并且可以取消
typedef struct {
    CFIndex version;
    void * info;
    const void *(*retain)(const void *info);
    void (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
    Boolean (*equal)(const void *info1, const void *info2);
    CFHashCode (*hash)(const void *info);
    mach_port_t (*getPort)(void *info);
    void * (*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
} CFRunLoopSourceContext1;
```
`__CFRunLoopSource`是事件产生的地方，共在2种类型：`Source0`和`Source1`，struct内部通过一个union保证同一个Source对象不会同时包含version0和version1; 
* `Source0`只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 `CFRunLoopSourceSignal(rlms)`方法将这个Source标记为待处理，然后手动调用`CFRunLoopWakeUp(rl)`方法来唤醒RunLoop，让其处理这个事件。
* `Source1`包含了一个`mach_port`和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种类型的Source能主动唤醒RunLoop的线程。
```objc
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```
`__CFRunLoopTimer`：是基于时间的触发器，它和`NSTimer`是toll-free bridged的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到RunLoop时，RunLoop会注册对应的时间点，当时间点到达时，RunLoop会被唤醒以执行那个回调。
```objc
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};
```
`__CFRunLoopObserver`：是观察者，每个Observer都包含了一个回调（函数指针），当 RunLoop的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点有以下几个：
```objc
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```
## RunLoop的设计
CFRunLoop的设计是一个循环处理多个类型的事件处理（Timers，Source、Observer等）模型。
工作流程如下图：
<img src="http://ipa-download.qiniudn.com/AAA622AD-DBF3-4D35-9086-333DDF9FE9AE.png" width="647"/>

RunLoop中虽然包含了多个Mode，但一次循环只能执行其中一个Mode，如果要切换Mode，必须等上一次的循环结束。

苹果公开提供的Mode有两个：kCFRunLoopDefaultMode（NSDefaultRunLoopMode）和 UITrackingRunLoopMode，你可以用这两个`ModeName`来操作其对应的Mode。

`注意`：这里有个概念叫`CommonModes`：一个Mode可以将自己标记为"Common"属性（通过将其 ModeName添加到RunLoop的"commonModes"中）。每当RunLoop的内容发生变化时，RunLoop都会自动将_commonModeItems里的Source/Observer/Timer同步到具有"Common"标记的所有Mode里。我们可以通过`NSRunLoopCommonModes`关键字操作CommonModeItems，这个也是苹果开放的API。

我们可以通过`addTimer:forMode:`方法把NSTimer（NSTimer默认是在NSRunLoopDefaultMode里）加载到CommonModeItems里，保证NSTimer的触发在UITrackingRunLoopMode里也能触发。

## RunLoop核心方法
源码中的核心方法整理如下：
```objc
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (modeName == NULL || modeName == kCFRunLoopCommonModes || CFEqual(modeName, kCFRunLoopCommonModes)) { //如果运行到了CurrentMode是CommonModes，RunLoop就退出了。
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            CFLog(kCFLogLevelError, CFSTR("invalid mode '%@' provided to CFRunLoopRunSpecific - break on _CFRunLoopError_RunCalledWithInvalidMode to debug. This message will only appear once per execution."), modeName);
            _CFRunLoopError_RunCalledWithInvalidMode();
        });
        return kCFRunLoopRunFinished;
    }
    if (__CFRunLoopIsDeallocating(rl))
        return kCFRunLoopRunFinished;
    
    __CFRunLoopLock(rl);
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode))
    {
        Boolean did = false; //这里的did是没写完么?永远是false
        if (currentMode)
            __CFRunLoopModeUnlock(currentMode); //这里返回的CFRunLoopMode是加锁的，所以在这里要进行一次解锁
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
    
    if (currentMode->_observerMask & kCFRunLoopEntry )
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode); //runLoop循环处理方法开始
    if (currentMode->_observerMask & kCFRunLoopExit )
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}

/* rl, rlm are locked on entrance and exit */
//这个方法相关长，精简出重要代码
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
   ...
    
    do {
         __CFRunLoopUnsetIgnoreWakeUps(rl);
        //通知Observers，处理Timers
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        //通知Observers，处理Sources
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        //处理加入的Block
        __CFRunLoopDoBlocks(rl, rlm);
        
        //处理Source0的事件
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        
        didDispatchPortLastTime = false;
        //通知Observers 即将睡眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        __CFRunLoopSetSleeping(rl);
        ...
        
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        // user callouts now OK again
        __CFRunLoopUnsetSleeping(rl);
        
        //通知Observer 睡眠唤醒
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        CFRUNLOOP_WAKEUP_FOR_SOURCE();
        // Despite the name, this works for windows handles as well
        CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
                mach_msg_header_t *reply = NULL;
                //处理Source1的事件
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop; 
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            }
         ...
         
        __CFRunLoopDoBlocks(rl, rlm); //睡眠方法唤醒以后处理加入了的Block事件
        
        if (sourceHandledThisLoop && stopAfterHandle) {
            retVal = kCFRunLoopRunHandledSource; //
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut; //判断是否超时
        } else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            retVal = kCFRunLoopRunFinished;
        }
    } while (0 == retVal);
    return retVal;
}
``` 

## 线程安全 
CFRunLoop是线程安全的，CFRunLopp是纯C的API封装。从源代码定义的各个结构体对象中都会包含pthread_mutex的成员变量。
pthread_mutex被实例成了`递归锁`，递归锁能保证在同一个线程里被多次调用不会造成锁等待的情况，但在多线程中能保证数据同步而存在锁等待的效果。
```c++
CF_INLINE void __CFRunLoopLockInit(pthread_mutex_t *lock) {
    pthread_mutexattr_t mattr;
    pthread_mutexattr_init(&mattr);
    pthread_mutexattr_settype(&mattr, PTHREAD_MUTEX_RECURSIVE); //初始化成递归锁
    int32_t mret = pthread_mutex_init(lock, &mattr);
    pthread_mutexattr_destroy(&mattr);
    if (0 != mret) {
    }
}
```
## RunLoop实现的功能
`广告`：更多代码[demo](https://github.com/openboy2012/DDCategory)可以进入我的[github](https://github.com/openboy2012/DDCategory)进行下载查看运行结果，基本上每行关键代码都有详细的注释。本文中出现的代码都在这个项目下面，请结合项目代码阅读本文，效果更佳。
### 事件响应 & 手势识别 & AutoReleasePool & UI更新 
当我们启动一个app的时候，点击暂停线程我们会看到这样一个堆栈关系：
<img src="http://ipa-download.qiniudn.com/0FFEFFA1-F7A0-4802-B54F-19C00A66EE27.png" width="410"/>
先来看下主线程的`po CFRunLoopGetCurrent()`，整理以后如下：
```objc
CFRunLoop
{ 
   wakeup port = 0x1e03, stopped = false, ignoreWakeUps = false, 
   current mode = kCFRunLoopDefaultMode,
   common modes = 
   {
        UITrackingRunLoopMode //CFString Mode名称
        kCFRunLoopDefaultMode //CFString
   },
   common mode items = 
   {
        
        0 : callout = PurpleEventSignalCallback
        1 : ...
        2 : callout = __handleHIDEventFetcherDrain //释放HIDEvent对象的callback（source0）
        5 : ...
        6 : callout = _wrapRunLoopWithAutoreleasePoolHandler //AutoReleasePool最高优先级处理 (observer)
        7 : callout = _wrapRunLoopWithAutoreleasePoolHandler //AutoReleasePool最低优先级处理 (observer)
        8 : callout = _afterCACommitHandler //监听CATransaction，刷新UI（observer）
        9 : callout = _beforeCACommitHandler //监听CATransaction
        10 : callout = _UIGestureRecognizerUpdateObserver //手势检测回调
        11 : callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv 
        13 : callout = FBSSerialQueueRunLoopSourceHandler //Front Board Services
        16 : callout = __handleEventQueue //用户事件回调
        19 : ...
        21 : callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ 
        22 : callout = PurpleEventCallback
   },
   modes = 
   {
        2 : CFRunLoopMode
        {
            name = UITrackingRunLoopMode,
            sources0 = 
            {
                0 : callout = PurpleEventSignalCallback
                2 : callout = FBSSerialQueueRunLoopSourceHandler 
                4 : callout = __handleEventQueue 
                5 : callout = __handleHIDEventFetcherDrain
            },
            sources1 = 
            {
                0 : callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ 
                3 : ...
                4 : ...
                5 : callout = PurpleEventCallback
                6 : ...
            },
            observers = ( 
                callout = _wrapRunLoopWithAutoreleasePoolHandler,
                callout = _UIGestureRecognizerUpdateObserver,
                callout = _beforeCACommitHandler,
                callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv,
                callout = _afterCACommitHandler,
                callout = _wrapRunLoopWithAutoreleasePoolHandler 
            ),
            timers = (null),
        },

        3 : CFRunLoopMode
        {
            name = GSEventReceiveRunLoopMode,
            sources0 = 
            {
                callout = PurpleEventSignalCallback
            },
            sources1 = 
            {
                callout = PurpleEventCallback
            },
            observers = (null),
            timers = (null),
        },

        4 : CFRunLoopMode 
        {
            name = kCFRunLoopDefaultMode,  
            sources0 =  
            {   
                
                0 : callout = PurpleEventSignalCallback
                2 : callout = FBSSerialQueueRunLoopSourceHandler
                4 : callout = __handleEventQueue
                5 : callout = __handleHIDEventFetcherDrain
            },
            sources1 = 
            {
                0 : callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ 
                3 : ...
                4 : ...
                5 : callout = PurpleEventCallback
                6 : ...
            },
            observers = (
                callout = _wrapRunLoopWithAutoreleasePoolHandler,
                callout = _UIGestureRecognizerUpdateObserver,
                callout = _beforeCACommitHandler,
                callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv,
                callout = _afterCACommitHandler,
                callout = _wrapRunLoopWithAutoreleasePoolHandler
            ),
            timers = (null),
        },

        5 : CFRunLoopMode
        {
            name = UIInitializationRunLoopMode, 
            sources0 = 
            {
                callout = FBSSerialQueueRunLoopSourceHandler 
            },
            sources1 = (null)
            observers = (
            callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv
            ),
            timers = (null),
            
        },

        6 : CFRunLoopMode 
        {
            name = kCFRunLoopCommonModes,
            sources0 = (null),
            sources1 = (null),
            observers = (null),
            timers = (null),
        },
   }
}
```
主线程默认创建的CFRunLoop包含了5个Mode类型：
* `UITrackingRunLoopMode`：该Mode能确保UIScrollView的滚动流畅性，UI滚动时切换到的是这个Mode，而NSTimer是在defaultMode里，存在NSTimer不触发的问题，所以我们才把NSTimer加到commonMode里以后在UITrackingRunLoopMode也会触发timer了。
* `GSEventReceiveRunLoopMode`：
* `kCFRunLoopDefaultMode`：默认Mode类型，用户事件一般都会被加载在这个Mode的Source0处。
* `UIInitializationRunLoopMode`：初始化app时的过渡Mode。
* `kCFRunLoopCommonModes`：公共Mode类型，一般情况下是空的。

在CommonModeItems里注册了以下可识别的Mode Items:
`Observer类型`：
`_wrapRunLoopWithAutoreleasePoolHandler`： //AutoReleasePool最高优先级处理 (observer) 
`_wrapRunLoopWithAutoreleasePoolHandler`： //AutoReleasePool最低优先级处理 (observer)
以上2个观察者与内存管理有关。
`_afterCACommitHandler`： //监听CATransaction，刷新UI（observer）
`_beforeCACommitHandler`： //监听CATransaction
以上2个观察者有界面刷新有关。
`_UIGestureRecognizerUpdateObserver`： //手势检测回调 (observer)，手势变化时都会被这个观察者捕获。mach_msg_trap状态时也需要被RunLoop唤醒以后处理。

`Source类型`：
`_handleHIDEventFetcherDrain`： //释放IOHIDEvent对象的callback（source0)，所以有IOHIDEvent事件的位置（通常是唤醒RunLoop的位置）都会有这个回调方法。
`_handleEventQueue`： //用户事件回调(source0)，一般的`addTarget: action: forControlEvents:`方法都会加在source0，并由`_handleEventQueue`执行。

再看下事件点击线程6的`po CFRunLoopGetCurrent()`，整理以后如下：
```objc
CFRunLoop
{
    wakeup port = 0x3403, stopped = false, ignoreWakeUps = false, 
    current mode = kCFRunLoopDefaultMode,
    common modes = 
    {  
        contents = "kCFRunLoopDefaultMode"
    },
    common mode items = 
    {
        callout = _UIEventFetcherTriggerHandOff
        callout = __IOHIDEventSystemClientAvailabilityCallback 
        callout = __IOMIGMachPortPortCallback
        callout = __IOHIDEventSystemClientQueueCallback
    },
    modes = 
    {
        CFRunLoopMode
        {
            name = kCFRunLoopDefaultMode
            sources0 = 
            {
                callout = _UIEventFetcherTriggerHandOff //标记UIEvent事件待处理的回调方法
            } ,
            sources1 = 
            { 
                callout = __IOHIDEventSystemClientQueueCallback //屏幕触摸事件回调
                callout = __IOHIDEventSystemClientAvailabilityCallback //待研究
                callout = __IOMIGMachPortPortCallback //待研究
            },
            observers = (null),
            timers = (null)，
        },
    }
}
```
然后我们在Symbolic BreakPoint中添加一个`__IOHIDEventSystemClientQueueCallback`和`_UIEventFetcherTriggerHandOff`断点（该处理需要阅读者自行处理，属于Xcode的配置）。再触摸屏幕时，我们可以看到断点在以下2个堆栈关系图里：
<img src="http://ipa-download.qiniudn.com/873A4049-C316-44A9-8BE7-14CE03035923.png" width="560"/>
<img src="http://ipa-download.qiniudn.com/1F89CF6C-237F-46FC-8E16-34ECDE208F20.png" width="537"/>
流程分析：当app启动默认会启动主线程的RunLoopM(M表示主线程)，它在处理完一些事件以后进入mach_msg_trap状态，同时开启了一个事件处理线程6，在RunLoop6（6表示线程6）的source1添加了监听`__IOHIDEventSystemClientQueueCallback`方法，source0里添加了`_UIEventFetcherTriggerHandOff`方法，用户不触摸屏幕时该RunLoop也会进入mach_msg_trap状态。当用户触摸屏幕以后，事件线程6的RunLoop6最先被唤醒后执行source1里的`__IOHIDEventSystemClientQueueCallback`方法来唤醒主线程的RunLoopM，同时RunLoop6的source0里的`_UIEventFetcherTriggerHandOff`方法会把主线程的RunLoopM里的source0里的用户事件标记为待处理状态，紧接着唤醒的主线程RunLoopM会处理source0里的用户事件。

### performSelector:object:afterDelay:
performSelector延时系列方法也需要RunLoop处理，在内部会创建一个Timer计时来延时执行，所以RunLoop必须是在运行状态才成处理成功。如果不在主线程，需要开发者启动RunLoop来让方法生效。

参考资料：
[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/) --ibireme大神的深入专研精神真的令人倾佩。
[NSRunLoop](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/uid/TP40003725)
[Event Loop](https://en.wikipedia.org/wiki/Event_loop)
[Mach(kernel)](https://en.wikipedia.org/wiki/Mach_(kernel)



