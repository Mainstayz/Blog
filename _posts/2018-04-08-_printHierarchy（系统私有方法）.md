---
layout: post
title: '_printHierarchy（系统私有方法）'
date: 2018-04-08 00:51:47
tags: []
---
实现**_printHierarchy**

<!-- more -->


**_printHierarchy** 为 UIViewController 的方法，说明了就在UIKit中。下载个Xcode7，在里面找出UIKit，拖到 **Hopper** 中查看。

![](/images/1564382707678.jpg)

- **_printHierarchy** 分类方法和实例方法。
- 在 **void * +[UIViewController _printHierarchy]** 类方法中
  - return [[UIWindow keyWindow] _printHierarchy];

------

再看看实例方法 **void * -[UIViewController _printHierarchy]**

![](/images/1564382727534.jpg)

由上图可得如下信息

- 返回值为 **NSMutableString** 类型
- **_appendDescription_1ba2c6();** 为核心实现，对**NSMutableString**实例进行拼接

------

下面再看看 核心方法**_appendDescription_1ba2c6()** 实现

```
int _appendDescription_1ba2c6() {
    var_108 = rdx; // 获取第三个形参
    var_30 = *___stack_chk_guard;
    r15 = [rdi retain];  // 获取第一个形参
    r13 = [rsi retain];  // 获取第二个形参    到这里说明了至少传进来了三个形参
    
    if ([r13 length] != 0x0) { // r13 推断出 NSMutableString * 类型
            [r13 appendString:@"\n"];
    }
    rbx = [[r15 _parentViewController] retain]; // 由 _parentViewController 推断出 r15 为 UIViewController ，也就说 第一个形参 为 UIViewController * 类型。
    rdi = r15;
    if (rbx != 0x0) { // 如果有 _parentViewController
            var_100 = rdi;
            [rbx release];
            if (var_108 != 0x0) {   // var_108 不为 0
                    r12 = sign_extend_64(var_108); // r12 = var_108
                    r14 = @selector(appendString:);
                    rbx = 0x1;
                    r15 = @"   | ";                
                    do {
                            _objc_msgSend(r13, r14);
                            rbx = rbx + 0x1;       //  rbx ++
                    } while (rbx <= r12);          //  rbx <= var_108  到这里推断出 第三个形参 为 int 类型，根据上文 appendString:@"   | "，又可推断出第三个形参代表着层级。
            }
            r15 = var_100;
            // 可猜出类似返回为<UINavigationController 0x7f81cc088000>, state: disappeared, view: <UILayoutContainerView 0x7f81cad640f0> not in the window 字符串
            rbx = [[r15 _descriptionForPrintingHierarchy] retain]; 
            var_F8 = r13;
            [r13 appendString:rbx]; //拼接
            [rbx release];
    }
    else { 
            var_100 = rdi;  //var_100 为 当前viewController
            r14 = [rdi _isRootViewController];  // 判断 viewController 是否为_isRootViewController 
            [rbx release];
            if (r14 != 0x0) {   // _parentViewController == nil && rootViewController 
                    if (var_108 != 0x0) { // 由上面可分析出语意 如果层级数不为0
                            r12 = sign_extend_64(var_108);
                            r14 = @selector(appendString:);
                            rbx = 0x1;
                            r15 = @"   | ";
                            do {
                                    _objc_msgSend(r13, r14);
                                    rbx = rbx + 0x1;
                            } while (rbx <= r12);
                    }
                    r15 = var_100;
                    rbx = [[r15 _descriptionForPrintingHierarchy] retain];
                    var_F8 = r13;
                    [r13 appendString:rbx];
                    [rbx release];
            }
            else {  // _parentViewController == nil && 也不是 rootViewController ，说明是被modal出来的， 说明 var_100 也就是 rdi 第一个参数，为modal出来的 👇
                    r14 = @selector(appendString:);
                    rbx = 0x1;
                    if (var_108 > 0x1) {
                            r12 = sign_extend_64(var_108);
                            r15 = @"   | ";
                            do {
                                    _objc_msgSend(r13, r14);
                                    rbx = rbx + 0x1;
                            } while (rbx < r12);
                    }
                    
                    _objc_msgSend(r13, r14); // 从下面推断到这里的时候，可以预判出拼接了  “ + ”字符串  👆
                    r15 = [[var_100 _descriptionForPrintingHierarchy] retain]; // r15 为 <UINavigationController 0x7f81cc9f2e00>, state: appeared, view: <UILayoutContainerView 0x7f81d1013100>， var_100 符合判断为modal出来的 👆
                    
                    var_120 = r15;
                    rax = [var_100 presentingViewController]; // rax 为  UINavigationController 的 presentingViewController 👆
                    rax = [rax retain];
                    var_110 = rax;
                    rax = [rax _presentationController];
                    rax = [rax retain];
                    var_118 = rax; 
                    r14 = [[rax _descriptionForPrintingViewControllerHierarchy] retain];    //  推出 rax 为 _UIFullscreenPresentationController 👆
                    r15 = [[NSMutableString stringWithFormat:@"%@, presented with: %@", r15, r14] retain];  // 拼接出 <UINavigationController 0x7f81cc9f2e00>, state: appeared, view: <UILayoutContainerView 0x7f81d1013100>, presented with: <_UIFullscreenPresentationController 0x7f81d1137710>，其中r15, r14可往上进行推断 👆

                    var_F8 = r13;
                    [r13 appendString:r15];
                    [r15 release];
                    r15 = var_100;
                    [r14 release];
                    [var_118 release];
                    [var_110 release];
                    [var_120 release];
            }
    }
    var_100 = r15;
    intrinsic_movaps(var_C0, 0x0);
    intrinsic_movaps(var_D0, 0x0);
    var_E0 = intrinsic_movaps(var_E0, 0x0);
    var_F0 = intrinsic_movaps(var_F0, 0x0);
    r12 = [[r15 childViewControllers] retain]; //获取 childViewControllers 
    rdx = var_F0;
    r14 = [r12 countByEnumeratingWithState:rdx objects:var_B0 count:0x10];
    if (r14 != 0x0) {
            r15 = *var_E0;
            rbx = var_108 + 0x1; // 层级 var_108  + 1
            do {    // 遍历 childViewControllers
                    r13 = 0x0;
                    do {
                            if (*var_E0 != r15) {
                                    objc_enumerationMutation(r12);
                            }
                            _appendDescription_1ba2c6();  // 递归
                            r13 = r13 + 0x1;
                    } while (r13 < r14);
                    rdx = var_F0;
                    r14 = [r12 countByEnumeratingWithState:rdx objects:var_B0 count:0x10];
            } while (r14 != 0x0);
    }
    [r12 release];
    rsi = @selector(childModalViewController);  
    r15 = var_100;
    rbx = [_objc_msgSend(r15, rsi) retain]; //获取 childModalViewController 
    [rbx release];
    r12 = var_F8;
    if (rbx != 0x0) {
            rbx = [[r15 childModalViewController] retain];
            rdx = var_108 + 0x1;            // 层级 var_108  + 1
            rsi = r12;
            _appendDescription_1ba2c6();    //递归
            [rbx release];
    }
    [r12 release];
    [r15 release];
    rax = *___stack_chk_guard;
    if (rax != var_30) {
            rax = __stack_chk_fail();
    }
    return rax;
}

```

最终完美实现私有方法，如下图

![](/images/1564382746953.jpg)

- 红色箭头是UIKit的实现
- 绿色箭头是自己的实现

最后附上代码地址：

[https://github.com/Mainstayz/ViewControllerHierarchy](https://github.com/Mainstayz/ViewControllerHierarchy)

