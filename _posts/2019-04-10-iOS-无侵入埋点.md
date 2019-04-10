---
layout:     post
title:     无侵入埋点的加载
subtitle:  埋点
date:       2019-04-10
author:     祝化林
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    -   iOS
    - 埋点
    - runtime
---
### 什么是埋点？
埋点是一种了解用户行为，分析用户行为，提高用户体验的一种方式。
常见的解决方案有三种，代码埋点、可视化埋点、和无埋点三种。

* 代码埋点主要就是通过手写代码的方式来埋点，能很精确的在需要埋点的地方，添加代码。存在开发量大，后期难以维护的问题。
* 可视化埋点，将埋点的增加和修改可视化，提升了增加和维护埋点的体验。
* 无埋点又叫全埋点，埋点代码不会出现在业务代码中，容易管理和维护，缺点是成本高，解析复杂。

无埋点可以做到，埋点被统一维护，与业务代码解耦，满足大部分需求。

### 无埋点的实现
无埋点的实现主要是基于，runtime,在运行时，替换原有方法，实现埋点。

#### 建立替换方法的类
新建一个class,主要代码如下,
```
#import "SMHook.h"
#import <objc/runtime.h>
@implementation SMHook

+(void)hookClass:(Class)classObject fromSelector:(SEL)fromSelector toSelector:(SEL)toselector{
    Class class = classObject;
    // 得到被替换类的实例方法
    Method fromMethod = class_getInstanceMethod(class, fromSelector);
    // 得到替换类的实例方法
    Method toMethod = class_getInstanceMethod(class, toselector);
    if (class_addMethod(class, fromSelector, method_getImplementation(toMethod), method_getTypeEncoding(fromMethod))) {
        class_replaceMethod(class, toselector, method_getImplementation(fromMethod), method_getTypeEncoding(toMethod));
    }else{
        method_exchangeImplementations(fromMethod, toMethod);
    }
}
@end
```
其中主要的作用是交换两个IMP指针的实现
```
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2) 
```
IMP是具体方法的实现。

#### 页面的进入次数和停留时间
分析：获取页面的停留时间，主要hook住`UIViewController`的`viewWillAppear：` 和 `viewWillDisappear:`的两个方法，获取执行这两个方法的时间差，就可以获取停留时长
,给`UIViewController`建一个`Category`
实现方法
```
#import "ViewController+time.h"
#import "SMHook.h"
@implementation UIViewController (logger)

+(void)initialize{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SEL fromSelector = @selector(viewWillAppear:);
        SEL toSelector = @selector(hook_viewWillAppear:);
        [SMHook hookClass:self fromSelector:fromSelector toSelector:toSelector];
        
        SEL fromDisappear = @selector(viewWillDisappear:);
        SEL toDisappear = @selector(hook_viewWillDisAppear:);
        [SMHook hookClass:self fromSelector:fromDisappear toSelector:toDisappear];
    });
}
```
hook方法的实现
```
-(void)hook_viewWillAppear:(BOOL)animated{
    NSLog(@"hook viwe");
    // 进来的时间 根据具体的业务去加时间的统计
    [self comeIn];
    [self hook_viewWillAppear:animated];
}

-(void)hook_viewWillDisAppear:(BOOL)animated{
    // 出去的时间 统计方法根据具体的业务加
    [self comeOut];
    [self hook_viewWillDisAppear:animated];
}

```
**注意： 在实现了 `hook_viewWillAppear：`方法后，又调用了一遍 `[self hook_viewWillAppear:animated]`;这里的IMP执行的是`viewWillAppear:`方法，注意前面的**
```
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)
```
**方法，这样不会因为hook住`viewWillAppear:`的实现，而影响了业务代码中，`viewWillAppear:`内容的实现，并不会造成循环调用。**

#### 点击事件的HOOK
UIButton的点击事件的hook和`UIViewController`的`viewWillAppear:`的hook基本上一样，
这里面要hook的是:
>// send the action. the first method is called for the event and is a point at which you can observe or override behavior. it is called repeately by the second.


```
- (void)sendAction:(SEL)action to:(nullable id)target forEvent:(nullable UIEvent *)event;
```
这个方法，在点击的时候会被调用。
不是
```
- (void)addTarget:(nullable id)target action:(SEL)action forControlEvents:(UIControlEvents)controlEvents;
```
这个方法。

给UIButton新建Category,主要有三步

1. 替换方法
2. hook住方法的实现
3. 上传埋点信息

