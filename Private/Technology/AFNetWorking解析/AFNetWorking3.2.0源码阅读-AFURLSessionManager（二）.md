# AFNetWorking3.2.0源码阅读-AFURLSessionManager（二）

AFURLSessionManager.m 文件内容解析

## Define

```

static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });

    return af_url_session_manager_creation_queue;
}

static void url_session_manager_create_task_safely(dispatch_block_t block) {
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
        // Fix of bug
        // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
        // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}
```

这两个方法是用C语言的写法封装了一个用来解决在iOS 8及之前task执行block时崩溃的问题。


```
static dispatch_queue_t url_session_manager_processing_queue() {
    static dispatch_queue_t af_url_session_manager_processing_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_processing_queue = dispatch_queue_create("com.alamofire.networking.session.manager.processing", DISPATCH_QUEUE_CONCURRENT);
    });

    return af_url_session_manager_processing_queue;
}
```
进度队列，这个队列用在请求结果返回后处理时，保证在处理返回结果的时候时是异步的。


```
static dispatch_group_t url_session_manager_completion_group() {
    static dispatch_group_t af_url_session_manager_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_completion_group = dispatch_group_create();
    });

    return af_url_session_manager_completion_group;
}
```
请求完成处理事务组，将请求完成后的结果（无论成功还是失败）回调放在 `url_session_manager_completion_group` 中的 指定队列或者主队列中去做，对应结果的通知也是在这里边做的。

`dispatch_group_t` 一般我们是用来操作一组相关的任务，在其中做任务之间同步和数据传递操作，还可以方便我们管理一组任务的状态，但是他在这里并没有做任何关于同步的动作，只是单纯添加进去了，我们暂且猜测他会在其他地方会用到这些功能，或者更有深意。
<br>
<br>

## AFURLSessionManagerTaskDelegate

```
<NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate>
```

这个类从名字和遵守的协议我们很容易得知它是一个网络请求的代理方法处理类

### Property

```
@property (nonatomic, weak) AFURLSessionManager *manager;
```
被代理的`manger`，`weak`防止循环引用

```
@property (nonatomic, strong) NSMutableData *mutableData;
```
在`NSURLSessionTaskDelegate`的`URLSession:task:didCompleteWithError:`方法中拼接下载的数据。


```
// 上传进度
@property (nonatomic, strong) NSProgress *uploadProgress;
// 下载进度
@property (nonatomic, strong) NSProgress *downloadProgress;
// 下载文件存储路径
@property (nonatomic, copy) NSURL *downloadFileURL;
// 完成下载回调
@property (nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
// 上传进度回调
@property (nonatomic, copy) AFURLSessionTaskProgressBlock uploadProgressBlock;
// 下载进度回调
@property (nonatomic, copy) AFURLSessionTaskProgressBlock downloadProgressBlock;
// 请求完成回调
@property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler;
```

<br>
<br>
## Life Cycle Method 

### Init

```

- (instancetype)initWithTask:(NSURLSessionTask *)task {
    self = [super init];
    if (!self) {
        return nil;
    }
     
    _mutableData = [NSMutableData data];
    _uploadProgress = [[NSProgress alloc] initWithParent:nil userInfo:nil];
    _downloadProgress = [[NSProgress alloc] initWithParent:nil userInfo:nil];
    
    __weak __typeof__(task) weakTask = task;
    for (NSProgress *progress in @[ _uploadProgress, _downloadProgress ])
    {
        progress.totalUnitCount = NSURLSessionTransferSizeUnknown;
        progress.cancellable = YES;
        progress.cancellationHandler = ^{
            [weakTask cancel];
        };
        progress.pausable = YES;
        progress.pausingHandler = ^{
            [weakTask suspend];
        };
#if AF_CAN_USE_AT_AVAILABLE
        if (@available(iOS 9, macOS 10.11, *))
#else
        if ([progress respondsToSelector:@selector(setResumingHandler:)])
#endif
        {
            progress.resumingHandler = ^{
                [weakTask resume];
            };
        }
        
        [progress addObserver:self
                   forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                      options:NSKeyValueObservingOptionNew
                      context:NULL];
    }
    return self;
}
```

