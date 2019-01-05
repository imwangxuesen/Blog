#Runtime objc4-723 isa_t与isa

前情提要：
runtime的源码版本: objc4-723
时间：2018-11-23

### Runtime 可能并不是你看到的那样

在搜索引擎中搜索“Runtime”

baidu:

![google runtime](https://raw.githubusercontent.com/imwangxuesen/Blog/master/Private/temp/runtime_baidu.png)

google:

![baidu runtime](https://raw.githubusercontent.com/imwangxuesen/Blog/master/Private/temp/runtime_google.png)

这里边第一页的内容，挨个点进去，所有的人都会拿以下代码说事情：

```
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;

```

但是这里边有

```
#if !__OBJC2__  如果不是Objective-C 2.0
OBJC2_UNAVAILABLE; Objective-C 2.0不可用
```

那我们现在是什么版本？Objective-C 2.0啊！！！

2006年7月它就发布了，虽然研究老的代码是有参考意义的，但是这么久远的实现应该被时间消逝的无影无踪了吧。有新的为什么不看？

通过[Opensource](https://opensource.apple.com/tarballs/objc4/)我下载了objc4-723版本，也就是截止2018-11-23最新的runtime开源源码

### 常用结构体的变化

之前的版本定义：

```
// 􏳒􏳓Class􏰋id
typedef struct objc_class *Class; typedef struct objc_object *id;
// 􏳒􏳓􏱅􏰶􏱏􏱐
typedef struct objc_method *Method;
typedef struct objc_ivar *Ivar;
typedef struct objc_category *Category; typedef struct objc_property *objc_property_t;

```

现在的定义：

```
typedef struct objc_class *Class; typedef struct objc_object *id;
typedef struct method_t *Method;
typedef struct ivar_t *Ivar;
typedef struct category_t *Category; typedef struct property_t *objc_property_t;

```

### objc_object

```
// 缩减版代码，public中都是方法，对结构的研究没有啥影响
struct objc_object {
private:
    isa_t isa;
}
```

## objc_object问题

###1，isa_t是个啥？

####1.1 定义
```
// 缩减版代码，我们只留下 arm64下的代码，还有__x86_64__
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;


# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; // 1：64bit 0:32bit
        uintptr_t has_assoc         : 1; // 是否有关联引用或者曾经有关联引用
        uintptr_t has_cxx_dtor      : 1; // 是否有析构函数
        uintptr_t shiftcls          : 33; // 所属类的内存地址，isa的指向的
        MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6; // 是否初始化完成
        uintptr_t weakly_referenced : 1; // 是否被弱引用或者曾经被弱引用
        uintptr_t deallocating      : 1; // 是否被释放中
        uintptr_t has_sidetable_rc  : 1; // 是否引用计数太大超出存储区域
        uintptr_t extra_rc          : 19; // 应用计数
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
#endif
};

```

- 1，这是一个`union`结构体，可以在内部写方法
- 2，cls/ bits/ isa_t三部分
- 3，一个描述性结构体,这是isa_t的核心，用它配合bits可以操作整个对象的内存空间

#### 1.2 作用

- `cls`是一个`Class`类型的，所以通过`isa_t->cls` 就可以获取到当前`object`的类对象，不用担心消息转发的方式有什么问题。

- `bits` 是`uintptr_t`（`unsigned long`别名）类型，在64位系统里，64位，配合一些宏定义做位运算可以方便的获取到对应的信息，比如：在`objc_object `中有个 `Class ISA()`方法，内部实现的核心就是`return (Class)(isa.bits & ISA_MASK);`

- 像`ISA_MASK`这样的宏还有很多，结构中的`nonpointer` `has_assoc ` 等只可以操作独自负责的内存区域，但是`bits`可以操作整个对象的内存区域。


### objc_class

#### 定义

```
//  缩减版代码
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    
    class_rw_t *data() { 
        return bits.data();
    }

```

因为`objc_class`是继承自`objc_object`的，可以通过`ISA()`方法获取到对应的元类地址，这里也不用担心原有的消息转发流程问题。

- `superclass` 根据注释我们可以看到出来它的作用就是原来isa指针的作用，指向父类的指针
- `cache` 是 `cache_t `类型，用来处理已经调用的方法缓存
- `class_data_bits_t` 类型的`bits`是重点,这里边`bits`和前边`isa_t`的作用是一样的
- `class_rw_t`与之对应的还有一个`class_ro_t`，前者是class在运行时状态的操作对象，它里边会持有对应的`class_ro_t`，可读可写。后者是编译时生成的class类型，只读。这两者的详细区别下篇文章解释