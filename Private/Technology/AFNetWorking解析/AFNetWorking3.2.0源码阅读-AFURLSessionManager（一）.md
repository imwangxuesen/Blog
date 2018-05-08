# AFNetWorking3.2.0源码阅读-AFURLSessionManager（一）


```
[TOC]
```

## AFURLSessionManager.h 介绍
 
根据开头注释中的介绍，大体意思就是，这个类是管理整个网络请求的生命周期，绘画调度，代理方法实现，请求发送及其回掉处理的类，可以说她是AFN的核心。

## 属性解析
```
@property (readonly, nonatomic, strong) NSURLSession *session;
```
**回话对象**

```
@property (readonly, nonatomic, strong) NSOperationQueue *operationQueue;

```
**委托执行回调操作队列,也就是说，这个队列中处理的都是回掉代理回调的操作**

```
@property (nonatomic, strong) id <AFURLResponseSerialization> responseSerializer;

```
**响应序列化**

**- 1.根据注释：它的作用就是方便对服务器的响应数据进行序列化的操作。**<br>

**- 2.默认是`AFJSONResponseSerializer` 并且这个属性在使用的时候不能为nil**<br>

**- 3.它必须是遵循`AFURLResponseSerialization`协议的对象，我们可以在文件中看到 `AFURLResponseSerialization.h`中`AFHTTPResponseSerializer` 的定义是：**
	**`@interface AFHTTPResponseSerializer : NSObject <AFURLResponseSerialization>`**
	**所以，这个对象我们可以在后边拜读AFURLResponseSerialization文件的时候详细探究**



##  Getting Session Tasks (获取不同task接口，不细说)

```
///----------------------------
/// @name Getting Session Tasks
///----------------------------

/**
 The data, upload, and download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionTask *> *tasks;

/**
 The data tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDataTask *> *dataTasks;

/**
 The upload tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionUploadTask *> *uploadTasks;

/**
 The download tasks currently run by the managed session.
 */
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDownloadTask *> *downloadTasks;
```

## Managing Callback Queues
```
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;
```
**执行完成时使用的回调队列，可以为空，为空时使用主队列**


####@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;

```
执行完成时使用的Group，可以为空，为空时使用私有的一个Group
```

###Working Around System Bugs

```
@property (nonatomic, assign) BOOL attemptsToRecreateUploadTasksForBackgroundSessions;
```
**在iOS7中 在后台创建 upload task 时可能失败，为nil，如果把这个属性设置为YES ，AFNetWorking会遵循Apple的建议去重新创建。**



## Initialization

```
- (instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)configuration NS_DESIGNATED_INITIALIZER;

```
**- 1.指定的初始化方法**<br>
**- 2.需要一个NSURLSessionConfiguration**<br>
**- 3.NS_DESIGNATED_INITIALIZER 宏代表初始化只能用这一个，这是一个指定的初始化方法**


```
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks;
```
**是否取消这个管理类下的Seesion中 pending 的 task**
<br>
<br>
<br>

##Running Data Tasks

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler 
DEPRECATED_ATTRIBUTE;
```
**- 1.使用DataTask，执行一个HTTP请求** <br>
**- 2.返回系统默认的response**<br>
**- 3.返回根据serializer序列化后的responseObject**<br>
**- 4.返回error** <br>
**- 5.DEPRECATED_ATTRIBUTE 标注为不建议使用，即将废弃**
<br>
<br>

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;
```
**- 1.同上一个方法一样，使用Data Task 发起一个HTTP请求**<br>
**- 2.多了uploadProgressBlock，downloadProgressBlock 回调上传和下载进度** <br>
**- 3.注意，这两个progressBlock并不是在主线程回调的，而是在session所在线程，如果需要UI操作，请放到主线程中**<br>
**- 4.completionHandler同上**<br>
**- 5.这样看来，这个方法可以说是上一个方法的替代 (悄悄的改调自己封装的内容。。。)**

## Running Upload Tasks

```
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;
```

```
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(nullable NSData *)bodyData
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
```

