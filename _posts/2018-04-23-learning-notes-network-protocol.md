---
layout: post
comments: true
title: "基础学习笔记——网络协议篇"
description: "基础学习笔记——网络协议篇"
category: Basis
tags: [Basis]
---

**1、HTTP协议**    
**（1）HTTP报文格式－－方法／状态码等**    
**（2）3次握手／4次挥手**    
**（3）HTTP请求过程**    
**2、HTTPS协议**    
**（1）数字签名**  
**（2）证书——为公钥加上数字签名**  
**（3）HTTPS握手过程**  
**（4）证书的认证机构CA**  
**3、HTTP2.0协议**    
**4、SPDY协议**    
**5、QUIC协议**    
**6、参考文档**    


<!--more-->

### 1、HTTP协议    

![](/image/2018-04-23-learning-notes-network-protocol/protocol.png)

#### （1）HTTP报文格式－－方法／状态码等

#### （2）3次握手／4次挥手    

![](/image/2018-04-23-learning-notes-network-protocol/http-shakehand.png)

#### （3）HTTP请求过程    

### 2、HTTPS协议    

HTTPS可以简单理解为HTTP + SSL/TLS

![](/image/2018-04-23-learning-notes-network-protocol/http-https.png)

#### （1）数字签名  

![](/image/2018-04-23-learning-notes-network-protocol/signature.png)

#### （2）证书——为公钥加上数字签名  

![](/image/2018-04-23-learning-notes-network-protocol/certificate.png)


#### （3）HTTPS握手过程  

![](/image/2018-04-23-learning-notes-network-protocol/https-shakehand.png)

#### （4）证书的认证机构CA  

根CA、自签名

![](/image/2018-04-23-learning-notes-network-protocol/certificate-pkl.png)



### 3、HTTP2.0协议    

### 6、参考文档    

（1）[HTTP权威指南](https://book.douban.com/subject/10746113/)    
（2）[HTTPS为什么安全 & 分析HTTPS连接建立全过程](http://wetest.qq.com/lab/view/110.html)    
（3）[图解密码技术](https://book.douban.com/subject/26822106/)    
