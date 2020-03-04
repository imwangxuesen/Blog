# Runtime objc4-779.1 OC中,为什么swizzleMethod时要先addMethod?

## 我们swizzleMethod的方法通常如下
```
void swizzleMethod(Class class, SEL originalSelector, SEL swizzledSelector)
{
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
    BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    
    if (didAddMethod) {
        IMP rel =  class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
        
    }
    
}
```
# 今天有同事问我为什么要先`class_addMethod`,直接交换不行么?

## 先说原因

我们的前提时`swizzledSelector`这个`SEL`是真实存在与本`class`的

1. `originalSelector`有可能存在父类(`class_getInstanceMethod`是按照继承链查找方法的)
2. 如果`originalSelector`为父类方法,而本类没有,直接交换带来的后果就是影响了所有父类的事例对象的方法实现,后果就不可控了.
3. 而如果`originalSelector`为父类方法,先调用`class_addMethod`将其添加到本类中,并且其方法实现为`swizzledSelector`的,这里相当于在添加的时候同步进行了方法实现交换.
4. 如果添加成功,则说明`originalSelector`本来不存在于本`class`,那么剩下的就是将`swizzledSelector`所对应的方法实现替换成`originalSelector`的方法实现即可.
5. 如果添加不成功,则说明`originalSelector`存在于本`class`,那么这时交换两个方法的实现是完全基于本类的,所以可控、安全.

## 再看源码
### Step1 class_getInstanceMethod objc-runtime-new.mm line 5800
```
Method class_getInstanceMethod(Class cls, SEL sel)
{
    ...只留关键代码
    return _class_getMethod(cls, sel);
}

static Method _class_getMethod(Class cls, SEL sel)
{
    mutex_locker_t lock(runtimeLock);
    return getMethod_nolock(cls, sel);
}

static method_t *
getMethod_nolock(Class cls, SEL sel)
{
   ...只留核心代码

	// 重点在这!!!! class_getInstanceMethod方法会沿着继承链网上找,如果本类没有,父类有,则找到的是父类方法.
    while (cls  &&  ((m = getMethodNoSuper_nolock(cls, sel))) == nil) {
        cls = cls->superclass;
    }

    return m;
}

```

### class_addMethod 方法是如何实现的? objc-runtime-new.mm line 6603
```
BOOL 
class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    mutex_locker_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}

static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;

   // 只留核心代码

    method_t *m;
    // 查看本类是否有此方法,这里也是个重点,NoSuper也就是不会沿着继承链查找,只在本类进行
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists
        // 如果有,但是不替换
        if (!replace) {
        	// 直接获取结果
            result = m->imp;
        } else {
        	// 如果需要替换,则直接将实现覆盖
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
    	// 如果本类中不存在此方法 
    	// 创建一个新的方法列表,并把此方法属性填充进去
        // fixme optimize
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdupIfMutable(types);
        newlist->first.imp = imp;
		
		//  对新的方法列表的添加做准备
		//  1, 注册方法名
		//  2, 对方法列表进行排序
		//  3, 标记此方法列表唯一,并且有序
        prepareMethodLists(cls, &newlist, 1, NO, NO);
        
        // 添加到现有的方法列表之后
        cls->data()->methods.attachLists(&newlist, 1);
		
		//  刷新缓存
        flushCaches(cls);

        result = nil;
    }

    return result;
}

```

### Step2 class_replaceMethod的实现比较有意思, objc-rutnime-new.mm line 6613

```
IMP 
class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;

    mutex_locker_t lock(runtimeLock);
	// 它的核心就是调用了addMethod 最后一个参数“YES”的意思如果class存在SEL为name的方法,则替换它的实现.这个过程在Step1的源码解析中有.
    return addMethod(cls, name, imp, types ?: "", YES);
}
```

### Step3 method_exchangeImplementations objc-runtime-new.mm line 3886
```

void method_exchangeImplementations(Method m1, Method m2)
{
	// 判空 
    if (!m1  ||  !m2) return;
	
	// 上锁
    mutex_locker_t lock(runtimeLock);

    // 经典指针指向交换
    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;

	// 修复其方法在外部已知的类的生成列表
    // RR/AWZ updates are slow because class is unknown ()
    // Cache updates are slow because class is unknown
    // fixme build list of classes whose Methods are known externally?
	
	// 刷新缓存
    flushCaches(nil);

	// 看方法名是根据方法变更 适配class 中 custom flag状态
	// 内部方法进行了RR/AWZ/CORE 等方法的扫描
    adjustCustomFlagsForMethodChange(nil, m1);
    adjustCustomFlagsForMethodChange(nil, m2);
}

```

```
/***********************************************************************
* Update custom RR and AWZ when a method changes its IMP
**********************************************************************/
static void
adjustCustomFlagsForMethodChange(Class cls, method_t *meth)
{
	// AWZ methods: +alloc / +allocWithZone:
    objc::AWZScanner::scanChangedMethod(cls, meth);
    
    // retain/release/autorelease/retainCount/
	// _tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference
    objc::RRScanner::scanChangedMethod(cls, meth);
	
	// +new, ±class, ±self, ±isKindOfClass:, ±respondsToSelector
    objc::CoreScanner::scanChangedMethod(cls, meth);

	// 最终都调用scanChangedMethod方法,因cls为nil 最终调用 scanChangedMethodForUnknownClass方法
}
```

```
这个方法的调用时机是在scanChangedMethod方法没有传入cls的时候,所以,只针对NSObject和元类进行操作.
static void
    scanChangedMethodForUnknownClass(const method_t *meth)
    {
        Class cls;

        cls = classNSObject();
        if (Domain != Scope::Classes && !isNSObjectSwizzled(NO)) {
            for (const auto &meth2: as_objc_class(cls)->data()->methods) {
                if (meth == &meth2) {
                    setNSObjectSwizzled(cls, NO);
                    break;
                }
            }
        }

        cls = metaclassNSObject();
        if (Domain != Scope::Instances && !isNSObjectSwizzled(YES)) {
            for (const auto &meth2: as_objc_class(cls)->data()->methods) {
                if (meth == &meth2) {
                    setNSObjectSwizzled(cls, YES);
                    break;
                }
            }
        }
    }
```