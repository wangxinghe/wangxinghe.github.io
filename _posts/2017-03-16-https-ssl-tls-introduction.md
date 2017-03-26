---
layout: post
comments: true
title: "关于https的那些事儿"
description: "关于https的那些事儿"
category: Basis
tags: [Basis]
---

前一阵子在研究抓包原理，发现https抓包和普通的http抓包原理完全不同。而在了解https抓包之前，有必要先掌握https协议的通信过程。

于是在写这篇文章的时候，又复习了一遍《图解密码技术》这本书的Chapter 14关于SSL/TLS的介绍。

<!--more-->

本文按以下思路来讲解。

- 1、为什么使用HTTPS协议？    
	a. http存在的问题    
	b. http和https的关系    
- 2、什么是SSL/TLS协议？    
	a. ssl/tls解决的问题    
	b. ssl/tls使用的密码技术    
	c. ssl/tls协议的构成    
- 3、SSL/TLS通信过程    
	a. tls记录协议    
	b. tls握手协议    
	c. 总结    

## 1、为什么使用HTTPS协议？

### http存在的问题

通常我们使用http协议来实现服务器和客户端之间通信，然而http通信存在如下几个安全问题：    
（1）机密性问题。消息有可能被中间人窃听    
（2）完整性问题。消息有可能被中间人篡改    
（3）认证问题。通信对方有可能不是本人。    
当我们传输一些敏感信息时，以上安全问题可能会带来不可预料的后果。    

为解决通信过程的上述安全性问题，https协议由此出现。

### http和https的关系

简单来说，**HTTPS = HTTP + SSL/TLS**。

https在http的基础上增加了一层SSL/TLS安全协议，SSL/TLS用于承载http通信过程确保安全性。而通信双方则是基于https协议进行通信。    
其层次关系如下图：

![](./image/2017-03-16-https-ssl-tls-introduction/https.png)

## 2、什么是SSL/TLS协议？

SSL/TLS中包含SSL和TLS2种协议。	
SSL - Secure Socket Layer，安全套接层，是由网景公司设计的一种协议。	
TLS - Transport Layer Security，传输层安全，由IETF在SSL 3.0基础上设计的一种协议。	

### SSL/TLS解决的问题

SSL/TLS解决了消息传输过程中的如下安全性问题。

- 机密性问题。
- 完整性问题
- 认证问题。

### SSL/TLS使用的密码技术

为了解决存在的安全性问题，用到了如下密码技术。

- 机密性问题 ——**对称加密**
- 完整性问题——**消息认证码**
- 认证问题——**数字签名**

### SSL/TLS协议的构成

![](./image/2017-03-16-https-ssl-tls-introduction/ssl.png)

以TLS协议为例，TLS协议由**TLS握手协议**和**TLS记录协议**组成，TLS记录协议位于TLS握手协议下层。

TLS记录协议，负责消息的压缩、数据认证、加密。

TLS握手协议，包括**握手协议**、**密码规格变更协议**、**警告协议**、**应用数据协议**4个子协议。握手协议主要用来协商数据通信过程需要的共享密钥，和进行身份认证。

**说的通俗一点，HTTPS通信过程的3大安全性问题，是由TLS记录协议来保证的，所有数据需要经过记录协议进行处理再传输；而记录协议需要的各种算法和密钥是由TLS握手协议得到的。**

#### TLS记录协议

![Alt text](./image/2017-03-16-https-ssl-tls-introduction/tls记录协议.png)

对于发送端，接收高层客户数据，然后分段 -> 压缩 -> 添加MAC -> 加密。	
对于接收端，解密 -> 校验MAC -> 解压 -> 重组，然后将数据传递给高层客户。	

以发送端为例。

TLS记录协议主要过程：	
i 对消息进行分段	
ii 对分段消息进行压缩	
iii 对压缩后的消息添加消息认证码	
iv 对消息进行加密处理	
v 对加密消息添加消息头	

用到的算法及需要的信息包括：	
i 压缩算法	
ii 消息认证码（使用单向散列算法和共享密钥）	
iii 对称加密算法（对称加密算法和CBC模式的初始化向量）	

上面用到的算法由通信双方在TLS握手协议过程中协商决定。	

**个人思考：**		
上图中**类型**指的是TLS握手协议的4个子协议之一。	
对于握手协议，由于此时还没有协商好加密算法和共享密钥，因此这部分通信过程是 **无密码套件**，所以在我看来，当类型值为握手协议时，对应的通信时的记录协议中使用的压缩算法、消息认证码、对称加密算法都是空。此时是不安全的，通信数据可能会被窃听，这也是为什么握手过程中不是直接传应用数据通信的各种密钥，而采用公钥加密或Diffie-Hellman密钥交换技术的原因。	
公钥加密和Diffie-Hellman密钥交换技术可以理解成一种**动态密钥配送**方案。更具体的信息可以翻阅《图解密码技术》这本书。

#### TLS握手协议

**握手协议**

握手协议的职责是生成通信过程所需的共享密钥和进行身份认证。这部分使用无密码套件，为防止数据被窃听，通过公钥密码或Diffie-Hellman密钥交换技术通信。

**密码规格变更协议**

用于密码切换的同步，是在握手协议之后的协议。握手协议过程中使用的协议是“不加密”这一密码套件，握手协议完成后则使用协商好的密码套件。

**警告协议**
当发生错误时使用该协议通知通信对方，如握手过程中发生异常、消息认证码错误、数据无法解压缩等。

