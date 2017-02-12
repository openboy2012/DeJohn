title: "[源码]滚动的数字：FlickerNumber"
date: 2015-03-02 22:50:32
tags: 
- iOS开发
- 开源代码
id: 20
categories: 
---
## 起因
最近的项目中要求实现支付宝的滚动数字的效果，查找到了一些第三方的代码，但是效果很不理想。  

求人不如求己，那我就自己动手来实现该效果。在学习github大牛的代码过程中，看到很多大牛都是用Category来扩展实现某些功能，
例如: SDWebImage中的UIImageView的Category、AFNetworking中的UIButton的Category等。
  
那我也尝试着使用Category的方式来实现该数字滚动的效果。  
<!--more-->

## 滚动思路
众所周知，UIKit中的UILabel控件格外强大，在开启iOS的动效以后，UILabel上的内容变换都会产生动画效果。那我就可以用UILabel的这个特性来实现数字的滚动效果。

设计思路：让数字从某个起点数字累加同一个平均数直到大于或等于目标数字，每次累加的数字结果设置为这个UILabel的text值，这个过程会形成动画，正好达到一个数字变化的效果。 当然我们可以控制刷新的间隔。 

## 代码实现
新建UILabel的Category: `UILabel+FlickerNumber`

首先，数字滚动的动画是一个过渡过程，我需要一个中间变量。
我的第一反应是增加一个静态变量作为中间变量，但是细细想来，如果我有2个UILabel同时滚动数字，那么这个静态变量会贯穿这2个UILabel，这样滚动动画肯定会产生问题的。所以这里是不能使用静态变量来作为中间变量的。

然后我就想到了使用属性，类似UILabel的text属性一样，相当于我对UILabel的属性进行扩展。

使用Category增加属性必须要使用iOS的runtime的特性：
``` objc
#import <objc/runtime.h>

@interface UILabel ()

@property (nonatomic, strong, readwrite) NSNumber *flickerNumber;
@property (nonatomic, strong, readwrite, nullable) NSNumberFormatter *flickerNumberFormatter;
@property (nonatomic, strong, readwrite, nullable) NSTimer *currentTimer;

@end

@implementation UILabel (FlickerNumber)

//The intermediate number, it's private variable. Extend property use the runtime feature.
- (void)setFlickerNumber:(NSNumber *)flickerNumber {
    objc_setAssociatedObject(self, @selector(flickerNumber), flickerNumber, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSNumber *)flickerNumber {
    return objc_getAssociatedObject(self, _cmd);
}

...

```
这样我就完成了对UILabel扩展了一个`flickerNumber`的属性，因为我把这个属性设置在了extentsion里，所有该属性只能在这个类实现内部使用，别人在外部是看不到这个属性的。

依葫芦画瓢，再增加`flickerNumberFormatter`、`currentTimer`这2个属性：

``` objc 
...

//Flicker animation timer.
- (void)setCurrentTimer:(nullable NSTimer *)currentTimer {
    objc_setAssociatedObject(self, @selector(currentTimer), currentTimer, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (nullable NSTimer *)currentTimer {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setFlickerNumberFormatter:(nullable NSNumberFormatter *)flickerNumberFormatter {
    objc_setAssociatedObject(self, @selector(flickerNumberFormatter), flickerNumberFormatter, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (nullable NSNumberFormatter *)flickerNumberFormatter {
    return objc_getAssociatedObject(self, _cmd);
}

...

```
计时器`currentTimer`：用来控制时间间隔和传值，定时去刷新UILabel的数字text；
数字格式化输出`flickerNumberFormatter`：是用来格式化输出数字的。

