title: gRPC OC 语言应用层类的解析  
description: gRPC OC 语言应用层中各个类的简单解析  
date: 2017/2/4  15:50  
comments: true  
toc: true  
category: iOS-gRPC  

---
# 简述  
本文主要讲述 gRPC Objective-C 语言中，常见应用层类的解析，了解其中的类，方便我们进一步解析 gRPC 更底层的信息。  


# GRPCHost  
## GRPCHost 公开成员变量  

```Objectivec
@property(nonatomic, readonly) NSString *address;
```
服务器地址。  

```Objectivec
@property(nonatomic, copy, nullable) NSString *userAgentPrefix;
```
用户名前缀，虽然协议不要求提供用户名功能，但建议提供用户名（比如说，调用库，版本和平台）更方便去调试问题。HTTP/2 用户名结构：  

```
User-Agent → "grpc-" Language ?("-" Variant) "/" Version ?( " ("  *(AdditionalProperty ";") ")" )
```
在 gRPC 中的例子：

```
grpc-java/1.2.3
grpc-ruby/1.2.3
grpc-ruby-jruby/1.3.4
grpc-java-android/0.9.1 (gingerbread/1.2.4; nexus5; tmobile)
```
在 OC 语言中，如果设置了这个用户名前缀，会添加到上面的用户名前面并发送给服务器。  

```Objectivec
@property(nonatomic, nullable) struct grpc_channel_credentials *channelCreds;
```
SSL 证书。  

```Objectivec
@property(nonatomic, getter=isSecure) BOOL secure;
```
GRPCHost 对象是否安全连接。  

```Objectivec
@property(nonatomic, copy, nullable) NSString *hostNameOverride;
```
覆盖服务器名字，在 SSL 连接中有用。  

```Objectivec
@property(nonatomic, strong, nullable) NSNumber *responseSizeLimitOverride;  
```
HTTP/2 中 信息的最大数据量，默认是 4MB。在设置 channel 参数的时候，GRPC_ARG_MAX_MESSAGE_LENGTH 会被赋值成这个。  

## GRPCHost 私有成员变量  

```Objectivec
GRPCChannel *_channel;
```
连接通道。  

## GRPCHost 公开类方法  

```Objectivec
+ (void)flushChannelCache;  
```
调用 kHostCache 里面存储的 GRPCHost 对象的`disconnect`方法。有一个全局的 kHostCache，里面存储了 GRPCHost 对象。  

```Objectivec
+ (void)resetAllHostSettings;  
```
重新创建 kHostCache，这样也会清空 kHostCache。  

```Objectivec
+ (nullable instancetype)hostWithAddress:(NSString *)address;
```
创建 GRPCHost 对象或者从 kHostCache 获取一个 GRPCHost 对象。  

## GRPCHost 公开对象方法  

```Objectivec
- (nullable instancetype)initWithAddress:(NSString *)address NS_DESIGNATED_INITIALIZER;
```
创建 GRPCHost 对象或者从 kHostCache 获取一个 GRPCHost 对象。  

```Objectivec
- (BOOL)setTLSPEMRootCerts:(nullable NSString *)pemRootCerts
            withPrivateKey:(nullable NSString *)pemPrivateKey
             withCertChain:(nullable NSString *)pemCertChain
                     error:(NSError **)errorPtr;
```
创建 SSL 证书，赋值到公开成员变量`channelCreds`。  

```Objectivec
- (nullable struct grpc_call *)unmanagedCallWithPath:(NSString *)path
                                     completionQueue:(GRPCCompletionQueue *)queue;
```
创建一个 grpc_call 对象去连接这个服务器。  

```Objectivec
- (void)disconnect;  
```
将调用者的私有 _channel 置为 nil。 


**设置 GRPCHost 的成员变量都应该在创建 grpc_call 方法前，否则设置的成员变量无用。** 


# GRPCChannel  

## GRPCChannel 公开成员变量  

```Objectivec
@property(nonatomic, readonly, nonnull) struct grpc_channel *unmanagedChannel;
```
grpc 通道层的 channel。  

## GRPCChannel 私有成员变量  

```Objectivec
NSString *_host;
```
服务器地址。  

```Objectivec
grpc_channel_args *_channelArgs;
```
gRPC 通道层 channel 参数。

##  GRPCChannel 公开类方法  

```Objectivec
+ (nullable GRPCChannel *)secureChannelWithHost:(nonnull NSString *)host;
```
使用默认的证书创建一个 GRPCChannel 对象。  

