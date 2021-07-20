---
layout: post
comments: true
title: "基础学习笔记——网络协议篇"
description: "基础学习笔记——网络协议篇"
category: Basis
tags: [Basis]
---

**1、HTTP协议**    
**（1）HTTP报文**    
**（2）TCP3次握手／4次挥手**    
**（3）HTTP请求过程**    
**（4）HTTP协议缺点**    
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

**TCP/IP协议簇分层模型：** 应用层、传输层、网络层、链路层、物理层。    
HTTP协议位于应用层，一个HTTP请求的流转方向为：    
发送端，`应用层->传输层->网络层->链路层->物理层`，且每一层都会增加首部（物理层除外）；    
接收端，`物理层->链路层->网络层->传输层->应用层`，每一层会删除首部（物理层除外）。    

**HTTP特性：**    
（1）HTTP的默认端口是80    
（2）`无连接`、`无状态`。    

**无连接：**指的是在非keep-alive模式下，每个请求／应答客户端和服务端都要新建一个连接，请求／应答完成之后立即断开。    
**无状态：**指的是每次请求都是独立的，上一次请求与下一次请求之间没有必然关系。    

#### （1）HTTP报文    

**报文分类：**`请求报文`和`响应报文`2类  
**报文结构：**`起始行`，`首部`，`实体`3部分        

**请求报文格式：**    

    <method> <request-URL> <version>
    <headers>

    <entity-body>

**响应报文格式：**    

    <version> <status> <reason-phrase>
    <headers>

    <entity-body>

**报文元素：**    
（1）`<method>`：方法。    

（2）`<request-URL>`：请求路径，服务器上的某个相对路径。    

（3）`<version>`：HTTP协议版本号    

（4）`<headers>`：首部，由key: value组成的键值对，分为`通用首部`、`请求首部`、`响应首部`    

（5）`<entity-body>`：实体，请求或响应发送的数据    

（6）`<status>`：响应的状态码，是一串数字    

（7）`<reason-phrase>`：响应的状态说明    

**常见的7个方法：**    
`GET`：向服务器请求某个资源    
`HEAD`：向服务器请求某个资源，但是只返回首部信息，不返回实体    
`POST`：向服务器发送表单数据    
`PUT`：向服务器的指定资源位置上传最新内容    
`DELETE`：向服务器请求删除<request-URL>所指定的资源        
`OPTIONS`：请求服务器告知其支持的各种方法    
`TRACE`：允许客户端在最终将请求发送给目的服务器时，看看它变成了什么样子（可能在到达目的服务器前会经过代理等）。目的服务器会弹出一条TRACE响应，并在响应实体中携带收到的上一站传过来的请求报文，这样客户端就能查看到，在请求／响应链上，原始报文是否以及如何被修改的。    

**常见的请求首部：**    
（1）`Accept`：告诉服务器，客户端可以接受的资源类型，如text/html，application/xhtml+xml，image/webp等    
（2）`Accept-Encoding`：客户端支持什么样的压缩算法，如gzip, deflate, br    
（3）`Accept-Language`：告诉服务器，客户端希望服务器返回的语言格式，如zh-CN,zh;q=0.9,en;q=0.8,zh-TW;q=0.7    
（4）`Cache-control`：指定请求和响应遵循的缓存机制，如no-cache、no-store、max-age、max-stale等    
（5）`Connection`：告诉服务器打开的是否是tcp长链接，keep-alive长连接，close非长连接    
（6）`Cookie`：服务器存储在客户端的数据，是一组键值对，用；隔开    
（7）`Host`：服务器的主机和端口号    
（8）`User-Agent`：用户代理，标识客户端的操作系统（类型及版本）、CPU类型、浏览器（类型、版本号、渲染引擎、语言等），如Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36    
（9）`Referer`：来源页面    
（10）`if-modified-since`：上次服务器响应请求后(服务器返的Date时间值后)资源是否发生过修改。如果没有修改过资源，服务器就会返回304 not modified，如果修改了，就返回资源。
（11）`if-none-match`：和if-modified-since类似    
（12）`Range`：告诉服务器请求资源的一段如bytes=100-1000，用于断点下载    

