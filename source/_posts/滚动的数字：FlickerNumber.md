title: "[源码]滚动的数字：FlickerNumber"
date: 2015-03-02 22:50:32
tags: 
- iOS开发
- 开源代码
id: 20
categories: 
---
##起因
最近的项目中要求实现支付宝的滚动数字的效果，查找到了一些第三方的代码，但是效果很不理想。  

于是准备自己来实现该效果，在学习github大牛的代码过程中，看到很多大牛都是用Category来扩展实现某些功能，  
例如: SDWebImage中的UIImageView的Category、AFNetworking中的UIButton的Category等。  
于是我也使用Category的方法来实现该功能。  
<!--more-->

##思路
数字从某个起点数字跳转到目标数字，定时的去刷新当前累加的数字，达到一个数字变化的效果。  

##实现
新建UILabel的Category: “UILable+FlickerNumber“  

``` objc
新建方法:
 *  flicker number only a number variable
 *
 *  @param number flicker number
 */
- (void)dd_setNumber:(NSNumber *)number;
/**
 *  flicker number in duration
 *
 *  @param number   flicker number
 *  @param duration duration time
 */
- (void)dd_setNumber:(NSNumber *)number
            duration:(NSTimeInterval)duration;
/**
 *  flicker number with format
 *
 *  @param number    flicker number
 *  @param formatStr format string
 */
- (void)dd_setNumber:(NSNumber *)number
              format:(NSString *)formatStr;
/**
 *  flicker number with attributes
 *
 *  @param number flicker number
 *  @param attrs  text attributes
 */
- (void)dd_setNumber:(NSNumber *)number
          attributes:(id)attrs;
/**
 *  flicker number with format in duration
 *
 *  @param number    flicker number
 *  @param duration  duration time
 *  @param formatStr format string
 */
- (void)dd_setNumber:(NSNumber *)number
            duration:(NSTimeInterval)duration
              format:(NSString *)formatStr;
/**
 *  flicker number with attribute in duration
 *
 *  @param number   flicker number
 *  @param duration duration time
 *  @param attrs    text attributes
 */
- (void)dd_setNumber:(NSNumber *)number
            duration:(NSTimeInterval)duration
          attributes:(id)attrs;  
/**
 *  flicker number method
 *
 *  @param number   flicker number
 *  @param duration duration time
 *  @param format   format string
 *  @param attri    text attribute
 */
- (void)dd_setNumber:(NSNumber *)number  
            duration:(NSTimeInterval)duration  
              format:(NSString *)formatStr  
          attributes:(id)attrs;  
```
这些方法满足了时长、格式化、NSMutableAttributeString的要求；

##更多

代码获得：  
[https://github.com/openboy2012/FlickerNumber](https://github.com/openboy2012/FlickerNumber)  
CocoaPods方式：  
`pod search 'FlickerNumber’`


