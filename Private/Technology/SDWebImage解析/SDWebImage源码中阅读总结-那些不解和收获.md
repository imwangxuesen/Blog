# SDWebImage源码中阅读总结|那些不解和收获

## 图片怎么加载出来的？

|流程编号|关键代码|代码位置|描述|附加补充|
|---|---|---|---|---|
|code_1|sd_setImageWithURL:placeholderImage:|UIImageView+WebCache.h_line:64|入口代码，不多解释|N|
|code_2|sd_internalSetImageWithURL:(nullable NSURL *)url <br> placeholderImage:(nullable UIImage *)placeholder <br> options:(SDWebImageOptions)options <br> operationKey:(nullable NSString *)operationKey <br> setImageBlock:(nullable SDSetImageBlock)setImageBlock <br> progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock <br> completed:(nullable SDExternalCompletionBlock)completedBlock <br> context:(nullable NSDictionary<NSString *, id> *)context  | UIView+WebCache.m_line:55|所有形式的入口代码都汇总到这个方法，隐藏的入口函数|N|
|code_3| NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);|UIView+WebCache.m_line:65|获取任务标记，一般operationKey都为空，所以，会被默认置为当前类名的字符串|此处用到了Runtime的关联|
|code_4|[self sd_cancelImageLoadOperationWithKey:validOperationKey]|UIView+WebCache.m_line:70|如果当前标记下有正在执行的任务，取消执行|这个方法的实现有很多值得我们学习的地方|
|code_4.1|SDOperationsDictionary *operationDictionary = [self sd_operationDictionary]|UIView+WebCacheOperation.m_line:49|获取当前view下关联的任务hash table,其内部实现是通过“loadOperationKey”作为key去获取关联对象，如果获取不到，则创建一个“NSMapTable”类型的任务hash table，这整个过程在@synchronized(self)保护下，线程安全|此处用到了@synchronized()确保线程安全，使用NSMapTable类创建hash table（比NSDictionary好在哪里？）|
|code_4.2|objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);|UIView+WebCacheOperation.m_line:72|将这个image url 关联到view 对象上|再次用到关联|
|code_5|[SDWebImageManager sharedManager];|UIView+WebCacheOperation.m_line:96|获取SDWebImageManager单例，这是下载、查找缓存的核类|此处用到单例确保任务管理的类的唯一性|
|code_6|loadImageWithURL:options: progress:completed:|UIView+WebCacheOperation.m_line:17,SDWebImageManager.m_line:117|开始加载图片的入口函数，会有一个completed的回调|采用block形式的回调，代码清晰易懂|N|
|code_6.1|NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");|SDWebImageManager.m_line:125|如果调用加载函数而没有实现回调block，会被认为是要预加载图片，抛出异常提示使用另外的方法完成预加载|此处使用了NSAssert进行友好的提示|
|code_6.2|SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];|SDWebImageManager.m_line:139|初始化一个综合操作任务|将加载任务实例化，因为一个view，一个imageManager会产生多个任务，这样写易于对任务的管理和阅读|
|code_6.3|isFailedUrl = [self.failedURLs containsObject:url|SDWebImageManager.m_line:148|查看已经失败的记录中是否有这个即将处理的url,再次之后如果options包含SDWebImageRetryFailed会直接调用完成的回调|failedURLs也是一个NSMutableSet类型的集合|
|code_6.4|[self.runningOperations addObject:operation]|SDWebImageManager.m_line:160|将当前的操作任务加入到自身持有的正在执行的记录中，在此句代码前后有两个锁，LOCK(self.runningOperationsLock)，UNLOCK(self.runningOperationsLock)，这两个宏使用GCD的信号量实现加锁。|dispatch_semaphore_wait，dispatch_semaphore_signal配合，实现加锁|
|code_6.5|NSString *key = [self cacheKeyForURL:url]|SDWebImageManager.m_line:164|通过url获取对应的缓存key，里边有个可自定义的过滤方法，如果实现了就会调用，否则就返回url的absoluteString|很多博客写的都是用url的md5值作为缓存的key，在这显然是不对的,需要把内存和磁盘两种缓存分开说,磁盘缓存是有MD5操作的|
|code_6.6|operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:]|UIView+WebCacheOperation.m_line:176|查询缓存，这是一个单独的operation，并且会被当前加载图片的operation引用|这里相当于在一个一部操作中又产生一个异步操作，会有线程同步的问题存在，比如当前加载图片的operation被取消了，但是查询缓存的operation依旧在执行，就会产生问题，处理方法我们往后看|
|code_6.6.1|queryCacheOperationForKey: options: done:|SDImageCache.m_line:514|内部流程：<br>1-key判空。<br>2-查询内存缓存，如果有缓存并且只查询内存缓存就调用done block回调。<br>3-查询磁盘缓存，如果上一步有内存缓存就回调。如果没有内存缓存，但是又磁盘缓存，这个时候就会把磁盘的图片解压，然后放到内存缓存中(默认，如果不想，通过SDImageCacheConfig中的shouldCacheImagesInMemory属性控制)，然后回调.|几个tip:1，内存缓存SDMemoryCache，是NSCache的子类，这么用的优势是什么？2,缓存刷新机制。|
|code_6.7|code_6.6 中的done block 回调做了什么|SDWebImageManager.m_line:180-335|1,当前加载图片operation是否被取消判断。<br>2,判断是否要下载。<br>3,下载使用SDWebImageDownloader执行下载方法并返回一个SDWebImageDownloadToken类型的downloadToken,这里也有一个下载operation的回调处理失败和成功的事件|这里捋一下查询缓存后的大步骤，接下来一步步分析。|
|code_7.1|[self safelyRemoveOperationFromRunning:strongOperation];| SDWebImageManager.m_line:182|当前operation不存在或者被取消,从执行队列中删除当前operation, code_6.4 的反向操作||
|code_7.2|[self.imageDownloader downloadImageWithURL:options:progress:completed:]|SDWebImageManager.m_line:222|开始下载，并将下载operation的token返回，当前加载进程强引用此token,它包含了当前的下载operation，url和用来取消时的token(此token其实是对下载进度和完成回调的一个强引用)||
|code_7.3|operation = [self createDownloaderOperationWithUrl:url options:options];|SDWebImageDownloader.m_line:294|创建下载operation(SDWebImageDownloaderOperation)|1,在这行代码前后都出现了`URLOperations`,它是一个可变字典，用来维护url和operation之间的对应关系，可以说是存储当前正在执行的下载operation<br>2, `SDWebImageDownloaderOperation` 的下载过程？<br>3，`[Array removeObjectIdenticalTo:]` API 的好处|
|code_7.4|对下载完成后的动作解析|SDWebImageManager.m_line:224-335|下载operation的完成回调处理过程：<br>1,如果operation被取消，什么都不做。<br>2,如果出现错误，调用completion block回调错误,并把URL存储起来,用在code_6.3处.<br>3,如果成功,从failedURLs记录中删除当前url(如果有的话).<br>4,如果只刷新缓存，下载图片位空，则什么都不处理，<br>5如果下载成功，并且实现了`imageManager:transformDownloadedImage:withURL:`代理方法，则进行图片转换.<br>6,再如果就只做图片的序列化(如果实现了序列化方法)，缓存到内存、磁盘中.<br>7,完成回调<br>8,线程安全的删除加载图片的operation|这个流程比较长，但是代码比较好理解，没有很高深的地方，需要注意几个tip:<br>1,缓存到内存并且缓存到磁盘（如果options中有SDWebImageCacheMemoryOnly就不会缓存到磁盘）.<br>2,`[Array removeObjectIdenticalTo:]` API 的好处.<br>3, SDWebImageDownloaderOperation 的内部实现解析|
|code_8|截至到code_7.4我们从code_6开始进入的SDWebImageManager加载图片的过程就结束了，下边我们来看加载完成之后的回调操作||||
|code_9|dispatch_main_async_safe(callCompletedBlockClojure);|UIView+WebCache.m_line:138| case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set OR case 1b: we got no image and the SDWebImageDelayPlaceholder is not set|不多解释|
|code_10|SDWebImageNoParamsBlock callCompletedBlockClojure|UIView+WebCache.m_line:124|自动设置图片，刷新当前view|重写了setNeedsLayout方法，在里边区分MAC系统和iPhone系统|