**常见的响应首部：**    
（1）`Content-Encoding`：服务端发送的资源的编码方式，如gzip    
（2）`Content-Type`：服务端发送的资源的文件类型及编码，如text/html;charset=UTF-8    
（3）`Date`：服务器当前时间，如Sun, 21 Sep 2014 06:18:21 GMT    
（4）`Expires`：缓存有效时间，在这个时间之前缓存有效。如Sun, 1 Jan 2000 01:00:00 GMT    
（5）`Transfer-Encoding`：服务器发送的资源的方式是否分块发送，chunked分块。一般分块发送的资源都是服务器动态生成的，在发送时还不知道发送资源的大小，所以采用分块发送，每一块都是独立的，独立的块都能标示自己的长度，最后一块是0长度的，当客户端读到这个0长度的块时，就可以确定资源已经传输完了。    
（6）`Content-Length`：服务端发送的资源的长度。如果服务器响应头中没有Transfer-Encoding：chunked，也就是说如果资源不是分块传输的，那么这个字段就是必须的，因为如果服务器端不告诉客户端资源的长度，那么在Connection：keep-alive,也就是说客户端与服务器端是长连接时，客户端就无法确定资源在什么时候结束。所以这个响应头对于静态资源时必须的。    

**状态码：**    
（1）1XX（100-101）：信息提示    
（2）2XX（200-206）：成功    
（3）3XX（300-305）：重定向    
（4）4XX（400-415）：客户端错误    
（5）5XX（500-505）：服务器错误    

**常见的状态码：**    
（1）200 OK 客户端请求成功    
（2）301 Moved Permanently 请求永久重定向
（3）302 Moved Temporarily 请求临时重定向    
（4）304 Not Modified 文件未修改，可以直接使用缓存的文件。    
（5）400 Bad Request 由于客户端请求有语法错误，不能被服务器所理解。    
（6）401 Unauthorized 请求未经授权。这个状态代码必须和WWW-Authenticate报头域一起使用    
（7）403 Forbidden 服务器收到请求，但是拒绝提供服务。服务器通常会在响应正文中给出不提供服务的原因    
（8）404 Not Found 请求的资源不存在，例如，输入了错误的URL    
（9）500 Internal Server Error 服务器发生不可预期的错误，导致无法完成客户端的请求。    
（10）503 Service Unavailable 服务器当前不能够处理客户端的请求，在一段时间之后，服务器可能会恢复    

**GET和POST比较：**    
（1）GET用于信息获取，是`安全和幂等`的；POST用于发送数据，可能修改变服务器上的资源        
（2）GET可提交的数据量受到URL`长度的限制`；POST是没有大小限制的。这个限制是特定的浏览器及服务器对它的限制（通常是2K左右），而非HTTP协议    
（3）GET的`参数`在< request-URL>里；POST的参数在实体里    

下面分别是GET和POST的请求报文：    
     
     GET /books/?sex=man&name=Professional HTTP/1.1
     Host: www.example.com
     User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
     Gecko/20050225 Firefox/1.0.1
     Connection: keep-alive
 
 
     POST / HTTP/1.1
     Host: www.example.com
     User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
     Gecko/20050225 Firefox/1.0.1
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 40
     Connection: keep-alive

     sex=man&name=Professional  


#### （2）TCP3次握手／4次挥手    

![](/image/2018-04-23-learning-notes-network-protocol/tcp.png)

**TCP的特点：**    
（1）面向连接的传输层协议    
（2）传输过程是可靠的。无差错、不丢失、不重复、按序到达    
（3）全双工通信    
（4）面向字节流    

实现可靠传输的关键点：    
TCP发送一个报文段，需要对方回复一个ACK来确认报文段是否发送成功，如果没有收到ACK则需要进行报文重传。    

