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
**（1）数字签名——消息到底是谁写的**  
**（2）数字证书——为公钥加上数字签名**  
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

`HTTPS`可以简单理解为`HTTP + SSL/TLS`，可以实现HTTP的安全传输。    
其中SSL/TLS可以理解为`安全层协议`，位于应用层和传输层之间，SSL-Secure Socket Layer，TLS-Transport Layer Security。  
在TCP/IP协议簇模型中，HTTPS和HTTP的对比如下。    

![](/image/2018-04-23-learning-notes-network-protocol/http-https.png)

在介绍SSL/TLS相关内容之前，有必要先了解下`数字签名`和`数字证书`。

#### （1）数字签名——消息到底是谁写的  

![](/image/2018-04-23-learning-notes-network-protocol/signature.png)

Alice发送数字签名，Bob签名校验的过程：    
（1）Alice用`单向散列函数`计算消息的散列值    
（2）Alice用自己的`私钥`加密散列值，得到`数字签名`    
（3）Alice将消息和签名发送给Bob    
（4）Bob用Alice的`公钥`解密数字签名，得到解密的散列值；用`单向散列函数`计算消息的散列值    
（5）Bob比较两个散列值，如果两者一致则签名验证成功，说明消息来源于Alice    

数字签名解决的问题是：确定消息到底是谁写的。数字签名是私钥加密，公钥解密。

应用场景：

#### （2）数字证书——为公钥加上数字签名  

**基本概念**：数字证书和驾照类似，里面登记有姓名、组织、邮箱等`个人信息`，以及属于此人的`公钥`，并由认证机构施加`数字签名`。    
**解决的问题**：只要看到数字证书，我们就可以知道该公钥的确属于此人。    

**证书内容：**    
（1）签名前的证书（包含个人信息、公钥等）    
（2）签名算法（包含散列函数和加密方法，如SHA1WithRSAEncryption）    
（3）签名内容（认证机构对公钥施加的数字签名）    

**应用场景：**    
当数据需要安全传输时，服务端需要一个公钥经认证机构签名的证书，如HTTPS。    

![](/image/2018-04-23-learning-notes-network-protocol/certificate.png)

（1）Bob生成密钥对    
（2）Bob在认证机构Trent注册自己的公钥    
（3）认证机构Trent用自己的私钥对Bob的公钥施加数字签名并生成证书    
（4）Alice得到带有认证机构Trent的数字签名的证书    
（5）Alice使用认证机构Trent的公钥验证数字签名，确认Bob的公钥的合法性    
（6）Alice用Bob的公钥加密消息并发送给Bob    
（7）Bob用自己的私钥解密密文得到Alice的消息    

ps：此处有两个密钥对：Bob的密钥对和认证机构密钥对，Bob密钥对用于消息的加解密，认证机构密钥对用于数字签名及校验。    

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
