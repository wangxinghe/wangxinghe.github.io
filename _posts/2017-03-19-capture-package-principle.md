---
layout: post
comments: true
title: "某学姐带你了解抓包原理"
description: "某学姐带你了解抓包原理"
category: Basis
tags: [Basis]
---


研究这个专题，起源于某天我在一个微信群看到有人讨论中间人攻击，而我自己工作中也经常使用抓包工具。对于工具，我一向都是只知道使用而不去了解实现原理。也是在某个不起眼的瞬间，我领悟到我需要比别人学东西更深入才能证明自己更牛逼。

我打算按如下思路来讲解。    

<!--more-->

- 1、抓包工具使用    
     a. 抓包常用工具    
     b. 如何配置及抓包    
- 2、抓包原理分析    
    a. http抓包原理    
    b. https抓包原理    

其中关于抓包工具使用部分，会用较少篇幅过一遍，毕竟网上资料一大堆。

## 1、抓包工具使用

### 抓包常用工具

目前常用的抓包工具有Charles、Wireshark和Fiddler，也可以使用tcpdump命令抓包。    	
Mac开发环境一般使用Charles，Window开发环境则使用Wireshark和Fiddler。事实上，我们到官网去看时会发现这些抓包工具同时有Mac/Windows/Linux版本。

