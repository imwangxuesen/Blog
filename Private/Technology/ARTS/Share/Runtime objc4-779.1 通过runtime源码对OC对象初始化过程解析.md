# Runtime objc4-779.1 通过runtime源码对OC对象初始化过程解析
## 常用对象初始化代码
```
[[NSArray alloc] init]
[NSArray new]
```
我们先根据 alloc+init的方式来捋一遍runtime初始化对象的过程,看看有哪些值得我们学习的地方.

==以下书写方式为 <步骤号><代码所在文件><代码行数>==

所以阅读本文,最好是同步对照 [objc4-779.1](https://opensource.apple.com/tarballs/objc4/) 源码一起
### 1 NSObject.mm line 2317
```
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

### 2 NSObject.mm line 1717

```
id _objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```

### 3 NSObject.mm line 1697

```

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
// 是否是Objective-C 2.0
#if __OBJC2__

	// 3.1 判定是否检查cls是否为空,如果检查,为空则返回nil
    if (slowpath(checkNil && !cls)) return nil;
    
    // 3.2 检查本类、父类的继承链上是否有实现自己的allocWithZone方法,如果没有,就默认调用元类的
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif
	
	// 3.3 判断是否要调用allocWithZone方法,使用runtime的方法调用常规操作.
    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}
```

### 4 NSObject.mm line2321
```
// 替代 alloc方法
// Replaced by ObjectAlloc
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}
```

### 5 NSObject.mm line2534
```

id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone)
{
    id obj;

    if (fastpath(!zone)) {
    	// class_createInstance内部实质调用的是 _class_createInstanceFromZone 方法
        obj = class_createInstance(cls, 0);
        
    } else {
        obj = class_createInstanceFromZone(cls, 0, zone);
    }

    if (slowpath(!obj)) obj = _objc_callBadAllocHandler(cls);
    return obj;
}
```

所以在本步骤,obj初始化的过程成看似分叉,其实都集合到了调用`_class_createInstanceFromZone`

### 6 objc-runtime-new.mm line7385
```
/***********************************************************************
* class_createInstance
* fixme
* Locking: none
*
* Note: this function has been carefully written so that the fastpath
* takes no branch.
**********************************************************************/
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
	// 6.1 如果类对象本身没有被实现,则发出警告,并终止程序
    ASSERT(cls->isRealized());
		
    // 6.2 通过位运算获取类的信息,以提高性能
    // Read class's info bits all at once for performance
    // 是否有C++ 构造函数
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    // 是否有C++ 析构函数
    bool hasCxxDtor = cls->hasCxxDtor();
    //如果isa_t指针可用,则为true,反之为false,2.0版本中绝大多数类都是支持isa_t的,如果对isa和isa_t不清楚的,可以看之前的博文 https://blog.csdn.net/wxs0124/article/details/84401718
    bool fast = cls->canAllocNonpointer();
    size_t size;
	
	// 初始化的大小,instanceSize保证大小必须大于等于16bytes,内部总结就是用class_ro_t的isa_t的大小(8bytes)加上extraBytes,如果小于16bytes则补齐为16bytes, 这样做是为了内存对齐.
    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    // 6.3开辟内存空间
    if (zone) {
    	// 空间大小为size大小
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        // 空间初始化默认为0或者nil
        obj = (id)calloc(1, size);
    }
    // 这里判断是否有构造函数,如果没有终止,到这里我们遇到了fastpath和slowpath,稍后我们看一下他的源码和作用
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }
    
	// 6.4 初始化isa_t
    if (!zone && fast) {
    	//initInstanceIsa方法内部也是调用initIsa方法,只是因没有空间并且有isa_t指针而传参不同
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }
	
    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}

```

### 7 objc-object.h line214

```

inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    ASSERT(!cls->instancesRequireRawIsa());
    ASSERT(hasCxxDtor == cls->hasCxxDtor());
	// 没有zone的情况下,该方法才回被调用,对应的,空间是默认为0或者nil的,需要创建isa_t
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    ASSERT(!isTaggedPointer()); 
    
    //	7.1 如果有isa_t
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
    // 7.2如果没有isa_t 则进行初始化isa_t
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```

### 8 NSObject.mm line2331
```
- (id)init {
    return _objc_rootInit(self);
}

id	_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

至此,对象通过alloc初始化的过程就完成了,最终目标就是对isa_t联合体内容的赋值,对内存的分配,并返回对象指针




###	解释一下`fastpath` `slowpath`
```
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
```
这两个方法是宏定义,核心为`__builtin_expect(EXP,N)` ,  这是`gcc`提供的指令,意思为EXP==N的概率很大


也就说`fastpath`中x的值大概率为真,`slowpath`中x的值大概率为假

## new方法初始化过程
```
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

这样看来new方式和alloc+init方式在本质上并没有什么不同

## 总结一下
- 对象初始化的核心过程为根据`objc_class`的状态开辟空间,并初始化isa_t内容,随后为对象分配内存并返回指向此内存区域的指针.
- OC中的对象大小因为CF系统的内存对齐限制,所有对象默认大于等于16bytes
- 在isa_t初始化过程中使用了位运算进行性能提升,可以在日常开发中应用
- `fastpath` `slowpath`应用`gcc`指令`__builtin_expect(EXP,N)` ,通过此指令优化编译器在编译时的代码布局,减少指令跳转带来的性能消耗.