扩展属性都声明好了，那就开始撰写实现代码了，先来一个核心代码：
``` objc 
- (void)fn_setNumber:(NSNumber *)number duration:(NSTimeInterval)duration format:(nullable NSString *)formatStr numberFormatter:(nullable NSNumberFormatter *)formatter attributes:(nullable id)attrs {
    /**
     *  check the number type
     */
    NSAssert([number isKindOfClass:[NSNumber class]], @"Number Type is not matched , exit");
    if(![number isKindOfClass:[NSNumber class]]) {
        self.text = [NSString stringWithFormat:@"%@",number];
        return;
    }
    
    /* limit duration is postive number and it is large than 0.3 , fixed the issue#1--https://github.com/openboy2012/FlickerNumber/issues/1 */
    duration = fabs(duration) < 0.3 ? 0.3 : fabs(duration);
    
    [self.currentTimer invalidate];
    self.currentTimer = nil;
    
    //initialize useinfo dict
    NSMutableDictionary *userInfo = [NSMutableDictionary dictionaryWithCapacity:0];
    
    if(formatStr)
        [userInfo setObject:formatStr forKey:DDFormatKey];
    
    [userInfo setObject:number forKey:DDResultNumberKey];
    
    //initialize variables
    long long beginNumber = 0;
    [userInfo setObject:@(beginNumber) forKey:DDBeginNumberKey];
    self.flickerNumber = @0;
    unsigned long long endNumber = [number unsignedLongLongValue];
    
    //get multiple if number is double type
    int multiple = [self multipleForNumber:number formatString:formatStr];
    if (multiple > 0)
        endNumber = [number doubleValue] * multiple;
    
    //check the number if out of bounds the unsigned int length
    if (endNumber >= INT64_MAX) {
        self.text = [NSString stringWithFormat:@"%@",number];
        return;
    }
    
    [userInfo setObject:@(multiple) forKey:DDMultipleKey];
    [userInfo setObject:@(endNumber) forKey:DDEndNumberKey];
    if ((endNumber * DDFrequency)/duration < 1) {
        duration = duration * 0.3;
    }
    [userInfo setObject:@((endNumber * DDFrequency)/duration) forKey:DDRangeIntegerKey];
    
    if(attrs)
        [userInfo setObject:attrs forKey:DDAttributeKey];
    
    self.flickerNumberFormatter = nil;
    if(formatter)
        self.flickerNumberFormatter = formatter;
    
    self.currentTimer = [NSTimer scheduledTimerWithTimeInterval:DDFrequency target:self selector:@selector(flickerAnimation:) userInfo:userInfo repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.currentTimer forMode:NSRunLoopCommonModes];
}
```
该方法中通过一个可变字典存储了起始数、目标数、`中间平均数`以及一些数字的扩展属性（MutableAtrributedString、format-string和number-formatter style），这个可变字典通过currentTimer的userInfo属性在函数中传递。

中间平均数是通过`目标数值 * 每秒帧数(默认值是1/30--即每秒闪动30次的标准) / 动画时长duration `计算获得。

滚动动画代码实现：
``` objc
/**
 *  Flicker number animation implemetation method.
 *
 *  @param timer  The schedule timer, the time interval decide the number flicker counts.
 */
- (void)flickerAnimation:(NSTimer *)timer {
    /**
     *  check the rangeNumber if more than 1.0, fixed the issue#2--https://github.com/openboy2012/FlickerNumber/issues/2
     */
    if ([timer.userInfo[DDRangeIntegerKey] floatValue] >= 1.0) {
        long long rangeInteger = [timer.userInfo[DDRangeIntegerKey] longLongValue];
        self.flickerNumber = @([self.flickerNumber longLongValue] + rangeInteger);
    } else {
        float rangeInteger = [timer.userInfo[DDRangeIntegerKey] floatValue];
        self.flickerNumber = @([self.flickerNumber floatValue] + rangeInteger);
    }
    
    
    int multiple = [timer.userInfo[DDMultipleKey] intValue];
    if(multiple > 0) {
        [self floatNumberHandler:timer andMultiple:multiple];
    }else {
        NSString *formatStr = timer.userInfo[DDFormatKey]?:(self.flickerNumberFormatter?@"%@":@"%.0f");
        self.text = [self finalString:@([self.flickerNumber longLongValue]) stringFormat:formatStr numberFormatter:self.flickerNumberFormatter];
        
        if(timer.userInfo[DDAttributeKey]){
            [self attributedHandler:timer.userInfo[DDAttributeKey]];
        }
        
        if([self.flickerNumber longLongValue] >= [timer.userInfo[DDEndNumberKey] longLongValue]){
            self.text = [self finalString:timer.userInfo[DDResultNumberKey] stringFormat:formatStr numberFormatter:self.flickerNumberFormatter];
            if(timer.userInfo[DDAttributeKey]){
                [self attributedHandler:timer.userInfo[DDAttributeKey]];
            }
            [timer invalidate];
        }
    }
}

```
代码是通过NSTimer的userInfo传值，处理了整型中间平均数和浮点型中间平均数两种情况，方法的核心是通过不断累加这个平均数来递增结果值，然后通过`finalString:stringFormat:numberFormatter:`方法输出到UILabel的text或者attributedText:
``` objc 

/**
 *  The final-string of each frame of flicker animation.
 *
 *  @param number    The result number.
 *  @param formatStr The string-format String.
 *  @param formatter The number-formatter style.
 *
 *  @return The final string.
 */
- (NSString *)finalString:(NSNumber *)number
             stringFormat:(NSString *)formatStr
          numberFormatter:(NSNumberFormatter *)formatter {
    NSString *finalString = nil;
    if (formatter) {
        NSAssert([formatStr rangeOfString:@"%@"].location != NSNotFound, @"The string format type is not matched. Please check your format type if it's not `%%@`. ");
        finalString = [NSString stringWithFormat:formatStr,[self stringFromNumber:number numberFormatter:formatter]];
    } else {
        NSAssert([formatStr rangeOfString:@"%@"].location == NSNotFound, @"The string format type is not matched. Please check your format type if it's `%%@`. ");
        //fixed the bug if use the `%d` format string.
        if ([formatStr rangeOfString:@"%d"].location == NSNotFound)
        {
            finalString = [NSString stringWithFormat:formatStr,[number doubleValue]];
        }
        else
        {
            finalString = [NSString stringWithFormat:formatStr,[number longLongValue]];
        }
    }
    return finalString;
}

```
注意：！！！这边格式化输出的时候我用2个断言来判断格式化输出的类型，如果格式化类型与输出的text不匹配，程序会Crash。 如果使用了numberFormatter就不能使用`%f、%d`等数字格式化，只能使用`%@`格式化输出text。相反，如果你想要用格式化数字，则不能用`%@`替代。

