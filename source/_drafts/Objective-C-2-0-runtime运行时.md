title: "Objective-C 2.0: runtime运行时"
tags: 
  - Objective-C
  - 越狱开发
categories:
---

最近正在消化自己一直使用的SQLitePersisentObject的大牛代码，虽然该代码写了已经很久，但内部还是有很多巧妙的写法值得我们学习的地方。今天我就先来讲解一些Objective-C中的runtime运行时。
<!--more-->

## 动态语言 VS 静态类型语言
大家熟知的动态语言有js、php、ruby、pythod等等，他们都属于脚本语言。  
而我们熟知的C、C++、Java和C#等语言都是静态类型语言。

## Objective-C 的动态特性
众所周知，Objective-C是款接近动态语言的语言。为什么说她是接近动态呢？就是因为runtime的存在。通过runtime我们能干很多事情，比如动态检测当前对象的所有属性的类型、名称和方法。

## runtime详解之runtime.h

``` objc

    var canvas = document.getElementById("canvas");

    var context = canvas.getContext("2d");
```

``` objc
@interface EzvizObject ：NSObject

@end
    EzvizOpenSDK *sdk = [[EzvizOpenSDK alloc] init];
    sdk.name = @"萤石SDK";
    sdk.version = @"v3.0.0.20150922";
    [sdk checkUpdate];
    [sdk runtime];
    - (void)tableView:(UITableView *)tableView numberRowsOfSection:(NSUInteger)section;

``` 


``` objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.

    [DDModelHttpClient startWithURL:@“https://api.app.net/“ delegate:self]; // initialzie a DDModelHttpClient

    return YES;
}
```
