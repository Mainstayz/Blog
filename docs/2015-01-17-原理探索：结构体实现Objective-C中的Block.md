---

layout: post
title: '原理探索：结构体实现 Objective-C 中的 Block'
date: 2015-01-17 11:28:07
tags: []
----------------------------------------------------------------------------------------------------

iOS 日常开发中，几乎天天与 **block** 接触，如 API 的回调，传值，替代 delegate，动画，GCD 等等，这里只简单探究 **block** 生成与执行过程原理，重点去思考代码是怎么来的。

<!-- more -->

**如有不正确的地方，请不吝赐教。**


先来看一段 Objective-C 简单的代码

```
int main(int argc, const char * argv[]) {
  
    __block int i = 1024;
  
    int j = 1;
  
    void (^blk)(void);
  
    blk = ^{
        printf("%d, %d\n", i, j);
    };
  
    i ++;
    j ++;
  
    blk();

}
```

blk 中对 i，j 进行了引用，并分别打印出它们的值。输出结果为 1025， 1，并不是 1015， 2。这是为什么呢？

我们由上至下一行行去分析上面的 Objective-C 代码。

**__block 究竟是什么**

首先是这一行代码：

`__block int i = 1024;`

观察一下在 Block_private.h 文件中对__block 进行了定义，被__block 修饰的变量最终转换成类似该结构体。

> 来源： http://llvm.org/svn/llvm-project/compiler-rt/trunk/lib/BlocksRuntime/Block_private.h

```
struct Block_byref {
	 // isa指针
    void *isa;
  
    // 指向该结构体类型的指针，通过修改forwarding指向的，实现copy等操作。
    struct Block_byref *forwarding;
  
    int flags; /* refcount; */
    int size;
  
    // 辅助函数，对block范围外的变量进行拷贝／销毁时使用，不在本文的讨论范围。
    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
    void (*byref_destroy)(struct Block_byref *);
  
    /* long shared[0]; */
    // 被包装的变量会被加到该结构体中
};
```

这个结构体中含有 isa 指针，所以是一个对象，它主要是用来包装局部变量的。

现在将__block int i = 1024 ;转换成结构体：

```

#include <iostream>
#pragma mark - __block 修饰的变量 i 的定义
struct block_byref_i {
    void *isa;
    block_byref_i *forwarding;
    int i;
};

#pragma mark - main 函数
int main(int argc, const char * argv[]) {
  
	// 下面的代码类似于 __block int i = 1024
	block_byref_i i;
	i = {(void*)0,&i,1024};
 	int j = 1;
  	return 0;
}
```

**block 的声明**

我们在看下一行代码：

```
void (^blk)(void);
```

这行代码声明了一个返回值为 void 名为 blk 的 block 变量，形参为 void。\^符号代表 blk 是 Block 变量，而不是普通的指针变量。

可以看到 Block 对象的声明语法和 C 函数指针的声明语法非常相似。

在 C 语言中，可以通过函数指针来调用其指向的函数，比如：

```
void foo(){
 printf("foo\n");
}
bool foo1(){
	return true;
}
int main(int argc, const char * argv[]) {
	void (*blk)(void); // 函数指针
	blk = &foo;
	blk();  // 这里会执行foo方法，这跟Objective-C一致
	bool (*bBlk)(void); // 函数指针
	bBlk = &foo1;
	printf("%d\n",bBlk());
	return 0;
}

```

不妨大胆猜想 void (\^blk)(void);本质上就是一个函数指针，它表示 block 实现的函数类型。

将 void (\^blk)(void);加入到 main 函数中，代码如下：

```
#pragma mark - __block 修饰的变量 i 的定义

struct block_byref_i {
    void *isa;
    block_byref_i *forwarding;
    int i;
};
#pragma mark - main 函数

int main(int argc, const char * argv[]) {
  
    block_byref_i i;
    i = {(void*)0,&i,1024};
    int j = 1;
  
    void (*blk)(void); // blk 为函数指针。
  
    return 0;
}

```

**block 的实际结构**

接下来，我们要解析 block，先去观察一下 Block_private.h 文件中对 block 的相关结构的真实定义：

```
/* Revised new layout. */
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
  
    //copy/dispose，辅助拷贝/销毁函数，处理block范围外的变量时使用
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
    // 被引用的所有变量
};

```

* Block_descriptor (不在本文讨论范围内)
* Block_layout
  * isa，指向所属类的指针，也就是 block 的类型
  * flags，标志变量，在实现 block 的内部操作时会用到
  * Reserved，保留变量
  * invoke, block 执行时调用的函数指针，block 定义时内部的执行代码都在这个函数中。
  * 被引用的所有变量。

现在来解析下面的 block

```
blk = ^{
        printf("%d, %d\n", i, j);
    };

```

有以下属性：

* 首先 isa，flags，Reserved，descriptor 暂时都不关心，已经脱离了本文讨论范围
* invoke 函数指针，指向 block 的实现
* __block int i
* int j

**结构体定义 Block**

```
#pragma mark - block 具体结构

struct block_impl {
  
    void *isa;
  
    // block 中代码块的函数指针
    void *invoke;
  
    // __block int i;
    block_byref_i *i;

    int j;
  
    //block_impl 结构体的构造函数
  
    /* 
     *  形参
     *  fp 为block执行时调用的函数指针
     *  _i 为block_byref_i结构体指针
     *  _j 简单的值传递
     */
  
  
   // 此时的j已经不是block外面的j,这就是为什么最后打印j的结果是1而不是2的原因。
    block_impl(void *fp, block_byref_i * _i, int _j):i(_i->forwarding),j(_j){
    		isa = (void *)0;
     		invoke = fp;
    }
};
```

