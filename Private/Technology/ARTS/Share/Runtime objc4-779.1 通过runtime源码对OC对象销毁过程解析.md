# Runtime objc4-779.1 通过runtime源码对OC对象销毁过程解析

### Step1 NSObject.mm line 2340
```
// Replaced by NSZombies
- (void)dealloc {
	// Setp2
    _objc_rootDealloc(self);
}
```

### Step2 NSObject.mm line 1814
```
void
_objc_rootDealloc(id obj)
{
// 判断对象是否真是存在,不存在则结束
    ASSERT(obj);
	// Step3
    obj->rootDealloc();
}

```

### Step3 objc-object.h line 433

```
inline void
objc_object::rootDealloc()
{
	// 如果是tagged pointer 则结束, 因为它不按照对象的内存管理来,仅仅是一个指针,指针中含有对象相关的信息,包含它的值, 如果不知道什么个tagged pointer 可以看之前的博文 https://blog.csdn.net/wxs0124/article/details/82712478
    if (isTaggedPointer()) return;  // fixme necessary?

	// 如果对象已成创建isa_t(初始化),没有被weak修饰,没有关联其他对象,没有实现C++析构方法,没有因引用计数过多而存储在sidetable中
    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        //判定本对象不存在于sidetable中
        assert(!sidetable_present());
        //释放
        free(this);
    } 
    else {
    	// 此步需要进一步解析
    	// Step4
        object_dispose((id)this);
    }
}

```

### Step4 objc-runtime-new.mm line 7564

```
id 
object_dispose(id obj)
{
    if (!obj) return nil;
	// 处理c++析构过程,移除对象关联,clear
	// Step5
    objc_destructInstance(obj);    
    free(obj);

    return nil;
}

```

### Step5 objc-runtime-new.mm line 7542
```
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        // 是否有c++/OC析构函数
        bool cxx = obj->hasCxxDtor();
        // 是否有其他关联
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        // 进行析构,详见 Step6
        if (cxx) object_cxxDestruct(obj);
        // 移除所有相关的关联对象,详见 Step7
        if (assoc) _object_remove_assocations(obj);
        // 清理内存空间 详见Step8
        obj->clearDeallocating();
    }

    return obj;
}
```

### Step6 objc-class.mm line 463
```
void object_cxxDestruct(id obj)
{
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    // Step6.1
    object_cxxDestructFromClass(obj, obj->ISA());
}
```

### 6.1 objc-class.mm line 437 
```
static void object_cxxDestructFromClass(id obj, Class cls)
{
    void (*dtor)(id);

    // Call cls's dtor first, then superclasses's dtors.

	// 从本类开始按照继承链依次便利父类
    for ( ; cls; cls = cls->superclass) {
    
    	// 如果没有实现c++/OC析构方法,则结束
        if (!cls->hasCxxDtor()) return; 
        
        // 在本类中找到析构方法并且记载到缓存中 , 方法实现在objc-class-old.mm line 578 此处不展开
        dtor = (void(*)(id))
            lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
        
        // 因为lookupMethodInClassAndLoadCache只会在本类中寻找方法(不会去父类),找不到则会返回objc_msgForward, _objc_msgForward_impcache是编译器的消息转发标记,代表此方法要走消息转发,如果析构方法在本类中被找到了,则此一定为true,进入代码块,执行代码.
        if (dtor != (void(*)(id))_objc_msgForward_impcache) {
            if (PrintCxxCtors) {
                _objc_inform("CXX: calling C++ destructors for class %s", 
                             cls->nameForLogging());
            }
            (*dtor)(obj);
        }
    }
}
```


### Step7 objc-references.mm line 217
从关联表中移除相关的关联对象
```
void
_object_remove_assocations(id object)
{
    ObjectAssociationMap refs{};

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        // 便利查找到和object相关的map并从其中移除记录
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            refs.swap(i->second);
            associations.erase(i);
        }
    }

    // release everything (outside of the lock).
    for (auto &i: refs) {
        i.second.releaseHeldValue();
    }
}

```

这里的代码实在associationHashMap中寻找当前对象相关关联,并且擦出
association是一块相对独立且重要的知识点.牵扯关联表的存储格式,存取对象方式,流程,内存管理等,我们会在后边的博客中详细探讨.此处不再展开

### Step8 objc-object.h line 417
```

inline void 
objc_object::clearDeallocating()
{
	// 如果是只有isa指针,没有优化成isa_t
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
    // 如果没有被弱引用,或者 在sidetable中查不到(意味着引用计数没有太大,在isa_t中存储)
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}

```

### Step8.1 NSObject.mm line 1552

清空在sidetable(用于存储引用计数过多导致isa存不下的对象的应用计数信息)的数据
```
void 
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}

```

### Step8.2 NSObject.mm line 1212
```
// Slow path of clearDeallocating() 
// for objects with nonpointer isa
// that were ever weakly referenced 
// or whose retain count ever overflowed to the side table.
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    ASSERT(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}
```

总结一下:

- 释放对象的核心方法为`void *objc_destructInstance(id obj) `
- 核心流程为 1, 沿着继承链调用c++/OC析构函数.2,移除与其相关联的对象记录.weak指针置为nil.3,移除weak修饰的对象在weak_table的记录并更新其引用计数数据.
- 万变不离其宗,其中知识点依旧是老生常谈的AssociationsHashMap,weak,引用计数.我们逐一击破