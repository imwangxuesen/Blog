#NSObject到底多么大引发的一些思考

本文引用及参考文献，感谢一下博主的分享：

- [C++ 内存对齐---by enos](http://www.cnblogs.com/TenosDoIt/p/3590491.html)

- [小码哥iOS学习笔记第一天: Objective-C的本质
 ---by 
冰凌天](https://juejin.im/post/5b248ad151882574e808d3c9)

- [Objective-C 检测运行时对象的内存大小
---by 蓝新](https://www.jianshu.com/p/83b13c65a9bb)

- [How to find the size of any object in iOS
](https://stackoverflow.com/questions/8223560/how-to-find-the-size-of-any-object-in-ios)


一个问题，一个NSObject的实例占多大内存？

## 几个概念
我们先来明确几个计算机概念，位(bit)、字节(byte)、字

- 位(bit)

计算机内部数据储存的最小单位，我们所谓的几位，就是常见的二进制中的一位。

- 字节(byte)

计算机中数据处理的基本单位，计算机中以字节为单位存储和解释信息。一个字节8bit

- 字(word)

计算机进行数据处理时，一次存取、加工和传送的数据长度称为字。和它相关的一个概念叫字长，是标识字的bit数，在32位机器中，计算机总线一次传输32位=4字节。字64位机器中，计算机总线一次传输64位=8字节。所以64位机比32位机速度快很多

内存中的计算都是用bit来标识的，可能是因为内存本身就是稀缺资源，并没有很大，存储的内容也不会过大。

言归正传

##实验环境
MacBook Pro (Retina, 13-inch, Early 2015)
10.14 Beta (18A365a)

XCode Version 9.4.1 (9F2000)

iPhone8 Plus 
iOS 11.4.1（15G77）

也就是64位的环境
## 一个NSObject的大小实验

### 一个绝对干净的NSObject多么大
```
#import <malloc/malloc.h>

    NSObject *obj = [[NSObject alloc] init];
    
    NSLog(@"point size: %ld\n", sizeof(obj));
    NSLog(@"object size: %ld\n", malloc_size((__bridge const void *)obj));

```

malloc_size: 返回指针所指向对象字节数。但是这种方法不会考虑到对象成员变量指针所指向对象所占用的内存。

sizeof: 返回一个对象或类型所占的内存字节数。[详细解释sizeof用法](https://blog.csdn.net/oktears/article/details/19352577)

使用C++的malloc_size计算实例大小，sizeof计算指针大小

```
TEST[1650:139260] point size: 8
TEST[1650:139260] object size: 16
```

我们看到一个object占16字节，一个object指针占8字节

### 一个基础类型变量多么大
```
int i = 11111;
double d = 0.0;
float f = 0.3;
long l = 11111;

NSLog(@"int size: %ld\n", sizeof(i));
NSLog(@"double size: %ld\n", sizeof(d));
NSLog(@"float size: %ld\n", sizeof(f));
NSLog(@"long size: %ld\n", sizeof(l));

结果：
TEST[1741:146323] int size: 4
TEST[1741:146323] double size: 8
TEST[1741:146323] float size: 4
TEST[1741:146323] long size: 8

```

### 一个“不干净”的NSObject多么大

根据上一个实验，如果一个继承自NSObject的类中有一个字符串属性，使用malloc_size方法计算出来的大小应该是 16(NSObject自身大小) + 8(NSString类型指针大小) = 24。
事实并不是这样的，我们尝试一下


定义一个We类

```
// We.h
#import <Foundation/Foundation.h>

@interface We : NSObject
- (void)logInfo;
@end

// We.m
@implementation We {
    NSString *str;
}

- (void)logInfo;
{
    NSLog(@"str size : %ld",sizeof(arr));
    NSLog(@"str  malloc size : %ld",malloc_size((__bridge const void *) str));
}

```

// 调用We类

```
We *we = [[We alloc] init];
[we logInfo];
        
NSLog(@"we point size: %ld\n", sizeof(we));
NSLog(@"we object size: %ld\n", malloc_size((__bridge const void *)we));
    
```

结果：

```
TEST[324:11186] str size : 8
TEST[324:11186] str malloc size : 0
TEST[324:11186] we point size: 8
TEST[324:11186] we object size: 16
```
这个和我们的预计结果又出入，他的size 是 16 而不是 24。我们再加一个看看



```
// We.m
@implementation We {
    NSString *str;
    NSString *str2;
}

```
```
TEST[324:11186] we object size: 32

```
好，再一次超出我的预料，上一个实验表明，如果 一个字符串的指针是8字节，Object本身是16字节，一个带有一个字符串指针的Object也是16字节，那么带两个呢？竟然是32字节。那我们再试试3个


```
// We.m
@implementation We {
    NSString *str;
    NSString *str2;
    NSString *str3;
}

```
```
TEST[324:11186] we object size: 32

```
**32！！！** 


## 探究原理
让我们来捋一下

| 带有字符串个数 | Object大小 |
|-------------|------------|
|1|16|
|2|32|
|3|32|
|4|48|
|5|48|
|6|64|

不难看出，object最小就是16字节，并且每次增长都是16的倍数，即便你添加的属性并没有占到16，它会自动补齐到16字节。

然后去查了一下，发现很多人写过这个问题的总结（孤陋寡闻了）,几篇博主的套路都是一样的

我们根据各位博主的代码做一下实验：

先解释一下所用的方法：

```
class_getInstanceSize

// 定义
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}


// Class's ivar size rounded up to a pointer-size boundary.
// (类中成员变量的指针的空间范围)
uint32_t alignedInstanceSize() {
	return word_align(unalignedInstanceSize());
}

```

实验代码：

```
NSObject *object = [[NSObject alloc] init];
        
//获得NSObject 类的实例对象的大小
NSLog(@"NSObject Instance Size:%zd",class_getInstanceSize([NSObject class])  );

//此处很多博主的注释是“获取obj对象指针获取的大小”，我觉得是有问题的，
//更合适的解答是指针指向的内存空间大小
NSLog(@"NSObject Point Size:%zd",malloc_size((__bridge const void *)object));  
```
```
TEST[1359:104954] NSObject Instance Size:8
TEST[1359:104954] NSObject Point Size:16
```
**也就是说Object的实例在内存中占8字节，但是指针所指向的内存空间确是16字节。**

**也就是说他多分配了8个字节的空间。为什么？这不是浪费吗？**

我们再来测试一下：

```

@implementation We {
    NSString *str;
    int age;
}

We *we = [[We alloc] init];
NSLog(@"We Instance Size:%zd",class_getInstanceSize([We class]));
NSLog(@"We Point Size:%zd",malloc_size((__bridge const void *)we));
```

预测结果： 20 ， 32.

实际结果： 24 ， 32

```
TEST[1859:153420] We Instance Size:24
TEST[1859:153420] We Point Size:32
```
这里牵扯到一个内存对齐的概念此处不展开解释，大家自行观看吧[内存对齐是什么](http://www.cnblogs.com/TenosDoIt/p/3590491.html)

##我们的核心问题
**带着这几个问题我们看下边的实验:**

**1,为什么NSObject分配的空间是16的倍数?**

**2,为什么class_getInstanceSize获取大小并不是真实的各个属性指针所占有的实际大小？**


### 问题1 为什么NSObject分配的空间是16的倍数?
使用
`xcrun  -sdk  iphoneos  clang  -arch  arm64  -rewrite-objc main.m -o mian.cpp`
命令，将OC代码转换成c++代码

```
#ifndef _REWRITER_typedef_NSObject
#define _REWRITER_typedef_NSObject
typedef struct objc_object NSObject;
typedef struct {} _objc_exc_NSObject;
#endif

struct NSObject_IMPL {
	Class isa;
};
```
这个是NSObject的结构，只有一个Class 指针，所以`class_getInstanceSize ` 计算 NSObject 实例的大小为8字节也就可以解释了，因为只有一个指针

我们再来看看为什么`malloc_size`的结果是16.
malloc是申请内存空间的函数，OC的分配空间函数是alloc，那我们来看一下它的实现

```
+ (id)alloc {
    return _objc_rootAlloc(self);
}

// Replaced by ObjectAlloc
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}

```

```

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}


id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone)
{
    id obj;

#if __OBJC2__
    // allocWithZone under __OBJC2__ ignores the zone parameter
    (void)zone;
    obj = class_createInstance(cls, 0);
#else
    if (!zone) {
        obj = class_createInstance(cls, 0);
    }
    else {
        obj = class_createInstanceFromZone(cls, 0, zone);
    }
#endif

    if (slowpath(!obj)) obj = callBadAllocHandler(cls);
    return obj;
}

```

代码冗长，我们来解读一下，我们在实例化一个对象的时候会用`alloc`申请空间，`alloc` 方法调用了 `callAlloc `方法， 在此方法的最后有一个判断，如果存在`allocWithZone`还是会调用`allocWithZone `，所以最终的内存空间的申请会落实到`allocWithZone`方法中。

`allocWithZone`方法中调用了`_objc_rootAllocWithZone`，此方法中再调用`class_createInstance`,此方法实现如下：

```
id  
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}


static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();

    size_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) {
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    return obj;
}

```

再`_class_createInstanceFromZone `方法中我们看到了我们期盼的`size`字眼，但是他是通过一个方法计算的，我们再看它的实现

```
size_t instanceSize(size_t extraBytes) {
	size_t size = alignedInstanceSize() + extraBytes;
	// CF requires all objects be at least 16 bytes.
	if (size < 16) size = 16;
	return size;
}
```
至此，问题1就真相大白了。

### 问题2 为什么class_getInstanceSize获取大小并不是真实的各个属性指针所占有的实际大小？

```
objc_runtime_new 文件中我们找到

// May be unaligned depending on class's ivars.
// 非内存对齐的实例大小
uint32_t unalignedInstanceSize() {
	assert(isRealized());
	return data()->ro->instanceSize;
}

// Class's ivar size rounded up to a pointer-size boundary.
// 内存对齐的实例大小
uint32_t alignedInstanceSize() {
	return word_align(unalignedInstanceSize());
}

```

然后我们再了解一下[内存对齐](http://www.cnblogs.com/TenosDoIt/p/3590491.html)的概念，也就不言而喻了

#总结：
1，OC中的对象是按16字节的倍数来分配内存的，会存在内存对齐的问题。
2，使用class_getInstanceSize获取的也并不是实际的对象、指针的内存空间，也会存在内存对齐问题。
3，基础类型变量的会比对象类型要节省内存。
4，源码解释一切。



