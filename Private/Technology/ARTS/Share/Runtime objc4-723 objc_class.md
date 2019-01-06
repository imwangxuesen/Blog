# Runtime objc4-723 objc_class


前情提要：
runtime的源码版本: objc4-723
时间：2019-01-06


# `objc_class`
```
struct objc_class : objc_object {
// Class ISA;
Class superclass;
cache_t cache;             // formerly cache pointer and vtable
class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

class_rw_t *data() { 
return bits.data();
}

...
}
```

## `Class superclass `

指向父类的指针
这里可以联想到一个经典的问题，isa指向什么？



复习一下：

`Class`是`objc_object`类型，在`objc_object`中只有一个`isa`指针


`superclass`是一个`Class`类型的指针. 每个实例对象有个`isa`的指针,他指向对象的类，而`Class`里也有个`isa`的指针, 指向`meteClass`(元类)。

**回忆一个小知识点：实例方法存放在类对象中，类方法存放在类对象的类中，即元类中。**

元类保存了类方法的列表。当类方法被调用时，先会从本身查找类方法的实现，如果没有，元类会向他父类查找该方法。同时注意的是：元类（meteClass）也是类，它也是对象。元类也有isa指针,它的isa指针最终指向的是一个根元类(root meteClass).根元类的isa指针指向本身，这样形成了一个封闭的内循环。

一张经典的图片解释：(图片为转载，详情见水印)
![isa](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1546694545675&di=df37f8e0c61cd2d9465270dc3d8af6de&imgtype=0&src=http%3A%2F%2Faliyunzixunbucket.oss-cn-beijing.aliyuncs.com%2Fjpg%2Ffeeb5005f1cfde5f671ad391d67190a0.jpg%3Fx-oss-process%3Dimage%2Fresize%2Cp_100%2Fauto-orient%2C1%2Fquality%2Cq_90%2Fformat%2Cjpg%2Fwatermark%2Cimage_eXVuY2VzaGk%3D%2Ct_100)


## `cache_t cache`

```
struct cache_t {
// 缓存数组
struct bucket_t *_buckets;
// 
mask_t _mask;
// 占用大小
mask_t _occupied;
...
}
```

在很多资料里我们也见过关于类中的方法缓存的介绍，这个`cache`就是方法缓存的结构。

`struct bucket_t *_buckets;` 标示它是一个`bucket_t`为单位的链表结构（数组）

```
struct bucket_t {
private:
//typedef uintptr_t cache_key_t;
cache_key_t _key;
IMP _imp;
...
}
```
不难看出它可以通过`key`快速的找到`Method`对应的`IMP`以实现快速查找。

## `class_rw_t`

与之对应的还有一个`class_ro_t`，`class_rw_t`中包含`class_ro_t`
`class_ro_t` 在编译期产生，它是类中的可读信息。
`class_rw_t` 在运行时产生，它是类中的可读写信息。

```

struct class_rw_t {
// Be warned that Symbolication knows the layout of this structure.
// 标记
uint32_t flags;
// 版本
uint32_t version;

// 类中的只读信息
const class_ro_t *ro;

// 方法列表
method_array_t methods;
// 属性列表
property_array_t properties;
// 协议列表
protocol_array_t protocols;

// 第一个子类
Class firstSubclass;
// 下一个相同父类的类
Class nextSiblingClass;

// 以上这两个属性将某个类的子类串联成一个列表，所以，我们可以通过class_rw_t 获取到当前类的所有子类
// superClass.firstSubclass -> subClass1.nextSiblingClass -> subClass2.nextSiblingClass -> ...

char *demangledName;
}
```

## `class_ro_t`

```

struct class_ro_t {
uint32_t flags;
// 开始位置
uint32_t instanceStart;
// 所占空间大小
uint32_t instanceSize;
#ifdef __LP64__
uint32_t reserved;
#endif
// ivar 布局， 在编译期这里就固定了，它标示ivars的内存布局，在运行时不能改变，这也是为什么我们在运行时不能动态给类添加成员变量的原因
const uint8_t * ivarLayout;

// 名字
const char * name;

// 方法列表
method_list_t * baseMethodList;

// 协议列表
protocol_list_t * baseProtocols;

// 成员变量列表
const ivar_list_t * ivars;

// weak 成员变量的内存布局
const uint8_t * weakIvarLayout;

// 属性列表
property_list_t *baseProperties;

method_list_t *baseMethods() const {
return baseMethodList;
}
};

```

## `class_data_bits_t bits`;

通过

```
class_rw_t *data() { 
return bits.data();
}

//  class_data_bits_t  data（）
class_rw_t* data() {
return (class_rw_t *)(bits & FAST_DATA_MASK);
}

```

以及它的结构体实现，可以看出，他是class_rw_t的一个封装，作用是可以通过它来操作、获取class_rw_t的信息。

## 概括一下class的结构
![](https://github.com/imwangxuesen/Blog/blob/master/Private/temp/timg.jpeg?raw=true)


**参考文献：**

[iOS开发之Runtime学笔记](https://www.jianshu.com/p/6faebc8f492a)


[RuntimeAnalyze](https://github.com/DeveloperErenLiu/RuntimeAnalyze)