单工/半双工/全双工：    
[https://zh.wikipedia.org/wiki/%E9%9B%99%E5%B7%A5](https://zh.wikipedia.org/wiki/%E9%9B%99%E5%B7%A5)    
[https://blog.csdn.net/chaofanwei/article/details/13274815](https://blog.csdn.net/chaofanwei/article/details/13274815)    


**报文格式：**    
关于TCP报文格式，本文暂时不讲，只介绍几个重要的字段：    
（1）`序列号Seq`：占4个字节，TCP连接中传输的每一个字节都按顺序编号    
（2）`确认号Ack`：占4个字节，是期望收到对方下一个报文段的第一个数据字节的序号    
（3）`SYN`：同步标志位，在连接建立时用来同步序列号，当SYN＝1时表明这是一个连接请求或连接接受请求    
（4）`ACK`：确认标志位，仅当ACK＝1时确认号字段才有效    
（5）`FIN`：终止标志位，用来释放连接，当FIN＝1时表明发送方的数据已发送完毕，并要求释放连接    

**整体流程：**    
1、连接建立阶段：    
（1）Client向Server发出连接请求报文段，此时SYN＝1，Client进入SYN_SEND状态    
（2）Server收到连接请求报文段后，同意建立连接并向Client发送确认，此时SYN=1，ACK=1，Server进入SYN_RECV状态    
（3）Client收到Server的确认后，向B发送确认报文，ACK=1，Client进入ESTABLISHED状态；若Server收到Client的确认报文，则Server也进入ESTABLISHED状态    

2、数据传输阶段：    
Client和Server都可以发送和接收数据，当发送方向接收方发送数据时，接收方需要发送ACK确认报文，若发送方没有收到ACK确认报文则需要重传    

3、连接释放阶段：    
发起方可以是Client或Server，以Client为例。    
（1）Client向Server发送连接释放报文段，此时FIN=1，Client进入FIN_WAIT_1状态    
（2）Server收到连接释放报文段后发出确认，此时ACK=1，Server进入CLOSE_WAIT状态，从Client到Server这个方向的连接就释放了（半关闭状态，即Client已经没有数据要发送了，但Server发送数据Client仍要接收）；Client收到确认报文段后进入FIN_WAIT_2状态    
（*）若Server还有数据报文段要发送，则向Client发送，Client收到后进行ACK应答    
（3）Server向Client发送连接释放报文段，此时FIN=1，ACK=1，Server进入LAST_ACK状态，等待Client的确认    
（4）Client收到连接释放报文段后，发送确认报文段，此时ACK＝1，Client进入TIME_WAIT状态；Server收到确认报文后进入CLOSED状态。如果Client在2MSL时间内没收到Server的重传报文则进入CLOSED状态，至此连接释放了。    

MSL：Maximum Segment Lifetime，最长报文段寿命，可根据实际需求设定。    

**Seq和Ack：**    
序列号Seq取值规则：    
序列号Seq对于某个端来说都是累加的，其取值只和本端的上一个报文的Seq有关。    
两端建立连接时的初始值是随机的；    
本端报文Seq取值 ＝ 本端上一个报文Seq取值 ＋ 本端上一个报文的Len(包大小，TCP Segment Length)    
SYN、SYN-ACK、FIN、FIN-ACK报文段，不能携带数据，Len＝0，但是根据TCP规范要消耗一个序列号    
ACK报文段可以携带数据也可以不携带，一般都不携带数据，本身会消耗一个序列号。不携带数据时，会影响本端下一个报文段的Seq，即下一个Seq取值和ACK一致。    

用WireShark看数据时，Length属性对应的是以太帧数据的大小，TCP Segment Length属性对应的是TCP段报文大小，两者之间相差头部（Ethernet Header +IP Header ＝ 54字节），TCP报文中的Len就是TCP Segment Length。    
`Frame=Ethernet Header +IP Header +TCP Header +TCP Segment Data`

延伸阅读：[TCP通信流程解析](https://blog.csdn.net/phunxm/article/details/5836034)    

确认号Ack取值规则：    
确认号是针对发送端发送一个报文段，然后接收端发一个确认报文告诉发送方自己收到了该报文，因此Ack取值和发送方的Seq有关        
Ack ＝ 发送报文段的Seq ＋ Len    


**为什么是3次握手／4次挥手：**    
为什么是3次握手？（答案来源于谢希仁《计算机网络》第五版。

![](/image/2018-04-23-learning-notes-network-protocol/why-tcp-shakehand-3times.png)    

按道理通过前2次握手，Client和Server就知道了对方已确认建立连接。之所以Client还需要发个ACK是因为：    
之前发送的已经失效的SYN消息由于在某些网络结点滞留的原因，在连接释放后才到达Server，这是如果Server回复了SYN-ACK报文，连接就又建立了。所以Client收到SYN-ACK后，还需要通过发送一个ACK确认这个是不是已失效的。    

为什么是4次挥手？    
这里主要是想问，为啥第2、3次不合并为一次握手，和建立连接过程一样。原因是：    
Client发了FIN-ACK后，表明Client端没有数据要发送了，但是Server收到报文后可能还有要发送的数据，如果第2、3次合并为一次FIN-ACK，那么没有合适的时机发送数据了，因此需要先发一个ACK，然后传数据，然后发送FIN-ACK。    

**为什么要等2MSL：**    

![](/image/2018-04-23-learning-notes-network-protocol/2msl.png)    

**补充：**    
网上有很多讲TCP建立连接和断开连接的文章，然而讲的很笼统，关键的地方没有讲清楚，没法做到融汇贯通。        

最后我还是看谢希仁的《计算机网络》第五版，再结合下面两篇博文搞明白的。    

[图解TCP传输过程（三次握手、数据传输、四次挥手）](https://www.polarxiong.com/archives/%E5%9B%BE%E8%A7%A3TCP%E4%BC%A0%E8%BE%93%E8%BF%87%E7%A8%8B-%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B-%E6%95%B0%E6%8D%AE%E4%BC%A0%E8%BE%93-%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.html)    这篇文章对整个流程以及seq,ack的理解是比较到位的。    

[就是要你懂 TCP－阿里中间件团队博客](http://jm.taobao.org/2017/06/08/20170608/)    这篇文章抓住了TCP的核心，通过WireShark抓包分析来验证seq,ack的值，不过关于为什么是3次握手/4次挥手理解不太到位。    

滑动窗口、拥塞控制

#### （3）HTTP请求过程    

从浏览器中输入URL到页面展示的全过程    

[What really happens when you navigate to a URL](http://igoro.com/archive/what-really-happens-when-you-navigate-to-a-url/)    
[https://github.com/skyline75489/what-happens-when-zh_CN](https://github.com/skyline75489/what-happens-when-zh_CN)    
[从输入URL到页面加载完成的过程中都发生了什么事情？](http://fex.baidu.com/blog/2014/05/what-happen/)        

#### （4）HTTP协议缺点    

keep-alive、pipeline、SPDY

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

**应用场景：**    
安全信息公告、软件下载、公钥证书、SSL／TLS

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

附上之前写的一篇HTTPS的文章：[关于https的那些事儿
](http://mouxuejie.com/blog/2017-03-16/https-ssl-tls-introduction/)

### 3、HTTP2.0协议    

### 4、SPDY协议   

### 5、QUIC协议    

### 6、参考文档    

（1）[HTTP权威指南](https://book.douban.com/subject/10746113/)    
（2）[计算机网络](https://book.douban.com/subject/2970300/)
（3）[图解密码技术](https://book.douban.com/subject/26822106/)    
（4）[HTTPS为什么安全 & 分析HTTPS连接建立全过程](http://wetest.qq.com/lab/view/110.html)    