```
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request
                                                 progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                        completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
```

**- 1.以上三个方法分别是本地文件，Http Body，标准文件流 上传方法**<br>
**- 2.HTTP传输过程中虽然都是以二进制的形式在传输数据，但是对于发起方可能存在需要传输的数据本身的差异性，比如它是一个本地文件，一个字符串，甚至直接就是流。**<br>
**- 3.所以，对于着几个方法我们也大可不必纠结为什么这么多形式，暂且把它们理解为对不同类型对象上传的友好封装（可能事实也是这样，需要我们看.m文件探究）**<br>
**- 4.第一个方法中对`attemptsToRecreateUploadTasksForBackgroundSessions`属性做了警告，不知道下边两个方法是否需要，根据该属性的说明，应该是都需要，具体需要实践**

## Running Download Tasks

```
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                          destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
```
**- 1.使用Download Task下载**<br>
**- 2.可以设置对应的文件下载后的存储路径destination**<br>
**- 3.但是，在destination在后台下载的时候是不管用的，因为在后台时，这个方法中的block是不能够回调的，需要使用`-setDownloadTaskDidFinishDownloadingBlock:`指明在后台下载时的文件存储路径**


```
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                             destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                       completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
```
**- 1.简单理解为断点续传/继续下载** <br>
**- 2.`resumeData` 已经下载完成的数据**


##Getting Progress for Tasks

```
- (nullable NSProgress *)uploadProgressForTask:(NSURLSessionTask *)task;
```
**获取当前session task的上传进度**

```
- (nullable NSProgress *)downloadProgressForTask:(NSURLSessionTask *)task;
```
**获取当前session task的下载进度**

##Setting Session Delegate Callbacks

以下方法中设置的block都会在对应的NSURLSessionDelegate方法中执行

> as handled by the `NSURLSessionDelegate` method `URLSession:didBecomeInvalidWithError:`

```
- (void)setSessionDidBecomeInvalidBlock:(nullable void (^)(NSURLSession *session, NSError *error))block;
```

**- 1.设置session失效时的回调block** <br>
**- 2.它的调用是根据`NSURLSessionDelegate`的`URLSession:didBecomeInvalidWithError:`调用**
<br>
<br>

>as handled by the `NSURLSessionDelegate` method `URLSession:didReceiveChallenge:completionHandler:`

```
- (void)setSessionDidReceiveAuthenticationChallengeBlock:(nullable NSURLSessionAuthChallengeDisposition (^)(NSURLSession *session, NSURLAuthenticationChallenge *challenge, NSURLCredential * _Nullable __autoreleasing * _Nullable credential))block;

```
**- 1.设置收到认证挑战时的方法的方法**<br>
**- 2.在这个方法中我们可以收到响应的`NSURLCredential *credential`，`NSURLAuthenticationChallenge *challenge`**<br>
**- 3.我们可以在2中的两个参数中获取到响应者的host port credential等信息，这样我们就可以判别响应者是否是我们的真实的服务器，或者是否满足我们特定的需求，进而做安全策略，防止postman/charles等抓包工具抓包，中间人劫持等都有效果**<br>
**- 4.`NSURLSessionDelegate`中的`URLSession:didReceiveChallenge:completionHandler:`回调时会调用这里设置的block**
<br>
<br>
<br>

##Setting Task Delegate Callbacks
以下方法中设置的block都会在对应的NSURLSessionTaskDelegate方法中执行

>as handled by the `NSURLSessionTaskDelegate` method `URLSession:task:needNewBodyStream:`

```
- (void)setTaskNeedNewBodyStreamBlock:(nullable NSInputStream * (^)(NSURLSession *session, NSURLSessionTask *task))block;
```

**当对于一个基于数据流上传主体的任务认证失败的时候，任务不能安全的倒回并且重使用之前的数据流。`NSRLSession` 对象会调用代理的 `URLSession:task:needNewBodyStream:` 方法来获取一个能够为请求提供新的主体数据的新的 `NSInputStream` 对象**
<br>
<br>

