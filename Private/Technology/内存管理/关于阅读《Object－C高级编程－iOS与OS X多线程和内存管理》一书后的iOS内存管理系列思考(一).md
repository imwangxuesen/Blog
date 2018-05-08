
关于阅读《Object－C高级编程－iOS与OS X多线程和内存管理》一书后的iOS内存管理系列思考

* [《关于iOS内存管理的规则思考》](http://blog.csdn.net/wxs0124/article/details/53787357)
* [《iOS内存管理——alloc／release／dealloc方法的GNUstep实现与Apple的实现》](http://blog.csdn.net/wxs0124/article/details/53840611)

# iOS内存管理——alloc／release／dealloc方法的GNUstep实现与Apple的实现
**接上篇[关于iOS内存管理的规则考](http://blog.csdn.net/wxs0124/article/details/53787357)我们通过alloc／release／dealloc方法的具体实现来深入探讨内存管理。**

##什么是GNUstep
		`GNUstep`是`Cocoa`框架的互换框架，从源代码的实现上来说，虽然和Apple不是完全一样，但是从开发者的角度来看，两者的行为和实现方式是一样，或者说是非常相似。
		因为`NSObject`类的`Foundation`框架是不开源的，不过，`Foundation`框架使用的`Core Foundation`框架的源码和通过调用`NSObject`类进行内存管理部分的源码是公开的。
		所以，我们先来了解`GNUstep`的实现方式，有助于我们去理解`Apple`的实现方式。

> [GNUstep更多详细内容请查看官网](http://gnustep.org/)

##搞出些事情
* GNUstep／modules/core/base/Source/NSObject.m alloc

```
+ (id)alloc {
    return [self allocWithZone:NSDefaultMallocZone()];
}

+ (id)allocWithZone:(struct _NSZone *)zone {
    return NSAllocateObject(self, 0, zone);
}
```

我们可以看到内存的分配最重是通过`allocWithZone`中`NSAllocateObject`方法分配的。

我们来看一下`NSAllocateObject`的声明：
```
NSAllocateObject(<#Class  _Nonnull aClass#>, <#NSUInteger extraBytes#>, <#NSZone * _Nullable zone#>)
```
实现：
```
static obj_layout {
    NSUInteger retained;
};

inline id

NSAllocateObject(Class  _Nonnull aClass, NSUInteger extraBytes, NSZone * _Nullable zone){
	int size = 计算容纳对象所需内存大小;
	id new = NSZoneMalloc(zone, size);
	memset(new, 0, zize);
	new = (id)&((static obj_layout *) new)[1];
        　
}

```
 通过上述代码可以看出，`NSAllocateObject`方法通过调用`NSZoneMalloc`方法来分配存放对象所需的内存空间，之后将该空间置0，最后返回作为对象而使用的指针。


> **区域**
> **NSZone是什么？**
> NSZone是Apple为了防止内存碎片化而引入结构。对内存分配的区域本身进行多重化管理，根据使用对象的目的，对象的大小分配内存，从而提高内存的管理效率。
> 但是，现在iOS的运行时系统知识简单的忽略了区域的概念。运行时系统中的内存管理本身已经极具效率，使用区域来管理区域反而会引起效率低下以及源码复杂化的问题。
> ![多区域内存管理](http://ww2.sinaimg.cn/mw1024/006bdQ7qjw1faznzzjz1pj30wb0uowk3.jpg)

去掉`NSZone`后的`alloc`源码

* GNUstep／modules/core/base/Source/NSObject.m alloc去NSZone版

```
+ (id)alloc {
    int size = sizeof(struct objc_layout) ＋ 对象大小;
    struct objc_layout *p = (struct objc_layout *)calloc(1, size);
    return (id)(p+1);
}
```
`objc_layout`中的`retained`是用来记录引用计数

此结构体在每一块对象内存的头部，盖对象内存块全部置0后返回，如图：
![alloc返回内存图](http://ww1.sinaimg.cn/mw1024/006bdQ7qjw1fazog7ur1gj30wg0jagnl.jpg)

我们可以使用`retainCount`实例方法获取当前的引用计数。
```
NSObject *obj1 = [[NSObject alloc] init];

NSLog(@"obj1 retain count = %lu", (unsigned long)[_obj1 retainCount]);
```
> 打印结果:
>  2016-12-21 16:04:07.579 acm[66972:880151] obj1 retain count = 1

```
- (NSInteger)retainCount {
    return NSExtraRefCount(self) + 1;
}

inline NSUInteger
NSExtraRefCount(id  _Nonnull object) {
    return ((struct obj_layout *)object)[-1].retained;
}
```

这段代码中 `((struct obj_layout *)object)[-1]`可能对C语言不熟悉的同学不太理解，我来解释一下。
> 首先［－1］的意思是指针向上移动一个对象的大小。
> 前边我们的讲述中，在对象内存的上方会有一个小的内存块用来存储引用计数的结构体`struct obj_layout`
> `(struct obj_layout *)`来修饰object是为了当我们移动指针的时候移动的大小是 一个`struct obj_layout`的大小，而不是一个`object`的大小.

********************

>前边我们也说到，每次内存分配后都会把对象的内存区域置0，这是为什么？
>这个问题我也有些蒙蔽，暂且搁置。希望有同学给予解答。


因为为了满足内存管理的规则，在分配时全部置`0`，所以`retained`为`0`.
在调用`retainCount`的时候会`return NSExtraRefCount（self）+ 1`,我们可以推测出，`retain`方法使`retained`变量`＋1`，而`release`方法是`retained`变量减`1`.


* GNUstep／modules/core/base/Source/NSObject.m retain

```
- (id)retain {
    NSIncrementExtraRefCount(self);
    return self;
}

inline void
NSIncrementExtraRefCount(id  _Nonnull object)  {
<!--retained最大值判断-->
	if (((struct obj_layout *)object)[-1].retained == UINT_MAX - 1) {
		[NSException raise:NSInternalInconsistencyException format:@"NSIncrementExtraRefCount() asked to increment too far"];
	}
<!--没有超过最大值则加1-->          
	((static obj_layout *)object)[-1].retained++;
	
}
```

* GNUstep／modules/core/base/Source/NSObject.m release

```
- (void)release {
    if (NSDecrementExtraRefCountWasZero(self)) {
        [self dealloc];
    }
}

BOOL
NSDecrementExtraRefCountWasZero(id  _Nonnull object) {
	if (((struct obj_layout *)object)[-1].retained == 0) {
		return YES;
	} else {
		return NO;
	}
}
```

相信大家对上边的两个方法的规则已经有一个了结，在调用`retain`／`release`都会判断极值，分别在条件允许时调用抛异常或`dealloc`


我们再来看看`dealloc`时如何实现的

* GNUstep／modules/core/base/Source/NSObject.m dealloc


```
- (void)dealloc {
    NSDeallocateObject(self);
}

inline void
NSDeallocateObject(id  _Nonnull object) {

	struct obj_layout *o = &((struct obj_layout *)object)[-1];
	
	free(o);
}

```

> 不难发现，`dealloc`方法获取到从存放 obj_layout 结构的内存头部开始的指针后释放它所指向的内存块。


##GNUstep中alloc／retain／release／dealloc中的实现总结

* 在`Objective－C`的对象中存有引用计数这个变量.
* 调用`alloc`或者`retain`后，引用计数`＋1`.
* 调用`release`后，引用计数`-1`.
* 引用计数为`0`时，调用`dealloc`废弃对象.

# 连猜带想，看苹果如何实现的alloc／retain／release／dealloc

我们看完了GNUstep中的各个方法实现方式，再来猜想推测一下苹果的实现，如今`NSObject`类的源码没有公开，我们利用`lldb`(Xcode调试器)和iOS的历史来追溯其实现方式。

在NSObject类的alloc类方法上设置断点，追踪函数调用

```
//NSObject alloc调用顺序
+alloc
+allocWithZone:
class_createInstance
calloc
```




可以看出这个调用顺序和GNUstep的调用顺序非常的相似
```
//GNUstep alloc调用顺序
+alloc
+allocWithZone:
NSAllocateObject
NSZoneMalloc
```

通过 [objc4](https://opensource.apple.com/source/objc4/)库中的 `reuntime/objc-runtime-new.mm`文件进行确认。

官方文档中的`class_createInstance`
![class_createInstance](http://ww2.sinaimg.cn/mw690/006bdQ7qjw1fb0puj15tsj30zo0zwjvj.jpg)


###我们在来看看`retainCount/retain/release`的实现方式（调用栈）。

*  retainCount

```
-retainCount
__CFDoExternRefOperation
CFBasicHashGetCountOfKey
```

* retain

```
-retain
__CFDoExternRefOperation
CFBasicHashAddValue
```

* release

```
-release
__CFDoExternRefOperation
CFBasicHashRemoveValue
```

这个时候们必需去看看__CFDoExternRefOperation做了什么

* CF/CFRuntime.c __CFDoExternRefOperation

```
int __CFDoExternRefOperation(uintptr_t op,id obj) {
	CFBasicHashRef table = 取得对象对应的散列表（obj）；
	int count；
	
	switch(op) {
		//retainCount
		case OPERATION_retainCount:
			count = CFBasicHashGetCountOfKey(table, obj);
			return count;
		
		//retain
		case OPERATION_retaion:
			CFBasicHashAddValue(table, obj);
			return obj;
			
		//release
		case OPERATION_release:
			count = CFBasicHashRemoveValue(table, obj);
			return 0==count;
	}
}
```

看到这里我们也就可以大致推测三个方法的实现方式了：

```
- (NSUInteger)retainCount {
    return (NSUInteger)__CFDoExternRefOperation(OPERATION_retainCount, self);
}

- (id)retain {
    return (id)__CFDoExternRefOperation(OPERATION_retain, self);

}

- (void)release {
    return __CFDoExternRefOperation(OPERATION_release, self);
}
```

##推测苹果实现alloc／retain／release/retainCount方法的总结

我们可以从`__CFDoExternRefOperation`函数和由次函数调用的各个函数名看出，苹果采用了[散列表](http://baike.baidu.com/link?url=gHP63psom8NnhkE-1v1nhCduPrTpITwlR6_tHZ74cgqGor8f48ZjI21TFXAbAp_TVKVY3U1PSOlU-52msbB5mZqIM1buzSgVaGwgF2GCmsi6RPR45W41CODhAX5B0Mg44tAJwjO1Qi5r3vzLctgA4KFWYnDBOt19Bf7YrIEiIasHAJ6PZOPBdXs5FbzKdVJz4PeEJTo3l2c457GnYNag2_)（引用计数表）来管理引用计数。

如图：

![](http://ww4.sinaimg.cn/mw690/006bdQ7qjw1fb0qvkrcyxj30ob0semzy.jpg)

不难看出，GNUstep将引用计数保存在对象占用内存块头部的变量中，而苹果的实现，则是保存在用用计数表的记录中，GNUstep的实现在我们看来是没有任何问题的，但是苹果这么实现也有它的道理。

> 使用内存块头部管理引用计数好处：

* 少量代码即可实现
* 能够在同一张表中统一的管理引用计数用内存块和对象用内存块。

> 使用引用计数表来管理引用计数的好处：

* 对象用内存块的分配无需考虑头部内存块的存在。
* 引用计数表各记录中存有内存块地址，可以从各个记录追溯到对象的内存块。即使出现故障导致对象占用的内存块损坏，但只要引用计数表没有损坏，就能确认各个内存块的位置。
* 有助于利用工具检测各个对象的持有者是否存在，即检测内存泄漏的原理。