初始化方法功能：
- `_mutableData` `_uploadProgress` `_downloadProgress`初始化。
- `_uploadProgress` `_downloadProgress` 添加`resumingHandler` `pausingHandler`block。
- `_uploadProgress` `_downloadProgress`  添加对 `fractionCompleted` key值变化的监听并反馈到`downloadProgressBlock` `uploadProgressBlock`方法中。

## NSURLSessionTaskDelegate

```
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
```
在这个代理方法中处理了请求响应

注意点：
- 1.使用`userInfo`可变字典存储`responseSerializer` `downloadFileURL` `mutableData` `error` `responseObject` `serializationError` , 并将其在错误/成功的时候发出`AFNetworkingTaskDidCompleteNotification`通知
- 2.在错误/成功的回调中使用`dispatch_group_async` 在指定或者默认的组中将处理放入指定或者主队列做操作集中处理，
- 3,在调用完成回调`completionHandler`block后，在主线程中启用异步操作发送`1`中的userInfo到`AFNetworkingTaskDidCompleteNotification`通知
<br>
<br>

## NSURLSessionDataDelegate

```
// 收到data回调
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    self.downloadProgress.totalUnitCount = dataTask.countOfBytesExpectedToReceive;
    self.downloadProgress.completedUnitCount = dataTask.countOfBytesReceived;

    [self.mutableData appendData:data];
}
```

- 1,在这个方法中更新`downloadProgress`的`totalUnitCount` `completedUnitCount`，更新时会改变其`fractionCompleted`,因在`manager`初始化中对`fractionCompleted`做了KVO，所以会触发NSProgress Tracking。
- 2,返回数据拼接到`self.mutableData`。

```
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend{
    
    self.uploadProgress.totalUnitCount = task.countOfBytesExpectedToSend;
    self.uploadProgress.completedUnitCount = task.countOfBytesSent;
}
```

上传数据的进度更新，套路同上一个方法
<br>
<br>

## NSURLSessionDownloadDelegate

```
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite{
    
    self.downloadProgress.totalUnitCount = totalBytesExpectedToWrite;
    self.downloadProgress.completedUnitCount = totalBytesWritten;
}
```
更新下载进度，通过通知调用进度的回调，套路同上


```

- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes{
    
    self.downloadProgress.totalUnitCount = expectedTotalBytes;
    self.downloadProgress.completedUnitCount = fileOffset;
}
```
更新重新（断电续传）下载进度，通过通知调用进度的回调，套路同上


```
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    self.downloadFileURL = nil;

    if (self.downloadTaskDidFinishDownloading) {
        self.downloadFileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (self.downloadFileURL) {
            NSError *fileManagerError = nil;

            if (![[NSFileManager defaultManager] moveItemAtURL:location toURL:self.downloadFileURL error:&fileManagerError]) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:fileManagerError.userInfo];
            }
        }
    }
}
```

完成下载回调
- 1,`downloadTaskDidFinishDownloading`block实现的情况下，通过它获取到下载的downloadFileURL。
- 2,`downloadFileURL`获取成功将临时文件移动到downloadFileURL目录中。
- 3,如果文件移动成功发送`AFURLSessionDownloadTaskDidFailToMoveFileNotification`通知，将当前下载的downloadTask 作为Object，`fileManagerError.userInfo`作为userInfo发送出去。

## _AFURLSessionTaskSwizzling

