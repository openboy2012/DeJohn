title: "[源码]iOS快速建模之DDModel"
date: 2015-05-05 07:06:30
categories: 
tags: 
- 开源代码
- iOS开发
---

## DDModel概要
不知不觉我已经毕业快4年了，iOS开发也做了4年多的时间了，也有了一定的经验积累，这次我将着重介绍我封装的模型基类--[DDModel](https://github.com/openboy2012/DDModel)。  
DDModel封装了SQLite、HTTP以及JSON/XML的ORM特性，能快速搭建一个具有本地持久化，快速获取HTTP请求数据以及NSDictionary ORM到模型的模型层工具类，你只要根据自己的业务建模，极大地提高了开发的效率，把更多的精力放在UI的编写中去。  
<!--more-->
## DDModel介绍
* DDModel继承了[SQLitePersistentObject](https://github.com/openboy2012/DDSQLiteKit.git)，这样就快速集成了SQLite存储ORM到对象的过程，之前的[一篇文章](http://www.dejohndong.com/2015/05/04/iOS快速建模之DDModel/#more)有介绍；
* DDModel封装了基于[AFNetworking](https://github.com/AFNetworking/AFNetworking)的HTTP请求，简化了大部分开发者把http请求放在Controller层的操作，起到了解耦合的作用；  
* DDModel封装JSON/XML到Model的功能，使JSON/XML ORM到对象模型。使用到的第三方库分别是[JTObjectMapping](https://github.com/jamztang/JTObjectMapping)和[XMLDictionary](https://github.com/nicklockwood/XMLDictionary)；  
* DDModel也封装了HUD的功能，使用了[MBProgressHUD](https://github.com/jdg/MBProgressHUD)，这样你再也不必为HUD烦恼了；  
* DDModel支持基于SQLite的Cache功能；  
* DDModel支持RESTfulAPI。  

## 如何获取代码？

* 方法1：通过CocoaPods安装DDModel:
`pod search ‘DDModel’`, 然后在你的Podfile中添加最新的版本`pod ‘DDModel’, ‘~> 0.4’`,这是最快捷的方法，也是我强烈推荐的方法

* 方法2：通过github的代码仓库获取[DDModel](https://github.com/openboy2012/DDModel);  
你可以把该项目中的`DDModel/Classes`目录下的所有文件拷贝到你的项目里，然后再把DDModel依赖的第三方库：[AFNetworking](https://github.com/AFNetworking/AFNetworking)、[XMLDictionary](https://github.com/nicklockwood/XMLDictionary)、[JTObjectMapping](https://github.com/jamztang/JTObjectMapping)、[SQLitePersistentObject](https://github.com/openboy2012/DDSQLiteKit)、[MBProgressHUD](https://github.com/jdg/MBProgressHUD)都要拷贝到你的项目。

## 如何使用DDModel?
### DDModelHTTPClient
参考Demo项目，你可以在你的AppDelegate里加入以下代码来启动一个DDModelHttpClient:
``` objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.

    [DDModelHttpClient startWithURL:@“https://api.app.net/“ delegate:self]; // initialzie a DDModelHttpClient

    return YES;
}
```
这样你就启动了一个DDModelHTTPClient了，你可以通过DDModelHttpClientDelegate
``` objc
@protocol DDHttpClientDelegate <NSObject>

@optional
/**
 *  Parameter encode method if you should encode the parameter in your HTTP client
 *
 *  @param params original parameters
 *
 *  @return endcoded parameters
 */
- (NSDictionary *)encodeParameters:(NSDictionary *)params;

/**
 *  Response String decode methods in you HTTP client
 *
 *  @param responseString origin responseString
 *
 *  @return new responseString
 */
- (NSString *)decodeResponseString:(NSString *)responseString;

/**
 *  Check the response values is an avaliable value.
    e.g. You will sign in an account but you press a wrong username/password, server will response a error for you, you can catch them use this protocol methods and handle this error exception.
 *
 *  @param values  should check value
 *  @param failure failure block
 *
 *  @return true or false
 */
- (BOOL)checkResponseValueAvaliable:(NSDictionary *)values failure:(DDResponseFailureBlock)failure;

@end  

```
以上这些方法来定制自己的业务逻辑。
当然你可以在DDModelHTTPClient里使用AFNetworking里的所有功能。

### DDModel
然后你就可以根据自己的业务创建各种继承于DDModel的模型了。

举例：  
 
``` objc
@interface User : DDModel

@property (nonatomic, strong) NSNumber *id;
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *avatarImageURLString;

@end

@interface Post : DDModel

@property (nonatomic, copy) NSString *text;
@property (nonatomic, strong) NSNumber *id;
@property (nonatomic, strong) User *user;


+ (void)getPostList:(id)params
           parentVC:(id)viewController
            showHUD:(BOOL)show
            success:(DDResponseSuccessBlock)success
            failure:(DDResponseFailureBlock)failure;

@end

#import “Post.h”

@implementation Post

/*
 * 对象解析的节点实现
 */
+ (NSString *)parseNode{
    return @“data”;
}

/* jsonMapping 实现
 * 使用场景： 
 * 1.返回数据与数据模型不一致时通过映射对应赋值,举例: @{@“id”:@“userId”} 返回数据中的id value 会赋值给数据模型的userId
 * 2.对象的嵌套关系，如本例里Post对象嵌套了User对象
 */
+ (NSDictionary *)parseMappings{
    /**
     *  in [mappingWithKey:mappings:] method, key is your defined property key;
     *  在’mappingWithKey:mapping:’中的key值是您定义的属性名名称,所以JSON字符串中的Value会映射给该属性字段；
     */
    id userHandler = [User mappingWithKey:@“user” mapping:[User parseMappings]];
    /**
     *  this ‘user’ key is the JSON String’s Key
     *  这个字典中的’user’ key值就是 JSON 字符串中的 user key;
     */
    NSDictionary *jsonMappings = @{@“user”:userHandler};
    
    /**
     *  所以整个JSON的映射关系就是把JSON字符串的User内容映射给我定义属性的user属性里，内部递归的关系按照user的parseMapping执行
     */
    return jsonMappings;
}

+ (void)getPostList:(id)params
           parentVC:(id)viewController
            showHUD:(BOOL)show
            success:(DDResponseSuccessBlock)success
            failure:(DDResponseFailureBlock)failure
{
    [[self class] get:@“stream/0/posts/stream/global” params:params showHUD:show parentViewController:viewController success:success failure:failure];
}


@end

@implementation User

+ (NSDictionary *)parseMappings{
    //支持keyPath的方式进行映射对象，可以随意重构数据对象
    return @{@“avatar_image.url”:@“avatarImageURLString”};
}

@end
```

你可以将更多的方法封装在该派生的模型里。

DDModel同时也支持从数据缓存中获取结果：
``` objc

/**
 *  Get json data first from db cache then from http server by HTTP GET Mehod.
 *
 *  @param path           HTTP Path
 *  @param params         GET Paramtters
 *  @param show           is show the HUD on the view
 *  @param viewController parentViewController
 *  @param dbResult       db cache result block
 *  @param success        success block
 *  @param failure        failre block
 */
+ (void)get:(NSString *)path
     params:(id)params
    showHUD:(BOOL)show
parentViewController:(id)viewController
  dbSuccess:(DDSQLiteBlock)dbResult
    success:(DDResponseSuccessBlock)success
    failure:(DDResponseFailureBlock)failure;

/**
 *  Get json data first from db cache then from http server by HTTP POST Mehod.
 *
 *  @param path           HTTP Path
 *  @param params         GET Paramtters
 *  @param show           is show the HUD on the view
 *  @param viewController parentViewController
 *  @param dbResult       db cache result block
 *  @param success        success block
 *  @param failure        failre block
 *
 */
+ (void)post:(NSString *)path
      params:(id)params
     showHUD:(BOOL)show
parentViewController:(id)viewController
   dbSuccess:(DDSQLiteBlock)dbResult
     success:(DDResponseSuccessBlock)success
     failure:(DDResponseFailureBlock)failure;
```
## 总结
DDModel可以简化你的开发工作，把更多的精力放在UI的编写上。