## 不解与收获
### @synchronized同步

在iOS中，这种同步机制是比较慢的。具体原因我们可以看[MrPeak的一篇文章](http://mrpeak.cn/blog/synchronized/)
使用这个同步锁的时候要控制好粒度，尽可能的细，并且要注意被同步函数中嵌套调用函数。

```
@synchronized(self) {
	do something 
}
```
这种传参self的，一定要慎重。因为很有可能这个类外部，也会把它的一个实例变量作为@synchronized的参数，这样就会产生死锁。

### LOCK(lock) UNLOCK(lock)

```
#define LOCK(lock) dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
#define UNLOCK(lock) dispatch_semaphore_signal(lock);

lock：dispatch_semaphore_t
```
这个不用多说，常用的。

### 衍生问题：我们对比一下几种iOS的锁
根据[ibireme写的不再安全的 OSSpinLock一文](https://blog.ibireme.com/)所做的测试如图(图片也来自这篇文章)：
![](https://blog.ibireme.com/wp-content/uploads/2016/01/lock_benchmark.png)

文中也有测试的代码，看了一下，基本来说是比较客观的，所以，我们用锁，最注重一下效率，当然自旋锁正如文中所描述的并不是绝对安全的，所以将其排除，推荐使用dispatch_semaphore

### NSMapTable

这个类的用法几乎和NSDictionary一样,最大的优势在于他可以方便的控制对value对象的强弱引用，而NSDictionary如果想实现弱引用，必须通过`[NSValue valueWithNonretainedObject:]`在做一层转换。

### 由NSMapTable衍生的问题:NSHashTable、NSPointerArray。
和NSMapTable的应用场景相似的还有对应的`NSHashTable `,`NSPointerArray `，同样提供了对象内存管理方式。和我们经常使用几个类型的对应关系是：

| NSSet |->| NSHashTable |
|----|----|----|

| NSArray |->| NSPointerArray |
|----|----|----|

| NSDictionary |->| NSMapTable |
|----|----|----|

在做一些操作封装，比如operation的时候，用这个类型去记录operation的状态是非常方便的，因为可以快速的形成弱引用，这样就不用担心后边的内存释放问题。

### NSAssert

断言，我们就不用过多解释了，温故一下，常用断言有 NSParameterAssert 、 NSAssert 、 NSCAssert 、NSCparameterAssert。
> 注意:在TARGET->Build Setting->ENABLE_NS_ASSERTIONS,可以控制Debug,Release模式下是否生效,千万不要让Release生效,那样线上及其不稳定,当然这个是默认不生效的。

我们来看一下`NSAssert`是怎么定义的

```
#define NSAssert(condition, desc)			\
	__PRAGMA_PUSH_NO_EXTRA_ARG_WARNINGS \
    _NSAssertBody((condition), (desc), 0, 0, 0, 0, 0) \
        __PRAGMA_POP_NO_EXTRA_ARG_WARNINGS
#endif
```
可见，核心是`_NSAssertBody `

```

#define _NSAssertBody(condition, desc, arg1, arg2, arg3, arg4, arg5)	\
    do {						\
	__PRAGMA_PUSH_NO_EXTRA_ARG_WARNINGS \
	if (!(condition)) {				\
            NSString *__assert_file__ = [NSString stringWithUTF8String:__FILE__]; \
            __assert_file__ = __assert_file__ ? __assert_file__ : @"<Unknown File>"; \
	    [[NSAssertionHandler currentHandler] handleFailureInMethod:_cmd object:self file:__assert_file__ \
	    	lineNumber:__LINE__ description:(desc), (arg1), (arg2), (arg3), (arg4), (arg5)];	\
	}						\
        __PRAGMA_POP_NO_EXTRA_ARG_WARNINGS \
    } while(0)
```

1,最外层的do-while(0),这是经典的宏定义写法,[不理解的话看这篇文章：《do{...}while(0)的妙用》作者：IvanRunning](https://www.jianshu.com/p/99efda8dfec9)

2,第二层一个条件非空控制.

3,紧接着获取当前文件的路径，有空提示.

4,调用`NSAssertionHandler`的方法抛出异常.


### NSAssertionHandler

内部就两个方法:

```
// 抛出OC的异常
- (void)handleFailureInMethod:(SEL)selector object:(id)object file:(NSString *)fileName lineNumber:(NSInteger)line description:(nullable NSString *)format,... NS_FORMAT_FUNCTION(5,6);

// 抛出C的异常
- (void)handleFailureInFunction:(NSString *)functionName file:(NSString *)fileName lineNumber:(NSInteger)line description:(nullable NSString *)format,... NS_FORMAT_FUNCTION(4,5);
```

这个类我们也可以用来重写,达到一种既能捕获异常，也可以保证程序正常运行的效果，设想，我们debug的时候，如果代码质量差，一会儿一个crash是不是很恶心。

**继承NSAssertionHandler创建TestAssertionHandler Class**

```
//TestAssertionHandler.m

- (void)handleFailureInMethod:(SEL)selector object:(id)object file:(NSString *)fileName lineNumber:(NSInteger)line description:(nullable NSString *)format,...  {
    NSLog(@"\n 当前方法 %@ \n 当前对象 %@ \n 当前文件路径 %@ \n 代码行数%li", NSStringFromSelector(selector), object, fileName, (long)line);
}

- (void)handleFailureInFunction:(NSString *)functionName file:(NSString *)fileName lineNumber:(NSInteger)line description:(NSString *)format,... {
    NSLog(@"\n 当前方法 (%@)\n  当前文件路径 %@ \n 代码行数%li", functionName, fileName, (long)line);
}

```


**初始化对象，并加入到当前线程**


> 每个线程都有它自己的NSAssertionHandler实例，并且会自动创建。

```
TestAssertionHandler *handler = [[TestAssertionHandler alloc] init];
[[[NSThread currentThread] threadDictionary] setValue:handler forKey:NSAssertionHandlerKey];
```
**TEST**

```
NSString *s = @"2";
NSAssert([s isEqualToString:@"12"], @"string == 123");

Log:

 当前方法 sy: 
 当前对象 <AppDelegate: 0x6000008c1c60> 
 当前文件路径 /Users/WangXuesen/Desktop/TEST/TEST/AppDelegate.m 
 代码行数80
_______________________________
NSParameterAssert(nil);

Log:
 当前方法 sy: 
 当前对象 <AppDelegate: 0x600002a90040> 
 当前文件路径 /Users/WangXuesen/Desktop/TEST/TEST/AppDelegate.m 
 代码行数76
```

### NSCache

1,线程安全<br>
2,内存告警时自动清理<br>
3,可设置最大缓存大小,超过自动回收,最早的最先释放<br>
4,可设置最大缓存对象数量,默认没有限制,超出同上。<br>

### 各种Operation

这里我们学习的主要是思想

1,单一原则，一种operation就专门做一件事情。<br>
2,operation操作完成后注意被取消的情况处理.<br>
3,对operation的管理、缓存.<br>