关于数字格式化以及字体颜色变化和字体大小不同，我是通过NSString的format特性和UILabel的NSMutableAtrributedString特性以及numberForamtter特性对输出的字符串进行输出格式的扩展：
``` objc
/**
 *  The attributed(s) text handle methods
 *
 *  @param attributes The attributed property, it's a attributed dictionary OR array of attributed dictionaries.
 */
- (void)addTextAttributes:(id)attributes {
    if ([attributes isKindOfClass:[NSDictionary class]]) {
        NSRange range = [attributes[DDDictRangeKey] rangeValue];
        [self addAttribute:attributes[DDDictArrtributeKey] range:range];
    } else if([attributes isKindOfClass:[NSArray class]]) {
        for (NSDictionary *attribute in attributes) {
            NSRange range = [attribute[DDDictRangeKey] rangeValue];
            [self addAttribute:attribute[DDDictArrtributeKey] range:range];
        }
    }
}

/**
 *  Add attributed property into the number text OR string-format text.
 *
 *  @param attri The attributed of the text
 *  @param range The range of the attributed property
 */
- (void)addAttribute:(NSDictionary *)attri
               range:(NSRange)range {
    NSMutableAttributedString *str = [[NSMutableAttributedString alloc] initWithAttributedString:self.attributedText];
    // handler the out range exception
    if(range.location + range.length <= str.length){
        [str addAttributes:attri range:range];
    }
    self.attributedText = str;
}

/**
 *  Get the number string from number-formatter style.
 *
 *  @param number    The result number.
 *  @param formattor The number-formatter style.
 *
 *  @return The number string.
 */
- (NSString *)stringFromNumber:(NSNumber *)number
               numberFormatter:(NSNumberFormatter *)formattor {
    if(!formattor) {
        formattor = [[NSNumberFormatter alloc] init];
        formattor.formatterBehavior = NSNumberFormatterBehavior10_4;
        formattor.numberStyle = NSNumberFormatterDecimalStyle;
    }
    return [formattor stringFromNumber:number];
}

```
FlickerNumber 使用的默认number-formatter style 是：
``` objc
/**
 *  Get the decimal style number as default number-formatter style.
 *
 *  @return The number-foramtter style.
 */
- (NSNumberFormatter *)defaultFormatter {
    NSNumberFormatter *formattor = [[NSNumberFormatter alloc] init];
    formattor.formatterBehavior = NSNumberFormatterBehavior10_4;
    formattor.numberStyle = NSNumberFormatterDecimalStyle;
    return formattor;
}
```