```
#import "UIButton+logger.h"
#import "SMHook.h"
#import <objc/runtime.h>
@implementation UIButton (logger)
+(void)initialize{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        SEL fromSelector = @selector(sendAction:to:forEvent:);
        SEL toSelector = @selector(hook_sendAction:to:forEvent:);
        [SMHook hookClass:self fromSelector:fromSelector toSelector:toSelector];
    });
}

-(void)hook_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event{
    [self insertAction:action to:target forEvent:event];
    [self hook_sendAction:action to:target forEvent:event];
    
}
-(void)insertAction:(SEL)action to:(id)target forEvent:(UIEvent*)event{
    NSString * actionName = NSStringFromSelector(action);
    NSString * targetName = NSStringFromClass([target class]);
    NSString *name = self.name;
    UIView *targetView = (UIView *)target;
    NSLog(@"%@",targetView);
    NSLog(@"button name == %@ actionName == %@, targetName == %@",name, actionName,targetName);
    // 缺少获取view_path的方法
    NSLog(@"viewPath %@",viewPath);
}

```
**这里有个关键，不仅仅是UIButton,而是任何你想Hook住对象的View_Path**

#### 为什么要View_path
每个控件需要有唯一的标识，这样才能对其埋点进行分析，如果一个视图下有多个UIButton,这样就不能仅仅通过 actionName 和 targetName 对齐分析
有一种解决办法是通过视图的层级结构，树状结构来分析
如下：
```
ViewController[0]/UIView[0]/UITableView[0]/UITableViewCell[0:2]/UIButton[0]
```
其中：

* 通过此标识可以在当前页面 view 树形结构中唯一的确定此元素。
* 标识的每一项由两部分组成：一是当前元素的 class 的字符串表示，二是当前元素在同级元素中的序号，自 0 开始计算。如当前第二个 UIImageView，则是 UIImageView1。
* 标识不同项之间以 / 拼接。
* 标识的最顶层是当前 view 所在的 ViewController。
* 对于 `UITableViewCell` 和 `UICollectionViewCell` 及类似的自定义组件，序号部分由两部分组成：`section` 和 `row`，并以: 拼接。
* 标识的最末端是当前被点击或触摸的元素

具体的实现：
```
+(NSString *)viewPath:(UIView *)currentView{
    __block NSString *viewPath = @"";
    
    for (UIView *view = currentView;view;view = view.superview) {
        NSLog(@"%@",view);
        if ([view isKindOfClass:[UICollectionViewCell class]]) {
            // 是一个
            UICollectionViewCell *cell = (UICollectionViewCell *)view;
            UICollectionView *cv = (UICollectionView *)cell.superview;
            NSIndexPath *indexPath = [cv indexPathForCell:cell];
            NSString *className = NSStringFromClass([cell class]);
            viewPath = [NSString stringWithFormat:@"%@[%ld:%ld]/%@",className,indexPath.section,indexPath.row,viewPath];
            continue;
        }
        
        if ([view isKindOfClass:[UITableViewCell class]]) {
            // 是一个
            UITableViewCell *cell = (UITableViewCell *)view;
            UITableView *tb = (UITableView *)cell.superview;
            NSIndexPath *indexPath = [tb indexPathForCell:cell];
            NSString *className = NSStringFromClass([cell class]);
            viewPath = [NSString stringWithFormat:@"%@[%ld:%ld]/%@",className,indexPath.section,indexPath.row,viewPath];
            continue;
        }
        
        
        if ([view isKindOfClass:[UIView class]]) {
            [view.superview.subviews enumerateObjectsUsingBlock:^(__kindof UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
                if (obj == view) {
                    NSString *className = NSStringFromClass([view class]);
                    viewPath = [NSString stringWithFormat:@"%@[%ld]/%@",className,idx,viewPath];
                    *stop = YES;
                }
            }];
        }
        
        UIResponder *responder = [view nextResponder];
        if ([responder isKindOfClass:[UIViewController class]]) {
            
            NSString *className = NSStringFromClass([responder class]);
            viewPath = [NSString stringWithFormat:@"%@/%@",className,viewPath];
            return viewPath;
        }
    }
    return viewPath;
}
```
感觉不是最优解，哈哈，
以上可以拿到。一个元素在当前控制器的路径。可以以此进行数据分析。
以上结束。谢谢！
感谢：
https://time.geekbang.org/column/article/87925（应该不能看）
https://www.infoq.cn/article/yoho-data-collection