这里因为!(这个bug)[https://github.com/AFNetworking/AFNetworking/pull/2702]
做了方法置换的处理，大体的原因时因为在iOS7，iOS8的时候NSURLSession的结构发生了变化

## AFURLSessionManager @interface()

### @property
属性内容和.h文件中的Set方法基本一致，不需过多解释
在这我们可以看到一个写法：
我们常见的一种场景
在`@interface`中对外暴露一个属性的只读属性，但是在`@implementation`中需要对其进行更改。

AFN中的写法是在`@interface`中用`(readonly)`修饰，在`Extension`中使用`readwrite`修饰
官方文档上也是这么建议的 !(Use Class Extensions to Hide Private Information)[https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html#//apple_ref/doc/uid/TP40011210-CH6-SW3]
<br>
当然我们还有其他的方式
比如直接在`@implementation`使用`_iVar`进行更改操作
还可以使用 `@synthesize someVar = _someVar`
// TODO:1,readonly 和 readwrite修饰符做了什么，使用readonly的时候有没有产生set方法？
// 		2,为什么`@synthesize someVar = _someVar`之后就可以更改被readonly修饰的属性？

## AFURLSessionManager @implementation



### Init 
```
// 重写init，调用指定初始化方法`initWithSessionConfiguration`
- (instancetype)init {
    return [self initWithSessionConfiguration:nil];
}

// 真正的初始化方法
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

	// 默认configuration设置
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

 	 // 操作队列初始化，默认最大并发数为1
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    // session 初始化，使用刚刚初始化的操作队列和sessionConfiguration
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
	  
	 // 默认的响应格式化器为JSON
    self.responseSerializer = [AFJSONResponseSerializer serializer];
	 // 默认的加密设置
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];
	 // 非WATCH环境设置网络监听
#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif
	  // 设置 Task和代理关系的记录dictionary
    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];
    
    // 初始化锁
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;
	  
	  // 将当前session中的所有task设置对应的代理
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}

```
<br>
### dealloc

```
// 移除所有的通知监听，释放对象
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

```
<br>
### addDelegateFor...Task
给data，upload，download task 设置代理等操作

```
// 设置data task
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}

- (void)addDelegateForUploadTask:(NSURLSessionUploadTask *)uploadTask
                        progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
               completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    uploadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:uploadTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
}

- (void)addDelegateForDownloadTask:(NSURLSessionDownloadTask *)downloadTask
                          progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                       destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                 completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    if (destination) {
        delegate.downloadTaskDidFinishDownloading = ^NSURL * (NSURLSession * __unused session, NSURLSessionDownloadTask *task, NSURL *location) {
            return destination(location, task.response);
        };
    }

    downloadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:downloadTask];

    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

这三个方法大体步骤是一样的，无非就是设置各种回调block，方法代理等
有几个点我们需要注意：
<br>
####`delegate.manager = self;`
这里delegate 是一个 `AFURLSessionManagerTaskDelegate` 的局部变量，delegate.manager是一个`AFURLSessionManager`类型的对象，在`AFURLSessionManagerTaskDelegate`class中使用weak关键字描述，是为了循环引用，因为`delegate.manager = self;`中self就是`AFURLSessionManager`类型的对象。
<br>
#### `dataTask.taskDescription = self.taskDescriptionForSessionTasks;`
`.taskDescriptionForSessionTasks`是一个get方法
```

- (NSString *)taskDescriptionForSessionTasks {
    return [NSString stringWithFormat:@"%p", self];
}

```
格式控制符“%p”中的p是pointer（指针）的缩写,所以给task设置的描述是本类对象为一的指针值
<br>
#### `    [self setDelegate:delegate forTask:dataTask];`
这个方法的实现：
```

- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
	 // 判空
    NSParameterAssert(task);
    NSParameterAssert(delegate);

   // 锁
    [self.lock lock];
    
   // 此字典记录所有task和他的代理（delegate，一个AFURLSessionManagerTaskDelegate类对象）的对应关系，是一个key-value的格式
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    
   // 设置进度相关的操作
    [delegate setupProgressForTask:task];
    
   // 添加启动和暂停的监听
    [self addNotificationObserverForTask:task];
    
   // 开锁
    [self.lock unlock];
}

```

## 启动/暂停
```

- (void)taskDidResume:(NSNotification *)notification {
    NSURLSessionTask *task = notification.object;
    
   // 是否有	`taskDescription`方法
    if ([task respondsToSelector:@selector(taskDescription)]) {
    		
    		// 如果有，是否是当前这个对象
        if ([task.taskDescription isEqualToString:self.taskDescriptionForSessionTasks]) {
        		
        		 // 如果是，发送task resume 的通知
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidResumeNotification object:task];
            });
        }
    }
}