>as handled by the `NSURLSessionTaskDelegate` method `URLSession:willPerformHTTPRedirection:newRequest:completionHandler:`

```
- (void)setTaskWillPerformHTTPRedirectionBlock:(nullable NSURLRequest * _Nullable (^)(NSURLSession *session, NSURLSessionTask *task, NSURLResponse *response, NSURLRequest *request))block;
```
**- 1.设置重定向request block**<br>
**- 2.比如我们请求百度的时候，将请求重定向到qq**
<br>
<br>

>as handled by the `NSURLSessionTaskDelegate` method `URLSession:task:didReceiveChallenge:completionHandler:`

```
- (void)setTaskDidReceiveAuthenticationChallengeBlock:(nullable NSURLSessionAuthChallengeDisposition (^)(NSURLSession *session, NSURLSessionTask *task, NSURLAuthenticationChallenge *challenge, NSURLCredential * _Nullable __autoreleasing * _Nullable credential))block;
```
**- 1.设置身份验证，自定义TLS，链验证等处理方法的block**<br>
**- 2.它和*Setting Session Delegate Callbacks*章节中的第二个方法是一样的作用**
####两个方法的区别：[引用自这里](http://ihomway.cc/2015/04/03/NSURLSession/)

> 对于会话层级别的授权请求 – NSURLAuthenticationMethodNTLM ，NSURLAuthenticationMethodNegotiate ，NSURLAuthenticationMethodClientCertificate 或者 NSURLAuthenticationMethodServerTrust – NSULRSession 对象会调用会话代理的 URLSession:didReceiveChallenge:completionHandler: 方法。如果 app 没有实现该会话代理方法，那么会代用任务代理的 URLSession:task:didReceiveChallenge:completionHandler: 方法来处理授权。

#

>对于非会话层级别的请求 NSURLSession 对象调用会话代理的URLSession:task:didReceiveChallenge:completionHandler: 方法来处理，如果提供了会话的代理同时也需要去处理授权认证，那么必须在任务级别也处理授权或者在任务中明确的调用每个会话来处理。会话代理方法 URLSession:didReceiveChallenge:completionHandler: 不会在非会话层级别的授权请求中被调用。

<br>
<br>


>`NSURLSessionTaskDelegate` method `URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:`

```
- (void)setTaskDidSendBodyDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionTask *task, int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend))block;

```
**- 1.定期获取当前task中的upload任务已经发送和总共需要发送的数据量**<br>
**- 2.获取当前上传进度**
<br>
<br>

>`NSURLSessionTaskDelegate` method `URLSession:task:didCompleteWithError:`

```
- (void)setTaskDidCompleteBlock:(nullable void (^)(NSURLSession *session, NSURLSessionTask *task, NSError * _Nullable error))block;
```
**task任务执行完毕的回调**
<br>
<br>
<br>

##Setting Data Task Delegate Callbacks
以下方法中设置的block都会在对应的NSURLSessionDataDelegate方法中执行

>as handled by the `NSURLSessionDataDelegate` method `URLSession:dataTask:didReceiveResponse:completionHandler:`

```
- (void)setDataTaskDidReceiveResponseBlock:(nullable NSURLSessionResponseDisposition (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLResponse *response))block;
```
**设定一个block，根据session，dataTask，response来决定task继续下载、取消，转成download task，转成Stream task（NSURLSessionResponseDisposition）**
<br>
<br>

>as handled by the `NSURLSessionDataDelegate` method `URLSession:dataTask:didBecomeDownloadTask:`

```
- (void)setDataTaskDidBecomeDownloadTaskBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLSessionDownloadTask *downloadTask))block;
```

**如果上边的方法中，block return NSURLSessionResponseBecomeDownload，则data task 会转变成一个download task， 进而会调用该方法**
**如果该方法执行了，说明task已经成为了download task，以后的代理方法也不会再调用data task 的代理方法，而是走download task的代理方法**
<br>
<br>


