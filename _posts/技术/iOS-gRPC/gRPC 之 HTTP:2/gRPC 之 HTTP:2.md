title: gRPC 之 HTTP/2  
description: 简述 HTTP/2 协议基本内容  
date: 2017/2/4 10:04  
comments: true  
toc: true  
category: iOS-gRPC  

---
# HTTP/2 的帧
HTTP/2 的数据格式有下：  
*  帧  
*  消息  
*  流  
HTTP/2 的基本数据是采用二进制编码的帧，由一个或多个帧组成消息，再由消息组成流。帧在数据流中乱序发送，再有帧首部的流标识符重新组装。  

## SETTINGS  
设置帧，客户端发送无 ACK 的 SETTINGS，服务器端接收到客户端 SETTINGS 后，处理 SETTINGS 的内容，然后发送有 ACK 的 SETTINGS。当客户端等待确认超时时，产生 SETTINGS_TIMEOUT 错误。设置帧表示当前连接，不针对某个流。  
## HEADER  
类似于 HTTP1.x 的请求头或者响应头，表明打开一个流。一般情况下，一个 HEADER 可以解决，但当 Cookie 超过16kib 的时候，需使用 CONTINUATION。  
## CONTINUATION  
当 HEADER 无法传送完整的报头的时候，使用 CONTINUATION 传送剩余的报头数据。  
## DATA  
报文数据，只有流处于开启或者半关闭状态才能发送。  
## PUSH_PROMISE  
服务器通知客户端将要推送消息给客户端。  
## PING  
优先级高于其他帧，发送者可测量最小往返时间，检测空闲连接是否有效。  
## PRIORITY  
发送方对某个流优先级的建议。  
## WINDOW_UPDATE  
流量控制帧。  
## RST_STREAWM  
发送方对某个流优先级的建议。  
## GOAWAY  
通知接收者停止创建流，并完成之前建立流的任务。  


**推荐文章阅读：**  
[HTTP/2笔记之帧](http://www.blogjava.net/yongboy/archive/2015/03/20/423655.html)


# HTTP/2 特性
## 多路复用  
HTTP1.x中，如果想并发多个请求，必须使用多个 TCP 链接，当超过一定数量的 TCP 链接后，浏览器会挂起后面的 TCP 链接，而 HTTP/2 中，是使用二进制帧的，实现多个请求，不再依赖 TCP 链接，而是在单个 TCP 链接中，使用多个流实现多路请求。  
## 流优先级  
每个请求都可以设置自己流的优先级。  
## 服务器推送
服务器可以向客户端推送资源。  
## 头部压缩  
HTTP/2 会压缩请求的头部，比如说，当第一个请求发送了头部字段，第二个请求可以只发送与第一个请求头部差异的部分，相同部分不用再次发送。  


**推荐文章阅读：**  
[HTTP/2协议–特性扫盲篇](http://www.cnblogs.com/yingsmirk/p/5248506.html)


# HTTP/2 连接建立  
可以详细阅读[HTTP/2笔记之连接建立](http://www.blogjava.net/yongboy/archive/2015/03/18/423570.html)，了解其中详情。  