```Objectivec
+ (nonnull GRPCChannel *)secureChannelWithHost:(nonnull NSString *)host
    credentials:(nonnull struct grpc_channel_credentials *)credentials
    channelArgs:(nullable NSDictionary *)channelArgs;
```
创建一个安全连接的 GRPCChannel 对象。  

```Objectivec
+ (nonnull GRPCChannel *)insecureChannelWithHost:(nonnull NSString *)host
                                     channelArgs:(nullable NSDictionary *)channelArgs;
```
创建一个不安全连接的 GRPCChannel 对象。  


## GRPCChannel 公开对象方法  

```Objectivec
- (nullable grpc_call *)unmanagedCallWithPath:(nonnull NSString *)path
                              completionQueue:(nonnull GRPCCompletionQueue *)queue;
```
返回一个 grpc_call 对象去发起连接，在 GRPCHost 中，有一个类似的方法，其实是调用这个方法。   


# GRPCCompletionQueue  
## GRPCCompletionQueue 公开成员变量  

```Objectivec
@property(nonatomic, readonly) grpc_completion_queue *unmanagedQueue;
```
gRPC 通道层对象。  

## GRPCCompletionQueue 公开类方法

```Objectivec
+ (instancetype)completionQueue;
```
创建一个 GRPCCompletionQueue 对象。调用这个方法会使用 GCD 创建一个新线程，一直监听进入  
`unmanagedQueue`的事件。监听到某些网络事件发生的时候，就可以回调相应的 block，因此我们可以在这里可以监听到网络数据。  


# ProtoService
## ProtoService 私有成员变量  

```Objectivec
NSString *_host;
```
服务器地址。  

```Objectivec
NSString *_packageName;
```
调用服务器方法的包名（具体可看 proto3）。

```Objectivec
NSString *_serviceName;
```
调用服务器方法的服务名（具体可看 proto3）。

**由上面三个私有成员变量，可精确在服务器中找到我们所要调用的方法，注意，三者缺一不可。**  

## ProtoService 公开对象方法  

```Objectivec
- (instancetype)initWithHost:(NSString *)host
                 packageName:(NSString *)packageName
                 serviceName:(NSString *)serviceName NS_DESIGNATED_INITIALIZER;
```
创建一个 ProtoService 对象。  

```Objectivec
- (GRPCProtoCall *)RPCToMethod:(NSString *)method
           requestsWriter:(GRXWriter *)requestsWriter
  	        responseClass:(Class)responseClass
  	   responsesWriteable:(id<GRXWriteable>)responsesWriteable;
```
创建一个 GRPCProtoCall 对象，用于发起网络请求。  

# GRPCProtoService
继承于 ProtoService 且并无任何增加。  


# GRXWriter  
GRXWriter 类似于一个 NSEnumerator，区别在于 NSEnumerator 是从 writeable 拿值，而 GRXWriter 压值给 writeable。  
## GRXWriter 公开成员变量  

```Objectivec
@property(nonatomic) GRXWriterState state;
```
GRXWriter 的状态，由这个状态选择当前使用的*writeable*。一些状态的改变可以触发相应的动作。   
GRXWriter 的状态枚举有：  

```Objectivec
typedef NS_ENUM(NSInteger, GRXWriterState) {

  /**
   * 这个 writer  还未拥有一个 writeable。可以发送 startWithWriteable: 信息改变到开始状态。
   * 一个 writer 的状态不能手动的设置为这个状态。
   */
  GRXWriterStateNotStarted,

  /** writer 可以压值到 writeable 任何时候。*/
  GRXWriterStateStarted,

  /**
   * writer 被临时挂起，除非切换到开始状态，否则不可以发送值到 writeable， 这个 writer 可以在任何
   * 时候切换到完成状态，而且允许 writeable 发送 writesFinishedWithError: 。 
   */
  GRXWriterStatePaused,

  /**
   * writer 释放 writeable，而且不再影响。
   * 很少情况下会设置到这个状态，它的 writeable 也不会发送 writesFinishedWithError: 消息。
   * writer 发送 finishWithError: 消息， writer 会广播 writeable 并且切换到这个状态。
   */
  GRXWriterStateFinished
};
```

## GRXWriter 公开对象方法  

```Objectivec
- (void)startWithWriteable:(id<GRXWriteable>)writeable;
```
转变到开始状态，而且开始发送信息到 writeable。  
如果 writer 的值来源于其他(如文件系统或者一个服务器)，调用这个方法要触发副作用(像网络连接)。  
这个方法只有 writer 在 NotStarted 才可以调用。  

