title: "[源码]iOS数据持久化SQLitePersistentObject"
date: 2015-05-04 21:24:19
categories: 
tags:
- 开源代码
- iOS开发
---

##源码概要
iOS开发必然离不开的一个话题就是本地持久化，iOS本地持久化的方式有很多，如SQLite、CoreData等原生API，也有技术大牛对SQLite进行抽象封装以后产生的优秀SQLite插件--[FMDB](https://github.com/ccgus/fmdb)等，今天我向大家着重介绍一款被遗忘的SQLite ORM神器--[SQLitePersistentObject](https://github.com/openboy2012/DDSQLiteKit.git),我要在iOS中将ORM进行到底。
<!--more-->
##SQLitePersistentObject介绍
SQLitePersistentObject最初的作者是Jeff LaMarche，它支持Cocoa中大部分的数据类型（如：UIImage、NSString、NSNumber、NSData等等）直接存储进入SQLite，是一款相当强大的SQLite ORM工具库。只要你定义的模型继承于SQLitePersistentObject，你就可以通过它定义的私有方法对数据库进行模型的增删改查。  

当然他还有一些不支持的数据类型如void*、struct、union等，这些也是Jeff LaMarche希望我们这些后辈们能改进的。  

###维护中断
但是现在Jeff LaMarche已经不再维护这个工具类了，而且随着时间的推移，这个工具的bug也越来越多。  
  
我从2012年开始使用这个工具类，作为这个工具的收益者的我就接过这个接力棒来做SQLitePersistentObject的维护，我也在SQLitePersistentObject中添加了几个异步的方法，一定程度上提高了工具的稳定性和性能，降低了使用的成本，提高了开发效率。  
##如何获得代码？
最直接的方法就是从我的github中直接clone[DDSQLiteKit](https://github.com/openboy2012/DDSQLiteKit.git)这个Repository，这个代码仓库已经包含了Demo。你只要拷贝出项目中的SQLitePersistentObject文件下面的所有文件到你的项目里，就可以使用SQLitePersistent的所有功能了。  

当然我也把SQLitePersistent发布到了CocoaPods，直接通过以下代码搜索：  
`pod search ‘SQLitePersistentObject’`  

##如何使用?
首先你创建一个新的Class继承于SQLitePersistentObject，然后可以根据自己的业务定义你需要的属性名：  
例如：  
```
header file:
#import “SQLitePersistentObject.h”

@interface Device : SQLitePersistentObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *model;
@property (nonatomic, strong) NSNumber *price;

@end


implementation file:
#import “Device.h”

@implementation Device

@end
```
这样，这个Device类就包含了SQLitePersistentObject的所有功能了。

##核心代码介绍
我这边介绍一下我添加的一些异步方法：  
```
#pragma mark - DeJohn Dong Added Methods
/**
 *  Asynchronous add/update an object to db.
 */
- (void)save;

/**
 *  Asynchronous delete an object from db.
 */
- (void)asynDeleteObject;

/**
 *  Asynchronous delete an object and the cascade objects from db.
 */
- (void)asynDeleteObjectCascade:(BOOL)cascade;

/**
 *  Asynchronous Query the object list with criteria from db.
 *
 *  @param criteria criteria string
 *  @param result   result list
 */
+ (void)queryByCriteria:(NSString *)criteria result:(DBQueryResult)result;

/**
 *  Asynchronous Query the first object with criteria from db
 *
 *  @param criteria criteria string
 *  @param result   result object
 */
+ (void)queryFirstItemByCriteria:(NSString *)criteria result:(DBQueryResult)result;

/**
 *  Asynchronous Query all the objects from db
 *
 *  @param result result list
 */
+ (void)queryResult:(DBQueryResult)result;
```

这些方法都是基于线程安全的异步操作数据库，你可以放心的使用。

###注意事项
但这些方法不能和SQLitePersistentObject原始的同步方法混用。  

SQLitePersistentObject所有的原始方法都是线程同步的，需要你自己创建多线程来控制异步加载数据，这也是因为大家觉得SQLitePersistentObject不好的最主要原因了。

经过我的改进以后，我相信还是会有不少开发者会喜欢上这个库的。  

###强烈建议
所以我强烈建议使用SQLitePersistentObject的开发者都使用我扩展的异步方法。  
##总结
SQLitePersistentObject虽然是很古老的库，但他还是有很多的内容值得我们去学习的，本篇文章只是介绍了这个库的功能和使用，以后我会推出新的文章具体讲解其内部实现，敬请期待。

