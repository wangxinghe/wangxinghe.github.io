---
layout: post
comments: true
title: "对REST API的理解"
description: "对REST API的理解"
category: Server
tags: [Server]
---

1. What is REST?        
2. REST Architectural Constraints        
3. How to design a REST API?        

<!--more-->

作为程序员，REST这个概念应该很熟悉。今天主要讲一下到底什么叫REST？什么又是RESTful API？        
REST这个概念最早由Roy Fielding的一篇博士论文提出，后面其他所有文章或解释都是基于这篇论文。        
[https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)        

## 1. What is REST?

REST全称`Representational State Transfer`，中文翻译为`可表示的状态转化`，是一种适用于分布式超媒体系统的架构设计风格。        
`the Representational State Transfer (REST) architectural style for distributed hypermedia systems`        
要理解REST，我们需要先理解一些数据元素，Resource / Representation / Components / Hypermedia        
![Alt text](./1572158795803.png)

（1）Representation        
`A representation is a sequence of bytes, plus representation metadata to describe those bytes. Other commonly used but less precise names for a representation include: document, file, and HTTP message entity, instance, or variant.`

Representation是一串携带信息的二进制字节码和元数据，比如文档、文件、HTTP消息实体、实例对象或变量等。

（2）Resource        
`The key abstraction of information in REST is a resource. Any information that can be named can be a resource: a document or image, a temporal service (e.g. "today's weather in Los Angeles"), a collection of other resources, a non-virtual object (e.g. a person), and so on.`

Resource是REST中重要的一个概念。        
任何能被命名的信息都可以称作Resource，比如电子文档、图片、天气信息、某些资源的集合、或者某个具体的实物等。

（3）Components        
![Alt text](./1572150060448.png)

Components可以理解为网络通信链路中一个个节点，如代理、网关、服务器等

（4）Hypermedia        
Hypermedia，即超媒体，WWW就属于超媒体。要理解什么是超媒体，我们先搞清楚Hyperlink / Hypertext / Hypermedia这几个概念。

wikipedia给出了如下解释：
 
`Hyperlink is a reference to data that the reader can follow by clicking or tapping. A hyperlink points to a whole document or to a specific element within a document. `

`Hypertext is text displayed on a computer display or other electronic devices with references (hyperlinks) to other text that the reader can immediately access.`

`Hypermedia, an extension of the term hypertext, is a nonlinear medium of information that includes graphics, audio, video, plain text and hyperlinks. `

Hyperlink：超链接，即通过点击可以跳转到某个页面或页面中指定元素的链接。        
Hypertext：超文本，即包含超链接的文本。        
Hypermedia：超媒体，即包含图像、音频、视频、文本、超链接的非线性信息媒介。        

简单来说，这3者是包含关系：Hypermedia > Hypertext > Hyperlink

（5）REST        

`REST components perform actions on a resource by using a representation to capture the current or intended state of that resource and transferring that representation between components.`

Resource通过Representation携带相应状态信息，Components之间可以传输Representation，Components可以对Resource执行操作。

### 一段话总结

前面废话这么多，那么到底什么是REST呢？

REST是一种适用于分布式超媒体系统的架构设计风格。具体来说在超媒体系统中，
电子文档、图片、天气信息、某些资源的集合、或者某个具体的实物等资源，通过二进制字节码和元数据的形式表示资源的状态信息，这些状态信息可以在系统各节点间进行传输和转化。

Resource通过URI的形式呈现，Resource通过二进制字节码和元数据描述自身状态，状态信息可以在系统各个节点之间传输和转化，系统中各个节点通过URI对Resource进行访问并执行CRUD（Create/Read/Update/Delete）操作。

## 2. REST Architectural Constraints

参考 [https://restfulapi.net/rest-architectural-constraints/](https://restfulapi.net/rest-architectural-constraints/)

(1) Uniform interface (统一的接口)        
一个Resource对应一个URI, 且访问Resource的接口规范统一        

(2) Client–server (C/S模式)        
Client和Server相互独立, Client通过URI访问Server资源        

(3) Stateless (无状态)        
Client自己维护程序的状态, 而不是将状态保存到Server        

(4) Cacheable (缓存机制)        
Resource支持缓存机制, 可以在Client或Server实现缓存        

(5) Layered system (分层系统)        
API部署, 数据存储, 身份认证等可以在不同的Server端进行, Client和Server的通信链路可能会经过几个中间Server才会到达目的Server

(6) Code on demand(可选)         
Resource不一定是JSON/XML的静态资源, 也可以是一段可执行的代码

## 3. How to design a REST API?

参考 [https://restfulapi.net/rest-api-design-tutorial-with-example/](https://restfulapi.net/rest-api-design-tutorial-with-example/)

RESTful API命名规范

(1) 资源命名为名词, 可以是单个资源或集合. 不要使用动词, CRUD操作通过HTTP Method体现

	HTTP GET http://api.example.com/device-management/managed-devices
	HTTP POST http://api.example.com/device-management/managed-devices
	HTTP GET http://api.example.com/device-management/managed-devices/{id}
	HTTP PUT http://api.example.com/device-management/managed-devices/{id}
	HTTP DELETE http://api.example.com/device-management/managed-devices/{id}

HTTP Method和CRUD的对应关系:        
POST: Create        
GET: READ        
PUT: Update        
DELETE: Delete        

(2) 使用小写字母

	HTTP://API.EXAMPLE.ORG/my-folder/my-doc  //DO NOT USE
	http://api.example.org/My-Folder/my-doc  //DO NOT USE
	http://api.example.org/my-folder/my-doc

(3) 过长的名词, 可通过`-`连接, 不要使用`_`

	http://api.example.com/inventory_management/managed_entities/{id}  //DO NOT USE
	http://api.example.com/inventory-management/managed-entities/{id}  

(4)资源可以包含子资源,  `/`表示资源的层次关系, URI末尾不要使用`/`
	
	http://api.example.com/device-management/managed-devices/  //DO NOT USE
	http://api.example.com/device-management/managed-devices

(5) 不要使用文件扩展名

	http://api.example.com/device-management/managed-devices.xml  //DO NOT USE
	http://api.example.com/device-management/managed-devices

(6) 使用query语句过滤集合(排序/过滤/分页等)

	http://api.example.com/device-management/managed-devices 
	http://api.example.com/device-management/managed-devices/{device-id} 
	http://api.example.com/user-management/users/
	http://api.example.com/user-management/users/{id}

### 总结        

对于REST, 先搞懂其缩写代表的含义, 再结合后面RESTful API的设计规范去理解,就更容易理解什么叫REST了.

前面总结了REST概念:        
Resource通过URI的形式呈现，Resource通过二进制字节码和元数据描述自身状态，状态信息可以在系统各个节点之间传输和转化，系统中各个节点通过URI对Resource进行访问并执行CRUD（Create/Read/Update/Delete）操作。

对应到RESTful API:        
资源对应URI, 状态对应Http Status Code, CRUD操作对应POST/GET/PUT/DELETE等HTTP方法

