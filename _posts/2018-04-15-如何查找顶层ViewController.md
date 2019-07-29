---
layout: post
title: '如何查找顶层ViewController'
date: 2018-04-15 10:48:16
tags: []
---

为什么会有这个需求呢？

这起源于公司项目组件化，在设计Router时需要拿到顶层可见的ViewController，以及其所属的NavigationController进行相应的跳转。

期间查询了很多资料,无奈发现效果都不是很理想。

<!-- more -->

### 网上大多数做法：

1. 获取 **keyWindow.rootViewController**，做为当前控制器
2. 判断 当前控制器 isKindOfClass **UITabBarController** 或 **UINavigationController** 等容器控制器
3. 获取对应 **selectedViewController**，**topViewController**，**visibleViewController**，或**presentedViewController**等对象做为当前控制器
4. 递归，直至找出最顶层的控制器

### 还有一种做法是：

1. swizzled **viewDidAppear** 和 **viewDidDisappear** 方法
2. 在 **viewDidAppear** 方法中，将 **viewController** 加入栈中
3. 在 **viewDidDisappear** 方法中，将 **viewController** 从栈中移除

其中使用 **NSPointerArray** 替代 NSArray 作为栈，存储 viewController 的弱引用。


## 突破

在抓头挠腮，焦头烂额，搔首弄姿许久之后，终于被我灵光一闪...

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1515606800088&di=d607023fc1db8f965c006b3b0fe679b0&imgtype=0&src=http%3A%2F%2Fp0.ifengimg.com%2Fpmop%2F2017%2F0108%2F17397B1585D611209C1FC720740986C92C3470C0_size337_w316_h217.gif)

**在逆向中，有个用来查看ViewController的层级关系_printHierarchy方法**

Xcode的lldb输入：

```
(lldb) po [UIViewController _printHierarchy]
<JFHomeViewController 0x7fcb1ac38ec0>, state: disappeared, view: <RCTRootView 0x7fcb1ac45600> not in the window
   | <UITabBarController 0x7fcb1b0ba600>, state: disappeared, view: <UILayoutContainerView 0x7fcb1ac42950> not in the window
   |    | <UINavigationController 0x7fcb1b828000>, state: disappeared, view: <UILayoutContainerView 0x7fcb1af031e0> not in the window
   |    |    | <JFHomePageViewController 0x7fcb1ac448a0>, state: disappeared, view: <UIView 0x7fcb1ae1e120> not in the window
   |    | <UINavigationController 0x7fcb1c068a00>, state: disappeared, view: <UILayoutContainerView 0x7fcb1af4a250> not in the window
   |    |    | <JFInvestViewController 0x7fcb1af571a0>, state: disappeared, view: (view not loaded)
   |    | <UINavigationController 0x7fcb1b872000>, state: disappeared, view: <UILayoutContainerView 0x7fcb1ae20fe0> not in the window
   |    |    | <JFExploreViewController 0x7fcb1ae1f990>, state: disappeared, view: (view not loaded)
   |    | <UINavigationController 0x7fcb1b89e600>, state: disappeared, view: <UILayoutContainerView 0x7fcb1ae27b70> not in the window
   |    |    | <JFMineMainPageViewController 0x7fcb1ae26ec0>, state: disappeared, view: (view not loaded)
   + <UINavigationController 0x7fcb1c8dca00>, state: appeared, view: <UILayoutContainerView 0x7fcb1af498f0>, presented with: <_UIFullscreenPresentationController 0x7fcb1ac613c0>
   |    | <JFLoginViewController 0x7fcb1ac65030>, state: appeared, view: <RCTRootView 0x7fcb1afa7cb0>

```

试问，要是理解**_printHierarchy**的打印内容中各个控制器的关系，还愁找不到顶层控制器吗？