### 如何处理float类型的数据？
思路：把float类型 乘以(*) 小数点的位数的倍数变成整型数字，还是从0开始累加到目标整型数字，只是在输出text的时候再除以相应的倍数，到达滚动float类型的数字的效果。

获取Float类型数值倍数方法，这里只处理了最大小数位为6位的情况：
``` objc
/**
 *  Get muliple from number
 *
 *  @param number past number
 *
 *  @return mulitple
 */
- (int)multipleForNumber:(NSNumber *)number
            formatString:(NSString *)formatStr {
    if([formatStr rangeOfString:@"%@"].location == NSNotFound) {
        if([formatStr rangeOfString:@"%d"].location != NSNotFound) {
            return 0;
        }
        formatStr = [self regexNumberFormat:formatStr];
        NSString *formatNumberString = [NSString stringWithFormat:formatStr,[number floatValue]];
        if([formatNumberString rangeOfString:@"."].location != NSNotFound){
            NSUInteger length = [[formatNumberString substringFromIndex:[formatNumberString rangeOfString:@"."].location +1] length];
            float padding = log10f(length < 6 ? length:6);
            number = @([formatNumberString floatValue] + padding);
        }
    }
    
    NSString *str = [NSString stringWithFormat:@"%@",number];
    if([str rangeOfString:@"."].location != NSNotFound) {
        NSUInteger length = [[str substringFromIndex:[str rangeOfString:@"."].location +1] length];
        // Max Multiple is 6
        return  length >= 6 ? pow(10, 6): pow(10, (int)length);
    }
    return 0;
}

```

浮点型数字滚动动画输出：

``` objc

/**
 *  Float number handle method.
 *
 *  @param timer    timer
 *  @param multiple The number's multiple.
 */
- (void)floatNumberHandler:(NSTimer *)timer andMultiple:(int)multiple {
    NSString *formatStr = timer.userInfo[DDFormatKey] ?: (self.flickerNumberFormatter ? @"%@" : [NSString stringWithFormat:@"%%.%df",(int)log10(multiple)]);
    self.text = [self finalString:@([self.flickerNumber doubleValue]/multiple) stringFormat:formatStr numberFormatter:self.flickerNumberFormatter];
    if (timer.userInfo[DDAttributeKey]) {
        [self attributedHandler:timer.userInfo[DDAttributeKey]];
    }
    if ([self.flickerNumber longLongValue] >= [timer.userInfo[DDEndNumberKey] longLongValue]) {
        self.text = [self finalString:timer.userInfo[DDResultNumberKey] stringFormat:formatStr numberFormatter:self.flickerNumberFormatter];
        if(timer.userInfo[DDAttributeKey]){
            [self attributedHandler:timer.userInfo[DDAttributeKey]];
        }
        [timer invalidate];
    }
}

```
### 处理滚动动画被打断
``` objc 
[[NSRunLoop currentRunLoop] addTimer:self.currentTimer forMode:NSRunLoopCommonModes];
```
当UILabel处于一个滚动的视图中（如UICollectionView、UIScrollView等），在该视图滚动过程中，UILabel的滚动动画会被打断，使用改代码以后可以防止动画被打断。

## 更多

代码获得：  
[https://github.com/openboy2012/FlickerNumber](https://github.com/openboy2012/FlickerNumber)  
CocoaPods获得方式：  
`pod search 'FlickerNumber'`
方法的列表可以参考源码和github上的ReadMe.

### Swift适配
FlickerNumber的Swift版本也已经开发完成，完美兼容了XCode7、Swift 2.0的语法，源码已经提交至github。
CocoaPods获得方式：
`pod search 'FlickerNumber-Swift'`

## 总结
在写这个控件的时候，用到了很多技术点：runtime(运行时)、数学算法方法(logf()等算法)、`UILabel`的本身的一些特性（attributedText）、`NSNumberFormatter`的格式化输出和`NSString`的`stringWithFormat:`方法输出。我收获了不少iOS的技术知识和用法。但这些都只是一个程序员该有的基本能力。

真正的意义是在于我这个开源过程。

这个开源过程能告诉其他程序员如何开源自己写的代码，如何写出一个个简单的方法让调用者能简单应用而不用太关注具体的实现，同时也能让善于研究的开发者能理解代码的含义，这才是最困难的。代码的易读性和可维护性是至关重要的。

我相信随着我的不断提高，我写的代码会越来越好。