- (void)taskDidSuspend:(NSNotification *)notification {
    NSURLSessionTask *task = notification.object;
    
    // 是否有`taskDescription`方法
    if ([task respondsToSelector:@selector(taskDescription)]) {
        
        // 如果有，是否是当前这个对象
        if ([task.taskDescription isEqualToString:self.taskDescriptionForSessionTasks]) {
            
            // 如果是，发送task suspend 的通知
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidSuspendNotification object:task];
            });
        }
    }
}
```

## get/remove Delegete For Task

```
// 获取task对应的delegate ， 是 `- setDelegate:foTask:` 方法的你操作，也会加锁，因为在并发情况下，可能会出现数据同步问题
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock];
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)];
    [self.lock unlock];

    return delegate;
}
```

```

// 删除task的delegate
- (void)removeDelegateForTask:(NSURLSessionTask *)task {
    // 判空
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    [self.lock lock];
    
    // 清除进度回调
    [delegate cleanUpProgressForTask:task];
    // 清除监听
    [self removeNotificationObserverForTask:task];
    // 在记录表中清除task的记录
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
}
```

## 获取各种tasks

```
- (NSArray *)tasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)dataTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)uploadTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

- (NSArray *)downloadTasks {
    return [self tasksForKeyPath:NSStringFromSelector(_cmd)];
}

```
不难看出，核心方法是 `-tasksForKeyPath:`
```

