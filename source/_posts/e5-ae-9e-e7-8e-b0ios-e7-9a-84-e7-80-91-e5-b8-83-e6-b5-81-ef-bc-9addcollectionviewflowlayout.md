title: “[iOS开发源码]实现iOS的瀑布流：DDCollectionViewFlowLayout"
date: 2015-02-25 22:01:32
tags: iOS开发
id: 5
categories: iOS开发
---
### 起因
前段时间一直在做iOS客户端的64位适配，所以把开发项目设置成了最低系统要求为iOS6.0。空闲之余，准备把之前用UIScrollView实现的瀑布流用UICollectionView重新实现一下。  
于是DDCollectionViewFlowLayout就这样诞生了。  
### 学习步骤
在学习UICollectionView的过程中,首先肯定是查阅苹果的官方文档[UICollectionView](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UICollectionView_class/index.html#//apple_ref/swift/cl/UICollectionView)。  
了解UICollectionView的基本信息以后得知要想实现瀑布流的效果必须使用[UICollectionViewLayout](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UICollectionViewLayout_class/index.html#//apple_ref/doc/c_ref/UICollectionViewLayout),继续参考苹果官方文档了解[UICollectionViewFlowLayout](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UICollectionViewFlowLayout_class/index.html#//apple_ref/occ/cl/UICollectionViewFlowLayout)必须实现的方法和生命周期。   
了解过UICollectionViewFlowLayout的Protocol方法以后，就可以着手写自己的代码了。首先 DDCollectionViewFlowLayout 继承了UICollectionViewFlowLayout,只要重载以下方法
 ```
 - (void)prepareLayout;  
 - (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect;   
 - (UICollectionViewLayoutAtttes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath;  
 ```
就可以实现瀑布流的效果。  
### 怎么获得代码？
可以直接通过下面的链接在github中获取：  
[https://github.com/openboy2012/DDCollectionViewFlowLayout](https://github.com/openboy2012/DDCollectionViewFlowLayout)   
当然你也可以在CocoaPods中搜索：  
` pod search ‘DDCollectioViewFlowLayout’ `
效果图:
![effect1](http://ipa-download.qiniudn.com/loadingmore.gif)  
![effect2](http://ipa-download.qiniudn.com/waterfall.gif)  
### 怎么使用？
```
如果你只想简单应用，导入DDCollectionViewFlowLayout以后实现以下代码：  

DDCollectionViewFlowLayout *layout = [[DDCollectionViewFlowLayout alloc] init];  
layout.delegate = self;  
[self.collectionView setLayout:layout];  

然后要实现UICollectionViewDataSource 方法中的必需方法：  
// The cell that is returned must be retrieved from a call to -dequeueReusableCellWithReuseIdentifier:forIndexPath:  
 - (NSInteger)collectionView:(UICollectionView *)collectionView 
      numberOfItemsInSection:(NSInteger)section;
 - (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView 
                   cellForItemAtIndexPath:(NSIndexPath *)indexPath; 

 我只在DDCollectionViewFlowLayout中新增了一个必需实现的delegate方法：  

 - (NSInteger)collectionView:(UICollectionView *)collectionView 
                      layout:(DDCollectionViewFlowLayout *)layout
    numberOfColumnsInSection:(NSInteger)section;  

因为DDCollectionViewFlowLayout继承了UICollectionViewFlowLayout，  
所以你可以选择性地实现UICollectionViewFlowLayoutDelegate中所有非必需delegate方法，  
例如：  

 - (CGSize)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout*)collectionViewLayout
   sizeForItemAtIndexPath:(NSIndexPath *)indexPath;  
   
 - (UIEdgeInsets)collectionView:(UICollectionView *)collectionView 
                         layout:(UICollectionViewLayout*)collectionViewLayout 
         insetForSectionAtIndex:(NSInteger)section;  
         
 - (CGFloat)collectionView:(UICollectionView *)collectionView
                    layout:(UICollectionViewLayout*)collectionViewLayout 
       minimumLineSpacingForSectionAtIndex:(NSInteger)section;  
        
 - (CGFloat)collectionView:(UICollectionView *)collectionView 
                    layout:(UICollectionViewLayout*)collectionViewLayout
       minimumInteritemSpacingForSectionAtIndex:(NSInteger)section;  
        
 - (CGSize)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout*)collectionViewLayout 
       referenceSizeForHeaderInSection:(NSInteger)section;  
       
 - (CGSize)collectionView:(UICollectionView *)collectionView 
                   layout:(UICollectionViewLayout*)collectionViewLayout
       referenceSizeForFooterInSection:(NSInteger)section;  
   
```  
###更多
这些方法都适配了DDCollectionViewFlowLayout,让你有亲切的感觉。  
如有不懂可以参考Demo[https://github.com/openboy2012/DDCollectionViewFlowLayout](https://github.com/openboy2012/DDCollectionViewFlowLayout)

