关于阅读《Object－C高级编程－iOS与OS X多线程和内存管理》一书后的iOS内存管理系列思考

* [《关于iOS内存管理的规则思考》](http://blog.csdn.net/wxs0124/article/details/53787357)
* [《iOS内存管理——alloc／release／dealloc方法的GNUstep实现与Apple的实现》](http://blog.csdn.net/wxs0124/article/details/53840611)


# 关于iOS内存管理的规则思考

- 自己生成的生成的对象，自己持有。
- 非自己生成的对象，自己也能持有。
- 不在需要自己持有的对象时释放。
- 非自己持有的对象无法释放。

```
注：这里的自己是对象使用的环境，理解为编程人员本身也没有错
```

**对象操作和Objective－C方法对应**

|对象操作			|Objectivew－C方法|
|------			|----------- |
|生成并持有对象	|alloc／copy／mutableCopy／new或以此开头的方法|
|持有对象			|retain|
|释放对象			|release|
|废弃对象			|dealloc|

##自己生成的对象，自己持有
```
//自己生成并持有对象
id obj1 = [[NSObject alloc] init];
        
id obj2 = [NSObject new];
        
id obj3 = [obj2 copy];
```

`copy`方法基于NSCopying方法约定，实现类中的`copyWithZone:`
`mutableCopy`方法基于NSMutableCopying方法约定，实现类中的`mutableCopyWithZone:`


##非自己生成的对象，自己也能持有
用alloc／new／copy／mutableCopy以外的方法取得的对象，自己不是该对象的持有者。

```
//取的非自己生成并持有的对象，
//取得对象的存在，但自己不持有对象。

id obj = [NSMutableArray array];
    
id obj2 = [NSDictionary dictionary];
    
//自己持有对象
[obj retain];
    
[obj2 retain];
```


> 注：这里有点不好理解，我们先来看一段代码：

```
//取的非自己生成并持有的对象，
//取得对象的存在，但自己不持有对象。
    
id unretain_obj = [NSMutableArray array];

NSLog(@"unretain_obj retain count = %lu", (unsigned long)[unretain_obj retainCount]);

    
//调用 release
[unretain_obj release];
```
上述代码，我们打印结果是：
> 2016-12-21 15:32:04.485 acm[65216:852108] unretain_obj retain count = 1

随后调用`release`方法会导致程序崩溃！

按照引用计数来说，这时`unretain_obj`是可以被执行一次release方法的。但是为什么我们直接调用会导致程序崩溃。

我们会想最开始提到的四条思想之一：

**无法释放非自己持有的对象**

这样我们就很好理解了。虽然打印出`unretain_obj`的`retainCount` 为 `1` 但是不能说明是因为它引用了对象。它只是单纯的获取到了对象的存在而已。

> 那么我们会产生一个问题。那么这个对象是谁在持有？？

我们先做一个猜测：
> 因为`[NSMutableArray array]`是一个工厂方法，在`array`肯定是要生成一个`NSMutableArray`实例对象。这时也必然会有一个指针引用它然后返回这个对象。so。。。

先想到这里，后边我们再去印证

我们再来看一段代码：
```
//取的非自己生成并持有的对象，
//取得对象的存在，但自己不持有对象。
id unretain_obj = [NSMutableArray array];

NSLog(@"unretain_obj retain count = %lu", (unsigned long)[unretain_obj retainCount]);
    
//自己持有对象
[unretain_obj retain];

NSLog(@"unretain_obj retain count = %lu", (unsigned long)[unretain_obj retainCount]);

//释放自己持有的对象
[unretain_obj release];

NSLog(@"unretain_obj retain count = %lu", (unsigned long)[unretain_obj retainCount]);
```

打印结果
> 2016-12-21 15:40:20.774 acm[65682:861135] unretain_obj retain count = 1
2016-12-21 15:40:20.774 acm[65682:861135] unretain_obj retain count = 2
2016-12-21 15:40:25.254 acm[65682:861135] unretain_obj retain count = 1

并且程序也不会崩溃。
着也印证了我们上边的想法。
**因为通过`retain`方法，非自己生成的对象跟用alloc／new／copy／mutableCopy方法生成并持有的对象一样，成了自己所持有的**


##不在需要自己持有的对象时释放

通过上边的例子我们知道，自己持有的对象在释放时调用release方法，eg：

```
//自己生成并持有对象
id release_obj = [[NSObject alloc] init];
    
//将自己持有的对象释放
[release_obj release];
    
/* 
 * 释放对象
 * 指向对象的指针依然被保留在变量release_obj 中，你依然可以调用它。
 * 但是对象一经释放绝对不可访问，否则会造成程序崩溃。
 * 出现EXC_BAD_ACCESS Crash问题
 */
```
###我们自己实现一个方法，返回一个方法调用着也可以持有的对象，即alloc的作用
```
- (id)allocObject {
	 //自己生成并持有对象
    id obj = [[NSObject alloc] init];
    //原封不动的返回一个由alloc方法生成的对象
    return obj;
}
```
> 注:方法名符合 `生成并持有对象	alloc／copy／mutableCopy／new或以此开头的方法 ` 规则

###我们自己实现一个方法，返回一个谁也不持有的对象，只是取得对象的存在

```
- (id)object {
	//自己生成并持有对象
    id obj = [[NSObject alloc] init];
    
    //调用autorelease方法 取得对象的存在，但自己不持有对象。
    [obj autorelease];
    
    return obj;
}

```

>`autorelease`方法可以取得对象的存在，但自己不持有对象。使对象在超出指定的生存范围时能够自动的并正确的释放（调用`release`方法）

####autorelease和release方法的区别

**autorelease：**

```flow
st=>start: 调用autorelease
e=>end: object释放
op=>operation: object不立即释放，注册到‘autoreleasepool’中
cond=>condition: pool结束？

st->op->cond
cond(yes)->e
```


**release：**

```flow
st2=>start: 调用release
e2=>end: object被释放
op2=>operation: object立即释放
cond=>condition: ？

st2->op2->cond
cond(yes)->e2
```

autorelease的详细解说我们后边介绍。

我们也可以通过调用`retain`方法来使 `autorelease`方法的来的对象自己持有eg:
```
//获取对象的存在，自己不持有
 id unretain_obj = [NSMutableArray array];
 
 //持有对象
[unretain_obj retain];
```

##无法释放非自己持有的对象
###自己已经释放了还继续释放
```
 	//自己生成并持有对象
    id release_obj = [[NSObject alloc] init];
    
    //将自己持有的对象释放
    [release_obj release];
    
    //释放已经释放的对象
    [release_obj release];

    /*
     * 释放对象
     * 指向对象的指针依然被保留在变量release_obj 中，你依然可以调用它。
     * 但是对象一经释放绝对不可访问，否则会造成程序崩溃。
     * 出现EXC_BAD_ACCESS Crash问题
     */
```

###只获取了对象的存在，试图释放对象
```
	//取的非自己生成并持有的对象，
    //取得对象的存在，但自己不持有对象。
    id unretain_obj = [NSMutableArray array];
    //释放自己不持有的对象
    [unretain_obj release];
```
> 程序崩溃，报EXC_BAD_ACCESS Crash问题