接下来我们实现 block 的执行函数。
因为 block 结构体包含了变量 i,j ，所以执行函数形参为该结构体的指针，如下图：

```
static void block_func(struct block_impl *__cself) {
    block_byref_i *_i = __cself->i;
    int _j = __cself->j;
    // 注意，这里的_、forwarding用来保证操作的始终是堆中的拷贝i，而不是栈中的i
    printf("%d, %d\n", (_i->forwarding)->i, _j);
}

```

> 注意上面的 forwarding，当 block 被 copy 到堆中时，block 中的拷贝辅助函数也会将栈中 block_byref_i i 拷贝至堆中，并将堆和栈中的 block_byref_i i 的 forwarding 指针都指向堆中的拷贝。
> 参考：http://llvm.org/svn/llvm-project/compiler-rt/trunk/lib/BlocksRuntime/runtime.c

将以上的代码加入的 main.cpp 文件中
![](/images/1564382185458.jpg)

**实现 block 的实例**

```
//创建一个block对象

block_impl blkImp = block_impl((void *)block_func, &i, j);
```

**那么问题来了？void (*blk)(void)中的函数指针 blk 应该指向谁？**

1. 函数指针 blk 指向 函数 block_func？
2. 还是指向结构体 blkImp？

按照平常思维来，函数指针应该指向相对应类型的函数，函数指针调用函数 blk()跟 Objective－C 中的 block 的执行一样。应该选 1,相关代码如下:

```
int main(int argc, const char * argv[]) {
  

    // 下面的代码类似于 __block int i = 1024;
  
    block_byref_i i;

    i = {(void*)0,&i,1024};
  
    int j = 1;
  
    // 注意这里修改了，函数指针类型
    // 之前为 void (*blk)(void); 
    void (*blk)(block_impl*); 
  
    block_impl blkImp = block_impl((void *)block_func, &i, j);

    i.i ++;
  
    j++;
  
    blk = &block_func;
  
    blk(&blkImp);

   
    return 0;
}
输出结果：
1025， 1     // 跟Objective-C中的block完全一致。
```

运行结果正确，仅需要修改 blk 的函数指针类型就可以实现。看起来好像已经把 block 给用结构体实现了。但是通过 blk 却拿不到 block 对象了，所以，这种做法是错误的。

现在选择 2，相关代码如下：

```
int main(int argc, const char * argv[]) {
  

    // 下面的代码类似于 __block int i = 1024;
  
    block_byref_i i;

    i = {(void*)0,&i,1024};
  
    int j = 1;
  
    void (*blk)(void);
  
    block_impl blkImp = block_impl((void *)block_func, &i, j);
  
    blk = (void(*)(void))&blkImp;

    i.i ++;
  
    j++;
  

    ((void(*)(block_impl*))((block_impl *)blk)->invoke) ((block_impl *)blk);

   
    return 0;
}
输出结果：
1025， 1

```

完美！通过下面这行代码强行让函数指针 blk 指向 block 对象 blkImp。

```
blk = (void(*)(void))&blkImp;

```

调用 block 代码如下：

```
((void(*)(block_impl*))((block_impl *)blk)->invoke) ((block_impl *)blk);

//	太长了可以看如下代码
//    block_impl* imp = (block_impl *)blk;    // 1
//  
//    void *fp = imp->invoke;               // 2
//  
//    void(*final)(block_impl*) = (void(*)(block_impl *))fp; // 3
//  
//    final(imp); // 4
```

1. 将函数指针 blk 类型强转为结构体指针 block_impl *。
2. 然后取其执行函数指针。
3. 然后将此指针类型强转为返回 void* 并接收一个 block_impl* 的函数指针。
4. 最后调用这个函数，传入强转为 block_impl* 类型的 blk。

最后奉上完整代码：

```

/* main.app */

#include <iostream>


#pragma mark - __block 修饰的变量 i 的定义

struct block_byref_i {
    void *isa;
    block_byref_i *forwarding;
    int i;
};
#pragma mark - block 具体结构

struct block_impl {
    void *isa;
    void *invoke;
    block_byref_i *i;
    int j;
    block_impl(void *fp, block_byref_i * _i, int _j):i(_i->forwarding),j(_j){
        isa = (void *)0;
        invoke = fp;
    }
};

#pragma mark - block 代码块的实现函数

static void block_func(struct block_impl *__cself) {
    block_byref_i *_i = __cself->i;
    int _j = __cself->j;
    printf("%d, %d\n", (_i->forwarding)->i, _j);
}


#pragma mark - main 函数

int main(int argc, const char * argv[]) {
  
    block_byref_i i;
    i = {(void*)0,&i,1024};
    int j = 1;
  
    void (*blk)(void);
  
    block_impl blkImp = block_impl((void *)block_func, &i, j);
    blk = (void(*)(void))&blkImp;

    (i.forwarding->i) ++;
    j++;
  
    ((void(*)(block_impl*))((block_impl *)blk)->invoke) ((block_impl *)blk);
    return 0;
}
```