Charles下载地址 [https://www.charlesproxy.com/download/](https://www.charlesproxy.com/download/)    
Wireshark下载地址 [https://www.wireshark.org/#download](https://www.wireshark.org/#download)    
Fiddler下载地址 [https://www.telerik.com/download/fiddler](https://www.telerik.com/download/fiddler)    

### 如何配置及抓包

以Fiddler为例，Fiddler运行机制是在本机监听8888端口的http/https网络请求。可以抓浏览器、Android、iOS和其他设备的数据。    
Fiddler默认作为系统代理，浏览器的设置选项中使用的代理也是系统代理，因此可以抓到浏览器的请求。对于Android/iOS移动设备，则需要将Wi-Fi代理设置为Fiddler所在主机ip和监听端口8888即可。然后就可以开心滴抓http包了。

如果想抓https的包，需要在Fiddler设置里支持https，同时Android/iOS设备需要下载并安装Fiddler Root Certificate根证书。

具体的配置及抓包，大家可以阅读这几篇文章。    
Fiddler抓包教程：    
(1) [http://www.jianshu.com/p/99b6b4cd273c](http://www.jianshu.com/p/99b6b4cd273c)    
(2) [http://www.trinea.cn/android/android-network-sniffer/](http://www.trinea.cn/android/android-network-sniffer/)    
tcpdump & Wiresharp抓包：[http://www.trinea.cn/android/tcpdump_wireshark/](http://www.trinea.cn/android/tcpdump_wireshark/)    
Charles抓包教程：[http://www.jianshu.com/p/9822e3f28f0a](http://www.jianshu.com/p/9822e3f28f0a)    
Android如何导入证书：[http://abool.blog.51cto.com/8355508/1429700](http://abool.blog.51cto.com/8355508/1429700)    

## 2、抓包原理分析

### http抓包原理

抓包方式有2种：一种是**代理模式**，一种是**网卡混杂模式**。

通过代理软件抓包的原理：

代理相当于一个中间人。客户端发送的所有请求需要先经过代理，然后再由代理发送给服务器；服务器对请求的响应由代理拦截，再由代理返给客户端。

对于客户端来说，代理扮演服务器的角色；对于服务器来说，代理扮演客户端角色。

如下图所示：

![Alt text](/image/2017-03-19-capture-package-principle/http.png)

对于黑客来说，一般使用网卡混杂模式。默认情况下，网卡会过滤掉目标地址不是自己MAC的数据包，当黑客将自己的电脑网卡设置成混杂模式时，则会收到经过网卡的所有数据包，从而污染ARP，同时告诉别人本机就是你要访问的网址，这样就能抓到包。

混杂模式，百度百科的解释是：

是指一台机器能够接收所有经过它的数据流，而不论其目的地址是否是他。是相对于通常模式而言的，这被网络管理员使用来诊断网络问题，但是也被无认证的想偷听网络通信的人利用。

[http://baike.baidu.com/link?url=rouuowDaArpAnwIB55CPXhk8isBEhsoeJw_VyGX6kBGPcBCV58SDD2iiSQwFuP81q1rS9YyykJ7pZbv_6KpPfVF1VX_eihWNX8xNmBTziJLhqNkx7Xj--HIHK-fcq4N6](http://baike.baidu.com/link?url=rouuowDaArpAnwIB55CPXhk8isBEhsoeJw_VyGX6kBGPcBCV58SDD2iiSQwFuP81q1rS9YyykJ7pZbv_6KpPfVF1VX_eihWNX8xNmBTziJLhqNkx7Xj--HIHK-fcq4N6)

### https抓包原理

上篇文章讲了https协议的通信原理。简单来说就是客户端和服务器先经过一个握手阶段协商好通信密钥，然后使用这个密钥对真正数据进行加解密传输实现安全通信。

通过代理软件抓包，代理相当于一个中间人。客户端和服务器之间的通信都要经过代理，对于代理来说整个通信是透明的。

由于https的握手阶段会有证书认证，那么代理是怎么认证通过的呢？

我们结合下图来讲解。

![Alt text](/image/2017-03-19-capture-package-principle/https.png)

握手阶段包括4次握手。

#### 1、Client -> MiddleMan -> Server

**(1) ClientHello**		
客户端发送会话ID、随机数Random_C、支持的所有压缩方式、支持的所有密码套件等；		
中间人获取到这些信息；			
中间人将这些信息发给服务器。

#### 2、Server -> MiddleMan -> Client

**(2) ServerHello**		
服务器发送会话ID、随机数Random_S、选择的压缩方式、选择的密码套件等；	
中间人获取到这些信息；	
中间人将这些信息发给客户端。

**(3) Certificate**		
服务器发送数字证书（服务器公钥＋认证机构数字签名）；	
中间人获取到服务器证书的公钥；	
中间人将自己伪造的数字证书（伪造的公钥＋伪造的数字签名）发给客户端，由于客户端已经安装了代理的根证书并且将其设置成可信任，因此会通过根证书的公钥验证伪造的数字签名，确认伪造的公钥的合法性，从而能够认证通过。伪造的数字签名，我个人理解是用根证书的私钥进行加密得到，纯属个人理解。

PS：	
所谓数字签名即用私钥加密，公钥解密，参看《图解密码技术》Chapter 9。	
数字证书即用经过这部分可以看《图解密码技术》Chapter 10。

**(4) ServerKeyExchange**		
服务器发送公钥相关信息；	
中间人获取到这些信息；	
中间人将这些信息发给客户端。

**(5) CertificateRequest**	
服务器发送证书请求消息（包含自己能理解的证书类型清单、能理解的认证机构名称清单）；		
中间人获取到这些信息；	
中间人将这些信息发给客户端。

**(6) ServerHelloDone**		
服务器发送问候结束消息；	
中间人获取到这些信息；	
中间人将这些信息发给客户端。

#### 3、Client -> MiddleMan -> Server

**(7) Certificate**	
客户端发送数字证书（客户端公钥＋认证机构数字签名）；		
中间人获取到客户端证书的公钥；	
中间人将自己伪造的数字证书（伪造的公钥＋伪造的数字签名）发给服务器。

个人理解，这一步就是发送客户端证书，客户端身份认证是通过CertificateVerify。

**(8) ClientKeyExchange**		
客户端发送中间人公钥加密的预备主密码；			
中间人获取到这些信息；		
中间人用私钥解密得到预备主密码，然后伪造一个预备主密码并用服务器公钥加密后发给服务器。

**(9) CertificateVerify**		
客户端发送认证确认消息（证明含有客户端证书的私钥）	
中间人获取到这些信息；	
中间人利用客户端证书的公钥解密从而验证客户端数字签名，然后中间人发送伪造的认证确认消息，服务器利用中间人伪造的证书公钥解密从而验证中间人数字签名。

个人理解，这一步就是对客户端身份进行认证。

**(10) ChangeCipherSpec**		
客户端发送切换密码消息；	
中间人获取到这些信息；	
中间人将这些信息发给服务器。

**(11) Finished**		
客户端发送Finished消息（用通信密钥加密固定内容）；		
中间人获取到这些信息；	
中间人将这些信息发给服务器。

#### 4、Server -> MiddleMan -> Client

**(12) ChangeCipherSpec**		
服务器发送切换密码消息；	
中间人获取到这些信息；	
中间人将这些信息发给客户端。

**(13) Finished**		
服务器发送Finished消息；	
中间人获取到这些信息；	
中间人将这些信息发给客户端。

再来捋一下以上步骤。

**中间人密码信息获取：**	
		
在ClientHello过程，获得Random_C		
在ServerHello过程，获得Random_S		
在服务器Certificate过程，获得服务器公钥		
在客户端Certificate过程，获得客户端公钥		
在ClientKeyExchange过程，获得中间人公钥加密的预备主密码，使用中间人私钥解密得到预备主密码。

再根据Random_C、Random_S、预备主密码得到客户端中间人通信的主密码master key1。

同理根据Random_C、Random_S、伪造的预备主密码得到中间人服务器通信的主密码master key2。

**中间人身份认证：**	
	
在服务器**Certificate**过程，中间人向客户端发送伪造的公钥和伪造的数字签名，客户端利用根证书的公钥解密数字签名，证明伪造的公钥的合法性，从而完成中间人的身份认证。

在客户端**Certificate**过程，中间人向服务器发送伪造的数字证书，服务器得到伪造的数字证明的公钥；在客户端**CertificateVerify**过程，中间人向服务器发送经过伪造的数字证书私钥签名的认证确认消息，服务器利用伪造的公钥解密，从而完成中间人的身份认证。

可见，https抓包相当于客户端－中间人，中间人－服务器 两部分https通信过程，这两部分的密码是不同的。关于身份认证，根证书是信任链的起点。

搜到一个讲的很详细的链接 [http://www.jianshu.com/p/c03f47e7b9de](http://www.jianshu.com/p/c03f47e7b9de)