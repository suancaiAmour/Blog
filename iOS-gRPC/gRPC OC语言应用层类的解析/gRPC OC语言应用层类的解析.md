title: gRPC OC语言应用层类的解析  
description: gRPC OC语言应用层中各个类的简单解析  
date: 2017/2/4  15:50  
comments: true  
toc: true  
category: iOS-gRPC  

---
# 简述  
本文主要讲述 gRPC Objective-C 语言中，常见应用层类的解析，了解其中的类，方便我们进一步解析 gRPC 更底层的信息。  


# GRPCHost  
## GRPCHost 公开成员变量  

```
@property(nonatomic, readonly) NSString *address;
```
服务器地址。  

```
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

```
@property(nonatomic, nullable) struct grpc_channel_credentials *channelCreds;
```
SSL 证书。  

```
@property(nonatomic, getter=isSecure) BOOL secure;
```
GRPCHost 对象是否安全连接。  

```
@property(nonatomic, copy, nullable) NSString *hostNameOverride;
```
覆盖服务器名字，在 SSL 连接中有用。在设置 channel 参数的时候，GRPC_ARG_MAX_MESSAGE_LENGTH 会被赋值成这个。  

```
@property(nonatomic, strong, nullable) NSNumber *responseSizeLimitOverride;  
```
HTTP/2 中 信息的最大数据量，默认是 4MB。  

## GRPCHost 私有成员变量  

```
GRPCChannel *_channel;
```
连接通道。  

## GRPCHost 公开类方法  

```
+ (void)flushChannelCache;  
```
调用 kHostCache 里面存储的 GRPCHost 对象的`disconnect`方法。有一个全局的 kHostCache，里面存储了 GRPCHost 对象。  

```
+ (void)resetAllHostSettings;  
```
重新创建 kHostCache，这样也会清空 kHostCache。  

```
+ (nullable instancetype)hostWithAddress:(NSString *)address;
```
创建 GRPCHost 对象或者从 kHostCache 获取一个 GRPCHost 对象。  

## GRPCHost 公开对象方法  

```
- (nullable instancetype)initWithAddress:(NSString *)address NS_DESIGNATED_INITIALIZER;
```
创建 GRPCHost 对象或者从 kHostCache 获取一个 GRPCHost 对象。  

```
- (BOOL)setTLSPEMRootCerts:(nullable NSString *)pemRootCerts
            withPrivateKey:(nullable NSString *)pemPrivateKey
             withCertChain:(nullable NSString *)pemCertChain
                     error:(NSError **)errorPtr;
```
创建 SSL 证书，赋值到公开成员变量`channelCreds`。  

```
- (nullable struct grpc_call *)unmanagedCallWithPath:(NSString *)path
                                     completionQueue:(GRPCCompletionQueue *)queue;
```
创建一个 grpc_call 对象去连接这个服务器。  

```
- (void)disconnect;  
```
将调用者的私有 _channel 置为 nil。 


**设置 GRPCHost 的成员变量都应该在创建 grpc_call 方法前，否则设置的成员变量无用。** 


# GRPCChannel  

## GRPCChannel 公开成员变量  

```
@property(nonatomic, readonly, nonnull) struct grpc_channel *unmanagedChannel;
```
grpc 通道层的 channel。  

## GRPCChannel 私有成员变量  

```
NSString *_host;
```
服务器地址。  

```
grpc_channel_args *_channelArgs;
```
gRPC 通道层 channel 参数。

##  GRPCChannel 公开类方法  

```
+ (nullable GRPCChannel *)secureChannelWithHost:(nonnull NSString *)host;
```
使用默认的证书创建一个 GRPCChannel 对象。  

```
+ (nonnull GRPCChannel *)secureChannelWithHost:(nonnull NSString *)host
    credentials:(nonnull struct grpc_channel_credentials *)credentials
    channelArgs:(nullable NSDictionary *)channelArgs;
```
创建一个安全连接的 GRPCChannel 对象。  

```
+ (nonnull GRPCChannel *)insecureChannelWithHost:(nonnull NSString *)host
                                     channelArgs:(nullable NSDictionary *)channelArgs;
```
创建一个不安全连接的 GRPCChannel 对象。  


## GRPCChannel 公开对象方法  

```
- (nullable grpc_call *)unmanagedCallWithPath:(nonnull NSString *)path
                              completionQueue:(nonnull GRPCCompletionQueue *)queue;
```
返回一个 grpc_call 对象去发起连接。  


# GRPCCompletionQueue  
## GRPCCompletionQueue 公开成员变量  

```
@property(nonatomic, readonly) grpc_completion_queue *unmanagedQueue;
```
gRPC 通道层对象。  

## GRPCCompletionQueue 公开类方法

```
+ (instancetype)completionQueue;
```
创建一个 GRPCCompletionQueue 对象。调用这个方法会使用 GCD 创建一个新线程，一直监听  
`unmanagedQueue`的数据，而新创建的线程就是 gRPC 网络数据接收到后，我们回调的 block 的调用线程，具体实现在 gRPC 的通道层中。   


# ProtoService
## ProtoService 私有成员变量  

```
NSString *_host;
```
服务器地址。  

```
NSString *_packageName;
```
调用方法的包名（具体可看 proto3）。

```
NSString *_serviceName;
```
调用方法的服务名（具体可看 proto3）。

**由上面三个私有成员变量，可精确在服务器中找到我们所要调用的方法，注意，三者缺一不可。**  

## ProtoService 公开对象方法  

```
- (instancetype)initWithHost:(NSString *)host
                 packageName:(NSString *)packageName
                 serviceName:(NSString *)serviceName NS_DESIGNATED_INITIALIZER;
```
创建一个 ProtoService 对象。  

```
- (GRPCProtoCall *)RPCToMethod:(NSString *)method
           requestsWriter:(GRXWriter *)requestsWriter
  	        responseClass:(Class)responseClass
  	   responsesWriteable:(id<GRXWriteable>)responsesWriteable;
```
创建一个 GRPCProtoCall 对象，用于发起网络请求。  

