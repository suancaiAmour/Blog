title: gRPC OC 语言应用层网络请求步骤解析  
description: 讲述 gRPC 在应用层是怎么运作的  
date: 2017/2/5 18:04  
category: iOS-gRPC  
comments: true  
toc: true  
  
---
# 简述  
本文主要讲解 gRPC 在 OC 语言应用层是怎么运作，怎么发起网络请求的，哪些是关键点。  

# gRPC 网络发起关键类 - GRPCCall
GRPCCall 是 gRPC 应用层发起网络请求，接收网络数据的关键类，是整个应用层的核心类。  
在我的另一篇博客[gRPC OC语言应用层类的解析](http://chenmaomao.com/2017/02/04/%E6%8A%80%E6%9C%AF/iOS-gRPC/gRPC%20OC%E8%AF%AD%E8%A8%80%E5%BA%94%E7%94%A8%E5%B1%82%E7%B1%BB%E7%9A%84%E8%A7%A3%E6%9E%90/gRPC%20OC%E8%AF%AD%E8%A8%80%E5%BA%94%E7%94%A8%E5%B1%82%E7%B1%BB%E7%9A%84%E8%A7%A3%E6%9E%90/)曾经简单讲述这个类，这里我将重新详细讲述它。  

## GRPCCall 私有成员变量  
GRPCCall 中私有成员变量是实现它功能的重要工具，这些工具分别是：  

1. 网络连接，发送接收网络数据的类:  

	```Objectivec
	GRPCWrappedCall *_wrappedCall;
	```
	这个类可以发起网络连接，发送网络数据和接收网络数据。  
2. 回调的 block 的处理类：

	```Objectivec
	GRXConcurrentWriteable *_responseWriteable;
	```
	这个类主要用来实现保证我们所写的 block 是线程安全的，且可以在接收到相应网络数据的时候回调相应的 block。  
3. 客户端所发送信息的存储的类：

	```Objectivec
	GRXWriter *_requestWriter;
	```
	这个类里面存储了我们所要发送的信息，其实也就是`proto 3`所转换成 OC 类的进一步封装。  

## 如何利用 _wrappedCall 完成网络事件注册  
在 gRPC GRPCCall 中，网络连接开始第一个 API 是：

```Objectivec
 (void)startWithWriteable:(id<GRXWriteable>)writeable {
  @synchronized(self) {
    _state = GRXWriterStateStarted;
  }
  _retainSelf = self;
  _responseWriteable = [[GRXConcurrentWriteable alloc] initWithWriteable:writeable];
  _wrappedCall = [[GRPCWrappedCall alloc] initWithHost:_host path:_path];

  [self sendHeaders:_requestHeaders];
  [self invokeCall];
  
  NSString *host = [NSURL URLWithString:[@"https://" stringByAppendingString:_host]].host;
  if (!host) {
    [NSException raise:NSInvalidArgumentException format:@"host of %@ is nil", _host];
  }
  __weak typeof(self) weakSelf = self;
  _connectivityMonitor = [GRPCConnectivityMonitor monitorWithHost:host];
  [_connectivityMonitor handleLossWithHandler:^{
    typeof(self) strongSelf = weakSelf;
    if (strongSelf) {
      [strongSelf finishWithError:[NSError errorWithDomain:kGRPCErrorDomain
                                                      code:GRPCErrorCodeUnavailable
                                                  userInfo:@{NSLocalizedDescriptionKey: @"Connectivity lost."}]];
      [[GRPCHost hostWithAddress:strongSelf->_host] disconnect];
    }
  }];
}
```
这个 API 主要是创建`_responseWriteable`和`_wrappedCall`，关键点在于接下来的两个方法：  

```Objectivec
[self sendHeaders:_requestHeaders];      
[self invokeCall];
```
1. `[self sendHeaders:_requestHeaders]`是发送请求头：  

```Objectivec
- (void)sendHeaders:(NSDictionary *)headers {
  [_wrappedCall startBatchWithOperations:@[[[GRPCOpSendMetadata alloc] initWithMetadata:headers
                                                                                handler:nil]]];
}
```
这其中就调用了`_wrappedCall`发起网络请求，这个传入的回调 handler 为空，所以发送请求头成功后不会有回调 block。  
2. `[self invokeCall]`是发送数据和读取数据最上层 API ：

```Objectivec
- (void)invokeCallWithHeadersHandler:(void(^)(NSDictionary *))headersHandler
                    completionHandler:(void(^)(NSError *, NSDictionary *))completionHandler {
  [_wrappedCall startBatchWithOperations:@[[[GRPCOpRecvMetadata alloc]
                                            initWithHandler:headersHandler]]];
  [_wrappedCall startBatchWithOperations:@[[[GRPCOpRecvStatus alloc]
                                            initWithHandler:completionHandler]]];
}

- (void)invokeCall {
  [self invokeCallWithHeadersHandler:^(NSDictionary *headers) {
    self.responseHeaders = headers;
    [self startNextRead];
  } completionHandler:^(NSError *error, NSDictionary *trailers) {
    self.responseTrailers = trailers;

    if (error) {
      NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
      if (error.userInfo) {
        [userInfo addEntriesFromDictionary:error.userInfo];
      }
      userInfo[kGRPCTrailersKey] = self.responseTrailers;
      if (self.responseHeaders) {
        userInfo[kGRPCHeadersKey] = self.responseHeaders;
      }
      error = [NSError errorWithDomain:error.domain code:error.code userInfo:userInfo];
    }
    [self finishWithError:error];
  }];

  @synchronized(_requestWriter) {
    [_requestWriter startWithWriteable:self];
  }
}
```
从上面可以看出`- (void)invokeCall`会调用  

```Objectivec
- (void)invokeCallWithHeadersHandler:(void(^)(NSDictionary *))headersHandler
                    completionHandler:(void(^)(NSError *, NSDictionary *))completionHandler
```
`invokeCallWithHeadersHandler`是使用`_wrappedCall`接收回应头和服务器状态。  

`- (void)invokeCall`还会调用`[_requestWriter startWithWriteable:self]`，这个方法主要获取`_requestWriter`里面我们所要发送的数据，最后会触发：

```Objectivec
- (void)writeMessage:(NSData *)message withErrorHandler:(void (^)())errorHandler {

  __weak GRPCCall *weakSelf = self;
  void(^resumingHandler)(void) = ^{
    // Resume the request writer.
    GRPCCall *strongSelf = weakSelf;
    if (strongSelf) {
      @synchronized(strongSelf->_requestWriter) {
        strongSelf->_requestWriter.state = GRXWriterStateStarted;
      }
    }
  };
  [_wrappedCall startBatchWithOperations:@[[[GRPCOpSendMessage alloc] initWithMessage:message
                                                                              handler:resumingHandler]]
                            errorHandler:errorHandler];
}
```
从中我们可以看到，它使用了`_wrappedCall`发送消息。  

已然有发送消息，但接收消息在哪呢？  
可以翻阅到`- (void)invokeCall`这个 API。当接收请求头后，会触发`headersHandler`回调 block。  
在`headersHandler`中有一个`[self startNextRead]`方法：  

```Objectivec
- (void)startNextRead {
  if (self.state == GRXWriterStatePaused) {
    return;
  }
  __weak GRPCCall *weakSelf = self;
  __weak GRXConcurrentWriteable *weakWriteable = _responseWriteable;

  dispatch_async(_callQueue, ^{
    [weakSelf startReadWithHandler:^(grpc_byte_buffer *message) {
      if (message == NULL) {
        return;
      }
      NSData *data = [NSData grpc_dataWithByteBuffer:message];
      grpc_byte_buffer_destroy(message);
      if (!data) {
        [weakSelf finishWithError:[NSError errorWithDomain:kGRPCErrorDomain
                                                      code:GRPCErrorCodeInternal
                                                  userInfo:nil]];
        [weakSelf cancelCall];
        return;
      }
      [weakWriteable enqueueValue:data completionHandler:^{
        [weakSelf startNextRead];
      }];
    }];
  });
}
```
`- (void)startNextRead`主要触发  
`- (void)startReadWithHandler:(void(^)(grpc_byte_buffer *))handler`  
这个 API，而这个 API 就是利用`_wrappedCall`去接收数据。注意，`handler`会回调`startNextRead`方法，这样便实现了不断从服务器读数据流，直到接收到错误或网络完成状态。  

**上面`_wrappedCall`调用的几个方法是往通道层注册这几个网络事件，这样当发生这几个网络事件的时候，就会触发相应的回调 block。**  

>上面过程还有一个客户端发送关闭自己端口的网络事件(仍然可以接收服务器数据，但不能发送数据)，这个事件注册在客户端发送完数据后，这时候客户端和服务器的连接处于半关闭状态。  


## 如何监听注册的网络事件  
上面中已经使用`_wrappedCall`在应用层注册网络事件在通道层。既然已经注册网络事件，就必须监听这些网络事件，监听网络事件使用的类是`_wrappedCall`的私有成员变量`_queue`。  
`_queue`是 GRPCCompletionQueue 类，当创建它的时候，会发生以下过程：  

```Objectivec
- (instancetype)init {
  if ((self = [super init])) {
    _unmanagedQueue = grpc_completion_queue_create(NULL);

    grpc_completion_queue *unmanagedQueue = _unmanagedQueue;
    
    static dispatch_once_t initialization;
    static dispatch_queue_t gDefaultConcurrentQueue;
    dispatch_once(&initialization, ^{
      gDefaultConcurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    });
    dispatch_async(gDefaultConcurrentQueue, ^{
      while (YES) {
        grpc_event event = grpc_completion_queue_next(unmanagedQueue,
                                               gpr_inf_future(GPR_CLOCK_REALTIME),
                                                      NULL);
        GRPCQueueCompletionHandler handler;
        switch (event.type) {
          case GRPC_OP_COMPLETE:
            handler = (__bridge_transfer GRPCQueueCompletionHandler)event.tag;
            handler(event.success);
            break;
          case GRPC_QUEUE_SHUTDOWN:
            grpc_completion_queue_destroy(unmanagedQueue);
            return;
          default:
            [NSException raise:@"Unrecognized completion type" format:@""];
        }
      };
    });
  }
  return self;
}
```
从上面可以看出，当创建`_queue`的时候，会创建一个新线程，并陷入死循环中，一直监听网络事件的发生，其中事件的`tag`就是我们上面注册网络事件时传进去的回调 block。这里封装的事件有三种类型，没有和上面的6种网络事件对应，要想知道此时发生的是哪个网络事件，可以从`tag`和回调的 block 相对应，这样就能清楚此时发生的是什么网络事件。  

>PS: 使用死循环，而不是 Runloop 之类的，不会造成性能损失吗？在我的测试当中，这个死循环会停下来，具体原因未明。  


# 通道层通道参数解析  
在通道层的核心是 channel，可以设置它的参数，来实现更高效的连接。本这是通道层的对象，但设置却可以在应用层设置，这里就说一下这些参数的意义。  
通道参数定义在`grpc_types.h`中：

```Objectivec
#define GRPC_ARG_MAX_RECONNECT_BACKOFF_MS "grpc.max_reconnect_backoff_ms"
#define GRPC_ARG_INITIAL_RECONNECT_BACKOFF_MS  "grpc.initial_reconnect_backoff_ms"
```
先说明两个参数的意义，这两个参数是一起使用的，所以这里统一说明。  
`""grpc.max_reconnect_backoff_ms""`是网络请求重连的最大间隔时间，单位 ms。  
`"grpc.initial_reconnect_backoff_ms"`是网络请求第一次重连的间隔时间，单位 ms。  
这两个有什么关系呢？当网络请求没有成功时，gRPC 底层会自动进行网络重连(当然是没有取消网络连接的情况下重连)。当我们设置网络请求第一次重连的间隔时间后，当发起网络请求，如果没有服务器没有回应，等待第一次重连的间隔时间，如果还没有回应，就发起重连，第一次发起重连后，重连的时间间隔会随着重连次数的增多不断增加，最后增加到网络请求重连的最大间隔时间，然后就以这个最大重连间隔时间重连，不再增加重连的间隔时间。  
这里有个问题，如果网络请求第一次重连的间隔时间大于网络请求重连的最大间隔时间怎么办？经我个人的测试结果是，此时的重连会以两个间隔时间轮番重连，一开始是网络请求第一次重连的间隔时间重连，然后是网络请求重连的最大间隔时间，而后又网络请求第一次重连的间隔时间重连，如此轮番，直到连接上去。  
如果没有设置网络请求重连的最大间隔时间，则默认是120秒。网络请求第一次重连的间隔时间如果我们没修改代码则设置为10秒。  
这个重连时间的设置也很讲究，如果设置短了，可能服务器回应还没到就重连了，如果设置太长，则太浪费时间。使用者在重连不上的时候，可以设置取消网络请求，而发起网络请求到取消网络请求的时间间隔要综合这两个重连时间间隔一起综合设置，总不能一次重连都没有就取消连接了吧。  

```Objectivec
#define GRPC_ARG_MAX_METADATA_SIZE "grpc.max_metadata_size"
```
请求头的最大数据量，单位 bytes。  

```Objectivec
#define GRPC_ARG_HTTP2_HPACK_TABLE_SIZE_ENCODER  "grpc.http2.hpack_table_size.encoder"
```
HTTP/2 头部压缩编码的空间大小，单位 bytes。  

```Objectivec
#define GRPC_ARG_HTTP2_HPACK_TABLE_SIZE_DECODER  "grpc.http2.hpack_table_size.decoder"
```
HTTP/2 头部压缩解码的空间大小，单位 bytes。  

```Objectivec
#define GRPC_ARG_MAX_MESSAGE_LENGTH "grpc.max_message_length"
```
接收信息的最大数据量，单位 bytes，默认是 4MB。  

```Objectivec
#define GRPC_ARG_MAX_CONCURRENT_STREAMS "grpc.max_concurrent_streams"
```
一个 HTTP/2 连接所拥有的最大并发的数据流的数量。  

```Objectivec
#define GRPC_ARG_HTTP2_STREAM_LOOKAHEAD_BYTES "grpc.http2.lookahead_bytes"
```
默认是 64kb，干嘛用的暂时还不清楚，但从注释可以看出，这个参数可以优化性能，所以在这里标出一下，希望有人能指点一下。增大这个参数可以帮助在高延迟的连接中增加吞吐量，在某些情况，我们应该让这个参数`auto-tune`，让它变成`no-op`。  