**应用数据协议**
通信双方真正进行应用数据传输的协议，传送过程通过TLS应用数据协议和TLS记录协议来进行传输。

## 3、SSL/TLS通信过程

**握手协议过程**

TLS握手协议过程，包括4次握手。

用对话形式来理解：

**（1）Client -> Server**

	1.你好，我能理解的密码套件有RSA/3DES或DSS/AES，请问我们使用哪一种密码套件进行通信呢？	

**（2）Server -> Client**

	2.你好，我们就用RSA/3DES来进行通信吧。
	3.这是我的证书（非匿名通信时）
	4.我们用这些信息来进行密钥交换吧。（当证书的信息不足时）
	5.对了，请给我看一下你的证书吧
	6.问候到此结束

**（3）Client -> Server**

	7.这是我的证书
	8.这是经过加密的预备主密钥
	9.我就是持有客户端证书的本人
	10.好，现在我要切换密码了
	11.握手协议到此结束

**（4）Server -> Client**

	12.好，现在我要切换密码了
	13.握手协议到此结束

用图形交互来理解：

![Alt text](./image/2017-03-16-https-ssl-tls-introduction/握手协议过程1.png)

分别对握手过程的每一步进行细化。

**(1) ClientHello**    
客户端向服务端发送如下信息：    
可用的版本号    
当前时间    
`客户端随机数`    
会话ID    
可用的密码套件清单    
可用的压缩方式清单    

------------------------------------

**(2) ServerHello**
使用的版本号
当前时间
`服务器随机数`
会话ID
使用的密码套件
使用的压缩方式

**(3) Certificate**    
证书清单    

**(4) ServerKeyExchange**    
RSA情况（公钥密码参数N-modulus E-exponent、散列值）    
Diffie-Hellman密钥交换情况（密钥交换参数P-prime modulus G-generator 服务端Y-Gx mod P、散列值）    

**(5) CertificateRequest**    
服务器能理解的证书类型清单    
服务器能理解的认证机构名称清单    

**(6) ServerHelloDone**    


-----------------------------------

**(7) Certificate**    
证书清单    

**(8) ClientKeyExchange**    
RSA情况（经过加密的`预备主密钥`）    
Diffie-Hellman密钥交换情况（客户端Y-Gx mod P）    

**(9) CertificateVerify**    
数字签名    

**(10) ChangeCipherSpec**    
**(11) Finished**    

-----------------------------------

**(12) ChangeCipherSpec**    
**(13) Finished**    

之后切换到应用数据协议。

**密码规格变更协议**    

	客户端：好，我们按照刚才的约定切换密码吧。1、2、3！

**警告协议**    

	服务器：刚才的消息无法正确解密哦！

**应用数据协议**    
将应用数据传达给通信对象。

我们再来捋一捋，从开始握手到握手完成，所需的对称密码的密钥、消息认证码的密钥、CBC模式使用的初始化向量IV是怎么产生的。

`客户端随机数 -> 服务器随机数 -> 预备主密码 -> 主密码 -> （对称密码的密钥、消息认证码的密钥、CBC模式的初始化向量）`

再回过头看：    
1.`客户端随机数`在ClientHello过程由客户端发送给服务器。    
2.`服务器随机数`在ServerHello过程由服务器发送给客户端。    
3.`预备主密码`在ClientKeyExchange过程由客户端使用服务器证书的公钥加密后发送给服务器，因为服务器在Certicifate过程已经将证书发送给客户端，然后服务器使用证书密钥进行解密即可得到预备主密钥。预备主密码是客户端产生的随机数。    

因此对于客户端和服务器，客户端随机数、服务器随机数、预备主密码都已知。

4.而`主密码`由客户端随机数、服务器随机数、预备主密钥共同计算产生。

5.根据主密码可以计算出`对称密码的密钥、消息认证码的密钥、CBC模式的初始化向量`。

用流程图表示这一过程。

![Alt text](./image/2017-03-16-https-ssl-tls-introduction/主密码.png)

**ServerKeyExchange的作用：**	

TLS握手过程的每一步的作用大部分都搞明白了，然而ServerKeyExchange这一步还有点疑惑，为啥要传这些参数呢？	
Google了下，搜到如下解释：    
https://security.stackexchange.com/questions/79482/whats-the-purpose-of-server-key-exchange-when-using-ephemeral-diffie-hellman

	In Diffie-Hellman, the client can't compute a premaster secret on its own; both sides contribute to computing it, so the client needs to get a Diffie-Hellman public key from the server. In ephemeral Diffie-Hellman, that public key isn't in the certificate (that's what ephemeral Diffie-Hellman means). So the server has to send the client its ephemeral DH public key in a separate message so that the client can compute the premaster secret (remember, both parties need to know the premaster secret, because that's how they derive the master secret). That message is the ServerKeyExchange.

大致意思是，ServerKeyExchange不是必须的。对于RSA来说，服务端发送Certificate时，提供了证书且证书中包含了公钥，所以加密预备主密钥时直接使用公钥；而对于Diffie-Hellman来说，Certificate过程的证书没有包含公钥，需要额外提供公钥相关信息，用来加密预备主密钥。

下一节需要抓包来看具体数据了。

后话：    
在我看来，学习这一过程是孤独而艰难的。需要花很多时间去学习新东西，而且还不是一学就能学明白，而且学过的东西过段时间又容易忘记。朋友和我说的是，多思考为什么，不要去记忆。