>as handled by the `NSURLSessionDataDelegate` method `URLSession:dataTask:didReceiveData:`

```
- (void)setDataTaskDidReceiveDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSData *data))block;
```
**在数据下载期间不定时的调用，返回每次下载回来的data，我们需要将这些data拼接，到最后获取到完整的数据**
<br>
<br>

>as handled by the `NSURLSessionDataDelegate` method `URLSession:dataTask:willCacheResponse:completionHandler:`

```
- (void)setDataTaskWillCacheResponseBlock:(nullable NSCachedURLResponse * (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSCachedURLResponse *proposedResponse))block;
```
**根据返回的proposedResponse，dataTask，session来判定当前task下的response缓存策略（TODO：NSURLCache NSCachedURLResponse）**
<br>
<br>


>as handled by the `NSURLSessionDataDelegate` method `URLSessionDidFinishEventsForBackgroundURLSession:`

```
- (void)setDidFinishEventsForBackgroundURLSessionBlock:(nullable void (^)(NSURLSession *session))block AF_API_UNAVAILABLE(macos);
```

>引用自 [nuclear](https://www.jianshu.com/p/0c10b922a691)
>
>note：iOS8之前，data task不支持后台session
>
当app不再运行而且后台传输结束或者请求证书的时候，iOS会调用`application:handleEventsForBackgroundURLSession:completionHandler:`方法在后台自动重启应用。这个方法提供了session标识来重启app，app应该存储的`comoletionHandler`，用这个标识来创建一个后台`configuration`对象，然后用这个`configuration`对象创建一个session.这个新的session会自动与后台的活动关联起来。之后当session晚餐最后一个后台任务的适合，会调用代理的`URLSessionDidFinishEventsForBackgroundURLSession:`方法，在这个代理方法中回到主线程调用之前存储的completionHandler，以便于让系统知道你的app又可以再次安全的处于挂起状态了。

## Setting Download Task Delegate Callbacks

>as handled by the `NSURLSessionDownloadDelegate` method `URLSession:downloadTask:didFinishDownloadingToURL:`

```
- (void)setDownloadTaskDidFinishDownloadingBlock:(nullable NSURL * _Nullable  (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location))block;
```
**- 1.下载完成的回调**<br>
**- 2.需要return 存储被下载文件的url**<br>
**- 3.当file manager在将临时文件移动到目的url的时候发生的错误，会触发`AFURLSessionDownloadTaskDidFailToMoveFileNotification`通知的发送**
<br>
<br>

>as handled by the `NSURLSessionDownloadDelegate` method `URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesWritten:totalBytesExpectedToWrite:`

```
- (void)setDownloadTaskDidWriteDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t bytesWritten, int64_t totalBytesWritten, int64_t totalBytesExpectedToWrite))block;
```
**在download session的整个生命周期中不定时的调用，反馈下载进度**<br>
<br>

>as handled by the `NSURLSessionDownloadDelegate` method `URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes:`

```
- (void)setDownloadTaskDidResumeBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t fileOffset, int64_t expectedTotalBytes))block;
```
**在重新启动下载的时候被调用**
**可以获取到文件的偏移（已经下载的数量），预期的总量**
<br>
<br>
<br>

##Notifications


- AFNetworkingTaskDidResumeNotification 重启
- AFNetworkingTaskDidCompleteNotification 完成
- AFNetworkingTaskDidSuspendNotification 挂起
- AFURLSessionDidInvalidateNotification 失效/被取消
- AFURLSessionDownloadTaskDidFailToMoveFileNotification download task将临时下载文件移动到目的url时出错

## UserInfo Keys
- AFNetworkingTaskDidCompleteResponseDataKey 非下载数据Key
- AFNetworkingTaskDidCompleteSerializedResponseKey 序列化后的非下载数据Key
- AFNetworkingTaskDidCompleteResponseSerializerKey 存储序列化器的Key
- AFNetworkingTaskDidCompleteAssetPathKey 下载完成后的文件路径Key
- AFNetworkingTaskDidCompleteErrorKey 请求出错信息Key