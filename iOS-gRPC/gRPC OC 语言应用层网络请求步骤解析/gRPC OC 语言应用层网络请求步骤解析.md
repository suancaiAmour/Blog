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

## 如何利用上面三个私有成员变量完成网络连接  
在 gRPC 中，网络连接开始第一个 API 是：

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
2. `[self invokeCall]`是发送数据和读取数据：

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
`invokeCallWithHeadersHandler`是使用`_wrappedCall`接收回应头和网络连接状态。  

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