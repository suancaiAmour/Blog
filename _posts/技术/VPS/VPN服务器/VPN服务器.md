title: VPS 创建 VPN 服务器  
date: 2017/1/26 17:18  
description: 利用 VPS 服务器创建 IKEv1 和 IKEv2 服务器  
category: VPS   
toc: true  
comments: true  

---
# 前述  
作为一名程序员，利用谷歌查找资料那是必须的，所以需要 VPN 这种东西。平常我是购买 SS VPN 来使用的，但其实那个 VPN 贵，而且不稳定，所以我就冒出了利用 VPS 服务器搭建一个自己专属的 VPN 服务器的想法，这样比较稳定且速度较快。说干就干，于是我就购买了 VPS 服务器，开始干了起来，网上有很多类似的教程，我也就根据我看到的一个教程来说，主要说说其中遇到的坑。  
我这里搭建的是 VPN 的 IKEv1 和 IKEv2 服务器。不用其他 VPN 服务器的原因，下面会讲述。  
在这里，我希望大家合法上网，在国家法律的范围内，科学上网。  
下面我就说说一些名词的解释吧。  


# 什么是 VPS
VPS 是机房搭建在国外的服务器，利用虚拟化技术，分配系统，内存，硬盘等资源给用户。因为在服务器在国外，所以我们就可以利用这个搭建 VPN 服务器，科学上网。 VPS 虚拟化的架构主要有三种，OpenVZ，KVM 和 Xen。  

## OpenVZ，KVM，Xen 区别
具体区别我也是不太明白，因为我对 Linux 一窍不通，但可以说明这三种技术的区别，因为在网上这种博客很多，我就说说我个人的见解:

* OpenVZ:  
	优点：便宜，性价比高。  
	缺点：超卖，VPS 提供商通常超卖，搭建 VPN 服务器不方便，提供商标出的性能看看就好了，不能装 Win。  
* KVM:
	优点：性能最为强悍，可以安装 Win 系统。  
	缺点：贵，如果你的 VPS 邻居装了 Win 系统，或许会影响到你的服务器。  
* Xen:
	优点：性能较强，分 pv 和 hvm 两种，pv 可以安装 Win 系统，hvm 不可以装 Win 系统。在 pv 和 hvm 中，pv 性能更强。  
	缺点：较贵。  

## 选择哪种架构的 VPS 好
这里我选择的是 OpenVZ 架构的，原因只有一个，便宜，主要在于搭个博客和 VPN 服务器不需要很高的性能，性价比这时候就体现出来了。

## 选择哪家的 VPS 好
我曾经使用过两家的 VPS，一个是 gigsgigscloud 的服务器，它的服务器位于香港，速度好，但我用它的 VPS 搭建 IKEv1 和 IKEv2 服务器总是失败，但搭建 PPTP 服务器是没问题的，所以最后我改用了另外一家的 VPS -- 搬瓦工，服务器位于洛杉矶。在这里说明一下，你在搭建 VPN 服务器的时候，必须确保 VPS 的 TUN 和 PPP 模块的开启，检测方法如下：  

1. 输入   
	``
	cat /dev/net/tun
	``   
	返回`cat: /dev/net/tun: File descriptor in bad state`说明 TUN 是开启的。  

2. 输入  
	``
	cat /dev/ppp
	``  
	返回`cat: /dev/ppp: No such device or address`或者  
	`cat: /dev/ppp: No such file or directory`是开启了 PPP 的。  
	
如果上面的两个任意一个没开启，就必须联系管理员，帮你开启这两个模块，不然，无法搭建 VPN 服务器，OpenVPN 服务器或许可以。


# 什么是 VPN 
VPN 是虚拟专用网络的简称，可以通过 VPN 从一个网络访问到另一个网络，从公网访问到局域网当中，这也就是实现科学上网的原因，我们通过访问到外国服务器（VPS）就可以访问到外国网站。

## VPN 的分类
* PPTP:最先推出的 VPN 协议，占用资源少，速度快，加密性比较弱，容易被封锁，iOS10 和 最新版 Mac OSX 已经不支持。
* L2TP:升级版的 PPTP。
* IKEv1:安全性比 PPTP 和 L2TP 高。
* IKEv2:IKEv1 的升级版，适合在移动设备使用，使用公钥证书和密码加密，安全性极高，支持硬件加速，速率极快。
* OpenVPN:开源 VPN，加密性和适应性都比较好，在大部分平台中，需要第三方软件。

## VPN 的选择
为了适应绝大部分的平台，PPTP 是不可接受的，因为它安全性较低，已经在苹果公司的平台中所抛弃了。为了更方便的使用，我这里也抛弃了 OpenVPN，因为需要安装第三方的软件。为了适应移动端，我选择了 IKEv1 和 IKEv2。


# 怎么安装 VPN 服务器
在这里，我推荐这个博客：[CentOS/Ubuntu一键安装IPSEC/IKEV2 VPN服务器](https://quericy.me/blog/699/)（[转载自quericy的博客](https://quericy.me)）。  
我说明一下我遇到的坑吧，上面的博客方法是可以使用的。一开始我使用的是 gigsgigscloud 的服务器，使用这个服务商的 VPS 搭建 PPTP 的 VPN 服务器是可以的，但我使用上面博客方法去搭建 IKEv2 和 IKEv1 服务器的时候，却出现了莫名其妙的问题，只能使用一次，当搭建完 VPN 服务器的时候，可以使用 IKEv1 连接上，但过一段时间就自动断掉，以后便无法再使用，确实很坑爹，IKEv2 根本就一开始连接不上去。我鼓捣了几天，后来发现换了搬瓦工的 VPS 就可以了。其实上还有一点小问题，就是有时候会连接不上，但是极少的情况下，我并不是这方面的专家，也就不了了之了，因为不太影响使用。
	
	