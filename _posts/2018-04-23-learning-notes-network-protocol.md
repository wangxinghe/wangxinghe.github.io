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
**（3）证书的认证机构CA**  
**（4）HTTPS握手过程**  
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

#### （3）认证机构CA  

认证机构（Certification Authoruty）是对证书进行管理的人。    

**认证机构层级：**    
认证机构是存在层级结构的。    
比如某组织从上到下的层级结构是：`东京总公司->北海道分公司->札幌办事处`，Bob是札幌办事处的员工，则该员工的公钥由札幌办事处颁发证书，札幌办事处的公钥又有北海道分公司颁发证书，北海道分公司的公钥由东京总公司颁发证书。    
如果要验证Bob的数字签名是否合法，要从上往下验证认证机构的公钥，即`东京总公司->北海道分公司->札幌办事处`。    

**根CA、自签名：**    
位于最顶层的是根CA，根CA是自签名，即自己的私钥给自己的公钥签名。    

![](/image/2018-04-23-learning-notes-network-protocol/certificate-pkl.png)

#### （4）HTTPS握手过程  

![](/image/2018-04-23-learning-notes-network-protocol/https-shakehand.png)

HTTPS握手可以理解为TCP握手＋SSL/TLS握手。    
整体来看占了3个RTT，其中TCP握手1个RTT，SSL/TLS握手2个RTT。    
RTT（Round-Trip Time）：往返时延。即从发送端发送数据开始，到发送端收到来自接收端的确认的时间。    

**基本流程：**    
（1）客户端发送一个SYN（SEQ=x）报文给服务端，进入SYN_SEND状态    
（2）服务端收到SYN报文，回应一个SYN-ACK（SEQ＝y，ACK＝x+1）报文，进入SYN_RECV状态    
（3）客户端收到服务端的SYN-ACK报文，回应一个ACK（ACK＝y+1）报文；并发起ClientHello消息（包含随机数random1，支持的加密算法）    
（4）服务端收到消息后，往客户端发送ServerHello、Certificate、ServerHelloDone（包含随机数random2，加密算法，将服务端公钥进行签名的数字证书）    
（5）客户端收到消息后，验证证书的合法性，得到服务端公钥，同时生成随机数random3，并将random3用服务端公钥加密得到preMaster，将preMaster发送给服务端，（涉及ClientKeyExchange、ChangeCipherSpec、Finished消息）。    
（6）服务端收到消息后，使用ramdom1、random2、preMaster和协商好的密码套件中的散列函数生成主密码，并发送ChangeCipherSpec、Finished消息，告诉客户端以后的通讯使用这一套密钥来进行（此时两端同时持有ramdom1、random2、preMaster及协商好的密码套件，两端分别计算主密码，再根据主密码得到对称加密的密钥）    

ps：SSL/TLS握手是个对称加密密钥协商的过程。涉及3个随机数交换和密码套件协商，从而最终生成主密码。

**优化点：**    
有个疑问。由于密钥协商的过程占了2个RTT，https握手时长是http的3倍，获取安全性的代价很高，实际上也没有必要每次建立连接都协商密钥。    
实际上在第（6）步，服务端会向客户端发一个Session Ticket，客户端收到后就可以保存下来，然后下一次握手发送ClientHello的时候将这个Session Ticket发送给服务端，服务端收到后直接告诉客户端使用上一次的密钥，这样SSL/TLS只占1个RTT，整个握手过程占2个RTT。    
当然Session Id也有类似的作用，这里不深究，可以参考下面的链接。    

### 3、HTTP2.0协议    

### 6、参考文档    

（1）[HTTP权威指南](https://book.douban.com/subject/10746113/)    
（2）[HTTPS为什么安全 & 分析HTTPS连接建立全过程](http://wetest.qq.com/lab/view/110.html)    
（3）[图解密码技术](https://book.douban.com/subject/26822106/)    