```Objectivec
- (void)finishWithError:(NSError *)errorOrNil;
```
writeable 发送 writesFinishedWithError:errorOrNil 消息，然后释放 writeable 且转换到完成状态。  
这个方法只有在 writer 在 Started 和 Paused 才可以调用。  


# GRXWriteable
## GRXWriteable 协议方法  

```Objectivec
- (void)writeValue:(id)value;
```
往接收消息队列中发送下一个值。  

```Objectivec
- (void)writesFinishedWithError:(NSError *)errorOrNil;
```
当消息对列完成或者出现错误调用。调用这个方法后，`writeValue:`不会再被调用。  

## GRXWriteable 私有成员变量  

```Objectivec
GRXValueHandler _valueHandler;
```
关联`writeValue:`方法。  

```Objectivec
GRXCompletionHandler _completionHandler;
```
关联`writesFinishedWithError:`方法。  

## GRXWriteable 公开类方法  

```Objectivec
+ (instancetype)writeableWithSingleHandler:(GRXSingleHandler)handler;
```
由 GRXSingleHandler 创建一个 GRXWriteable 对象。  

```Objectivec
+ (instancetype)writeableWithEventHandler:(GRXEventHandler)handler;
```
由 GRXEventHandler 创建一个 GRXWriteable 对象。  

## GRXWriteable 公开对象方法  

```Objectivec
- (instancetype)initWithValueHandler:(GRXValueHandler)valueHandler
                   completionHandler:(GRXCompletionHandler)completionHandler
```
创建一个 GRXWriteable 对象。  


# GRPCCall  
GRPCCall 继承于 GRXWriter。  
**每个 RPC 使用不同的 HTTP2 流但使用同一个 TCP 连接**

## GRPCCall 公开成员变量  

```Objectivec
@property(atomic, readonly) NSMutableDictionary *requestHeaders;
```
请求头，区分大小写。  
当请求已经开始后，修改这个成员变量会产生错误。  
这个成员变量初始化是一个空的`NSMutableDictionary`。  

```Objectivec
@property(atomic, readonly) NSDictionary *responseHeaders;
```
响应头。key 是响应头的名字，如果响应头名字以`-bin`结尾，则 value 是 NSDate 类型，否则 NSString 类型。  
这个成员变量是一个 nil 直到接收到所有响应头，在 writeable 调用`-writeValue:`  
或者`-writesFinishedWithError:`前会改变。  

```Objectivec
@property(atomic, readonly) NSDictionary *responseTrailers;
```
响应正文。  
这个成员变量是一个 nil 直到接收到所有响应正文，在 writeable 调用  
`-writesFinishedWithError:`前会改变。  

## GRPCCall 公开对象方法  

```Objectivec
- (instancetype)initWithHost:(NSString *)host
                        path:(NSString *)path
              requestsWriter:(GRXWriter *)requestsWriter NS_DESIGNATED_INITIALIZER;
```
创建一个 GRPCCall 对象。  
`requestsWriter`可以提供 NSData 对象给 Writeable。  

```Objectivec
- (void)cancel;
```
call 完成请求，通知服务器将完成 RPC，响应方会恢复一个 CANCELED 错误的响应给 call。  

## GRPCCall 私有成员变量  

```Objectivec
dispatch_queue_t _callQueue;
```
GCD 队列，发请网络请求的对列。  

```Objectivec
NSString *_host;
```
服务器地址。  

```Objectivec
NSString *_path;
```
服务器方法调用路径。  

```Objectivec
GRPCWrappedCall *_wrappedCall;
```
基于 grpc_call 封装起来的 OC 对象，是应用层调用通道层的接口类。  

```Objectivec
GRPCConnectivityMonitor *_connectivityMonitor;
```
网络监控类。  

```Objectivec
GRXConcurrentWriteable *_responseWriteable;
```
底层 C 语言库难以保证读取信息的数据和缺乏对错误的处理。这个封装类可以确保数据的读取顺序和线程安全。  

```Objectivec
GRXWriter *_requestWriter;
```
为了网络连接线程安全，具体应看相对应的类(这里应该是 GRXMappingWriter 类)，我们所要发送给服务器的信息存储在这个类里面(所要发送的信息包含在`proto 3`语言转变后的类)。   

```Objectivec
GRPCCall *_retainSelf;
```
创建一个循环引用，直到完成任务。  


# GRXConcurrentWriteable  
是对 GRXWriteable 的进一步封装(注意不是继承)，保证线程安全和信息顺序。  

## GRXConcurrentWriteable 私有成员变量  

```Objectivec
@property(atomic, strong) id<GRXWriteable> writeable;
```
GRXConcurrentWriteable 是对这个 writeable 的进一步封装。  