- (NSArray *)tasksForKeyPath:(NSString *)keyPath {
    __block NSArray *tasks = nil;
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return tasks;
}
```

这里需要我们注意的是用到了dispatch的信号量，这是因为`getTasksWithCompletionHandler`是一个异步操作:[官方文档](https://developer.apple.com/documentation/foundation/nsurlsession/1411578-gettaskswithcompletionhandler)

## invalidateSeesionCancelingTasks:

```
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks {
    dispatch_async(dispatch_get_main_queue(), ^{
        if (cancelPendingTasks) {
        		 // 立即取消所有任务
            [self.session invalidateAndCancel];
        } else {
        		 // 立即returns，但是并不会回影响正在执行的任务，它们会继续执行到结束
            [self.session finishTasksAndInvalidate];
        }
    });
}
```
// TODO：这里有个放在主线程操作的代码，不知道是为了什么着想，想必是为了安全，有大牛来给解答一下？

## Send Request

### data task request

```
// interface
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    return [self dataTaskWithRequest:request uploadProgress:nil downloadProgress:nil completionHandler:completionHandler];
}
```

```

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {
	
    __block NSURLSessionDataTask *dataTask = nil;
    
    // 解决iOS 8 及其以下版本的bug
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

`__block` 修饰符的背后究竟发生了什么，为什么用它修饰就可以在block中更改？
为什么不用它修饰就不能在block中更改?,这个问题我在之前的blog中写过，请看：[Block截获自动变量实现与__block修饰符内部实现](https://blog.csdn.net/wxs0124/article/details/65635664)

## upload task request
### `uploadTaskWithRequest:fromFile:`

```

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionUploadTask *uploadTask = nil;
    url_session_manager_create_task_safely(^{
        uploadTask = [self.session uploadTaskWithRequest:request fromFile:fileURL];
    });
	
	// uploadTask 为 nil， 并且 self.attemptsToRecreateUploadTasksForBackgroundSessions == YES， 并且 self.session.configuration.identifier存在的情况下
	// 尝试重新创建uploadTask，最多尝试3次
	// 这个操作是因为在iOS7以下版本会出现background情况下upload有时会变成nil
    if (!uploadTask && self.attemptsToRecreateUploadTasksForBackgroundSessions && self.session.configuration.identifier) {
        for (NSUInteger attempts = 0; !uploadTask && attempts < AFMaximumNumberOfAttemptsToRecreateBackgroundSessionUploadTask; attempts++) {
            uploadTask = [self.session uploadTaskWithRequest:request fromFile:fileURL];
        }
    }

    [self addDelegateForUploadTask:uploadTask progress:uploadProgressBlock completionHandler:completionHandler];

    return uploadTask;
}

```

```

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(NSData *)bodyData
                                         progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
                                

- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request
                                                 progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                        completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
```
对data，和stream的请求我们不用详细探究

## download task request
```

- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                          destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
                                    
                                    
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                             destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                       completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
```
download task和upload task操作几乎一摸一样

## get upload/download progress for task

```
- (NSProgress *)uploadProgressForTask:(NSURLSessionTask *)task {
    return [[self delegateForTask:task] uploadProgress];
}

- (NSProgress *)downloadProgressForTask:(NSURLSessionTask *)task {
    return [[self delegateForTask:task] downloadProgress];
}

```
看明白`delegateForTask:`方法即可

## NSObject
```
// 重载description方法便于在NSLog的时候获取更具像化的信息
- (NSString *)description {
    return [NSString stringWithFormat:@"<%@: %p, session: %@, operationQueue: %@>", NSStringFromClass([self class]), self, self.session, self.operationQueue];
}
```

```
// 重载是否可以响应selector的判断方法，让本类对象在没有实现对应block时可以将方法响应继续向下传递，而不是阻断。
- (BOOL)respondsToSelector:(SEL)selector {
    if (selector == @selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:)) {
        return self.taskWillPerformHTTPRedirection != nil;
    } else if (selector == @selector(URLSession:dataTask:didReceiveResponse:completionHandler:)) {
        return self.dataTaskDidReceiveResponse != nil;
    } else if (selector == @selector(URLSession:dataTask:willCacheResponse:completionHandler:)) {
        return self.dataTaskWillCacheResponse != nil;
    } else if (selector == @selector(URLSessionDidFinishEventsForBackgroundURLSession:)) {
        return self.didFinishEventsForBackgroundURLSession != nil;
    }

    return [[self class] instancesRespondToSelector:selector];
}

```

## NSURLSessionDelegate
```
// 发生错误回调
- (void)URLSession:(NSURLSession *)session
didBecomeInvalidWithError:(NSError *)error
{
    if (self.sessionDidBecomeInvalid) {
        self.sessionDidBecomeInvalid(session, error);
    }
	  // 发送错误通知
    [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDidInvalidateNotification object:session];
}

// 发生身份验证回调
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;
		
	 // 如果实现了对身份验证的block就执行
    if (self.sessionDidReceiveAuthenticationChallenge) {
    		// 执行结果
        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
    } else {
    
    		// 如果验证方式是 NSURLAuthenticationMethodServerTrust ，
    		// @abstract SecTrustRef validation required.  Applies to any protocol.
    		// 这里的验证方式还有很多，请看 NSURLProtectionSpace.h 中的定义
    		// [了解更多看这里](https://draveness.me/afnetworking5)
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        
        		 // 如果我们信任这个host
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
            
            	//获取证书
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
                
                // 如果有证书
                if (credential) {
                		// 指定的证书，可以为nil
                    disposition = NSURLSessionAuthChallengeUseCredential;
                } else { 如果没有证书
                	   // 使用默认的处理，如果这个代理方法没有实现，将会忽略这个证书参数
                    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
                }
            } else {
            		// 拒绝
                disposition = NSURLSessionAuthChallengeRejectProtectionSpace;
            }
        } else {
        		 // 使用默认的处理，如果这个代理方法没有实现，将会忽略这个证书参数 
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }
	  
	  // 完成身份验证后的回调block
    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}

```

## NSURLSessionTaskDelegate

```
// 处理重定向操作，
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
willPerformHTTPRedirection:(NSHTTPURLResponse *)response
        newRequest:(NSURLRequest *)request
 completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    NSURLRequest *redirectRequest = request;

    if (self.taskWillPerformHTTPRedirection) {
        redirectRequest = self.taskWillPerformHTTPRedirection(session, task, response, request);
    }

    if (completionHandler) {
        completionHandler(redirectRequest);
    }
}

```
[什么是重定向](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Redirections)

```
// 这里和NSURLSessionDelegate 中同样有个认证回调，区别就在于，这里并没有针对获取证书是否成功做disposition的对应处理，看样子是默认证书一定会获取成功，这里稍微有点懵逼，有大神指正没有？
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler；
```

```
请求输入流回调
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
 needNewBodyStream:(void (^)(NSInputStream *bodyStream))completionHandler
{
    NSInputStream *inputStream = nil;
	 
	 // 如果self.taskNeedNewBodyStream实现
	 if (self.taskNeedNewBodyStream) {
    		
        inputStream = self.taskNeedNewBodyStream(session, task);
    } else if (task.originalRequest.HTTPBodyStream && [task.originalRequest.HTTPBodyStream conformsToProtocol:@protocol(NSCopying)]) {
    
    		// 如果task.originalRequest.HTTPBodyStream存在 ，并且遵循了NSCopying协议
    		// 因为HTTPBodyStream不是线程安全的，所以在使用的时候需要copy
        inputStream = [task.originalRequest.HTTPBodyStream copy];
    }

    if (completionHandler) {
        completionHandler(inputStream);
    }
}

```

```
// 已经发送的数据回调
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{

    int64_t totalUnitCount = totalBytesExpectedToSend;
    if(totalUnitCount == NSURLSessionTransferSizeUnknown) {
    		// 用KVC的方式获取内容长度
        NSString *contentLength = [task.originalRequest valueForHTTPHeaderField:@"Content-Length"];
        if(contentLength) {
            totalUnitCount = (int64_t) [contentLength longLongValue];
        }
    }

    if (self.taskDidSendBodyData) {
        self.taskDidSendBodyData(session, task, bytesSent, totalBytesSent, totalUnitCount);
    }
}
```

```
// 完成
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // delegate may be nil when completing a task in the background
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }
}
```

## NSURLSessionDataDelegate

```
// 完成
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;

    if (self.dataTaskDidReceiveResponse) {
        disposition = self.dataTaskDidReceiveResponse(session, dataTask, response);
    }

    if (completionHandler) {
        completionHandler(disposition);
    }
}

```

```
// Tells the delegate that the data task was changed to a download task.

// 告诉代理，data task 变成 download task
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    // 所对这个地方需要把data task 移除，重新按照download task重新对代理进行配置
    if (delegate) {
        [self removeDelegateForTask:dataTask];
        [self setDelegate:delegate forTask:downloadTask];
    }

    if (self.dataTaskDidBecomeDownloadTask) {
        self.dataTaskDidBecomeDownloadTask(session, dataTask, downloadTask);
    }
}

```

```
// 询问代理是否在缓存中响应请求
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler
{
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (self.dataTaskWillCacheResponse) {
        cachedResponse = self.dataTaskWillCacheResponse(session, dataTask, proposedResponse);
    }

    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}
```

```
// 后台session完成任务回调
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    if (self.didFinishEventsForBackgroundURLSession) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.didFinishEventsForBackgroundURLSession(session);
        });
    }
}
```

## NSURLSessionDownloadDelegate

```

- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
    
    // 如果有这个block
    if (self.downloadTaskDidFinishDownloading) {
        
        // 获取这个请求对应的文件路径
        NSURL *fileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (fileURL) {
        
        		 // 如果获取到了fileURL
            delegate.downloadFileURL = fileURL;
            NSError *error = nil;
            
            // 将下载的临时数据移动到fileURL
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:fileURL error:&error];
            if (error) {
            		// 如果移动出错，发送AFURLSessionDownloadTaskDidFailToMoveFileNotification通知
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:error.userInfo];
            }
				 // 如果此过程完成，则不在继续往下进行
            return;
        }
    }
	 
    if (delegate) {
        [delegate URLSession:session downloadTask:downloadTask didFinishDownloadingToURL:location];
    }
}

```

```
// 下载过程中多次回调，告诉代理下载进度
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    if (self.downloadTaskDidWriteData) {
        self.downloadTaskDidWriteData(session, downloadTask, bytesWritten, totalBytesWritten, totalBytesExpectedToWrite);
    }
}
```

```
// 告诉代理重新下载的开始位置和预期总字节数，断点续传的核心方法
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes
{
    if (self.downloadTaskDidResume) {
        self.downloadTaskDidResume(session, downloadTask, fileOffset, expectedTotalBytes);
    }
}

```

## NSSecureCoding 
这是一个继承自NSCoding协议的协议

他的作用就是更安全的实现decode，encode

[官方文档](https://developer.apple.com/documentation/foundation/nssecurecoding)