> 要想了解**_printHierarchy**的具体实现可查看[ViewControllerHierarchy](https://github.com/Mainstayz/ViewControllerHierarchy)，在本文中可以暂时不去理会。

## 探索

已知 **JFHomeViewController** 的地址为 **0x7fcb1ac38ec0** 在LLDB中输入以下语句：

```
(lldb) po [0x7fcb1ac38ec0 _ivarDescription]  
```

为了了解 **UINavigationController 0x7fcb1c8dca00** 是否与其有关系

在输入控制台中搜索地址 **0x7fcb1c8dca00** 
![](/images/1564384183858.jpg)

**JFHomeViewController** 的 **_childModalViewController** 为 **< UINavigationController 0x7fcb1c8dca00 >**

如此不断通过调用**_ivarDescription**,**_shortMethodDescription**这两个方法。得出以下关系：

![](/images/1564384196974.jpg)


总结一下：

* 外层与内层是**childViewControllers**与**parentViewController**关系。
* 如果有通过**presentViewController:animated:completion:**进行modal，还会有个**_childModalViewController**和**_parentModalViewController**链关系。
* 谁负责modal，则被modal出来控制器的**_modalSourceViewController**属性就会指向谁。如图中绿色箭头指向。
* 最顶层的**ViewController**的**State**都是**appeared**的。


## 思路

知道了以上关系，稍作思考，笔者的思路也确定下来了，如下：

![](/images/1564384208493.png)

途中碰到的问题：

### 如何去获取私有变量 **UIViewController** 的 **_modalSourceViewController**？

通过

`po [[UIViewController new] _shortMethodDescription]`

可找到如下代码

```
......
@property (nonatomic, setter=_setModalSourceViewController:) UIViewController* _modalSourceViewController;  (@synthesize _modalSourceViewController = _modalSourceViewController;)
......
- (void) _setModalSourceViewController:(id)arg1; (0x10bc84dd0)
- (id) _modalSourceViewController; (0x10bc84dbf)
......
```
那么通过KVC简单粗暴即可对其进行取值操作，不过KVC会触发 **setter**、**getter**方法，有可能会带来一些意想不到的结果。建议通过**Runtime**的**class_getInstanceVariable**，**ivar_getOffset**方法进行取值。

### 如何获取ViewController的state，并了解其枚举的含义？

在这里，我想到有两个方法。

* 直接通过 **_ivarDescription**以及**_shortMethodDescription** 查找关联性比较大的属性或者方法名。

```

_viewControllerFlags (struct ?): {
		appearState (b2): 2
		isEditing (b1): NO
		....
	}

// 比较相关的方法
...
- (BOOL) _hasAppeared; (0x104a2ad7e)
- (BOOL) _isAppearingOrAppeared; (0x104a2ad31)
- (int) _appearState; (0x104a2ad4c)
....

```
先看看这几个方法的反编译源码


```
int -[UIViewController _appearState](void * self, void * _cmd) {
    rax = self->_viewControllerFlags;
    rax = rax & 0x3;
    return rax;
}
```

1. 推测出4种枚举类型

```
//- (BOOL) _hasAppeared; (0x104a2ad7e)

bool -[UIViewController _hasAppeared](void * self, void * _cmd) {
    rax = self->_viewControllerFlags;
    rax = (rax & 0x3) == 0x2 ? 0x1 : 0x0;
    return rax;
}

```
1. 通过箭头指针获取_viewControllerFlags，与 0x3 进行二进制"与"运算 。可推测出_viewControllerFlags应该是一个位移枚举。
2. state有4种枚举类型。其中第2种为Appeared。

在看看**_isAppearingOrAppeared**的实现

```
bool -[UIViewController _isAppearingOrAppeared](void * self, void * _cmd) {
    rax = self->_viewControllerFlags;
    rax = (rax & 0x3) < 0x3 ? 0x1 : 0x0;
    return rax;
}
```

1. 已知第2种为Appeared。则第0个或第1个都可代表Appearing

感觉都不是很清晰。。。


* 反编译 **_printHierarchy** 。

**_printHierarchy** 内部调用的是 **_appendDescription()** 这个方法。

![](/images/1564384241970.jpg)

发现一个很可疑的方法**_descriptionForPrintingHierarchy**。

![](/images/1564384248586.jpg)

真相大白，当 **( self->_viewControllerFlags & 0x3 )** 结果为 **0** 时为 **disappeared** ，**1** 为 **appearing**，**2** 为 **appeared**，**3** 为 **disappearing**。