```Objectivec
dispatch_queue_t _writeableQueue;
```
GRXConcurrentWriteable 方法调用的线程。  

```Objectivec
dispatch_once_t _alreadyFinished;
```
保证封装的 writeable 的`writesFinishedWithError:`只会执行一次。  

## GRXConcurrentWriteable 公开对象方法  

```Objectivec
- (instancetype)initWithWriteable:(id<GRXWriteable>)writeable NS_DESIGNATED_INITIALIZER;
```
构造器。  

```Objectivec
- (void)enqueueValue:(id)value completionHandler:(void (^)())handler;
```
在`_writeableQueue`执行`writeable`的`writeValue:`和`handler`。  

```Objectivec
- (void)enqueueSuccessfulCompletion;
```
在`_writeableQueue`只执行一次`writeable`的  
`writesFinishedWithError:nil`且置空`writeable`。  

```Objectivec
- (void)cancelWithError:(NSError *)error;
```
error 不准为空，当发生错误时，调用的`writeable`的`writesFinishedWithError:`。  

```Objectivec
- (void)cancelSilently;
```
置空`writeable`。  


# GRPCWrappedCall
GRPCWrappedCall 是对 grpc_call 的封装。  

## GRPCWrappedCall 私有成员变量

```Objectivec
GRPCCompletionQueue *_queue;
```
监听网络事件，回调相应的 block。

```Objectivec
grpc_call *_call;
```
gRPC 通道层发起网络请求的对象。  


## GRPCWrappedCall 公开对象方法  

```Objectivec
- (instancetype)initWithHost:(NSString *)host
                        path:(NSString *)path NS_DESIGNATED_INITIALIZER;
```
构造器。  

```Objectivec
- (void)startBatchWithOperations:(NSArray *)ops errorHandler:(void(^)())errorHandler;
```
注册网络事件到通道层，包括接收数据和发送数据，决定于`ops`。  

```Objectivec
- (void)startBatchWithOperations:(NSArray *)ops;
```
同上。  

```Objectivec
- (void)cancel;
```
取消网络连接。  


# GRPCOperation 及其子类
GRPCOperation 是 GRPCWrappedCall 发起网络连接所必须传入的数组的成员，根据 GRPCOperation 类型可以知道 GRPCWrappedCall 所要传递和接收的数据类型。  
**GRPCOperation 一共有6种子类。**  

## GRPCOperation 公开成员变量  

```Objectivec
@property(nonatomic, readonly) grpc_op op;
```
gRPC 通道层对象。  

## GRPCOperation 私有成员变量  

```Objectivec
void(^_handler)();
```
回调 block。  

## GRPCOperation 公开对象方法  

```Objectivec
- (void)finish;
```
调用`_handler`。  

## GRPCOpSendMetadata 公开对象方法  

```Objectivec
- (instancetype)initWithMetadata:(NSDictionary *)metadata
                         handler:(void(^)())handler NS_DESIGNATED_INITIALIZER;
``` 
构造器。 GRPCOpSendMetadata 用于发送请求头。`handler`常为空。  

## GRPCOpSendMessage 公开对象方法  

```Objectivec
- (instancetype)initWithMessage:(NSData *)message
                        handler:(void(^)())handler NS_DESIGNATED_INITIALIZER;
```
构造器。 GRPCOpSendMessage 用于发送消息。  

## GRPCOpSendClose 公开对象方法  

```Objectivec
- (instancetype)initWithHandler:(void(^)())handler NS_DESIGNATED_INITIALIZER;
```
构造器。 GRPCOpSendClose 用于告诉服务器关闭网络连接。  

## GRPCOpRecvMetadata 公开对象方法  

```Objectivec
- (instancetype)initWithHandler:(void(^)(NSDictionary *))handler NS_DESIGNATED_INITIALIZER;
```
构造器。 GRPCOpRecvMetadata 用于接收响应头。`handler`通常不为空。  

## GRPCOpRecvMessage 公开对象方法

```Objective
- (instancetype)initWithHandler:(void(^)(grpc_byte_buffer *))handler NS_DESIGNATED_INITIALIZER;
```
构造器。 GRPCOpRecvMessage 用于接收信息。`handler`通常不为空。  

## GRPCOpRecvStatus 公开对象方法  

```Objectivec
- (instancetype)initWithHandler:(void(^)(NSError *, NSDictionary *))handler
    NS_DESIGNATED_INITIALIZER;
```
构造器。 GRPCOpRecvStatus 用于接收状态。`handler`通常不为空。  

