---
layout: post
comments: true
title: "Android学习笔记——OkHttp"
description: "Android学习笔记——OkHttp"
category: Android
tags: [Android]
---

**1、OKHttp整体流程**    
**2、任务队列**    
**3、拦截器Interceptor**    
**（1）RetryAndFollowUpInterceptor**    
**（2）BridgeInterceptor**    
**（3）CacheInterceptor**  
**（4）ConnectInterceptor**      
**（5）CallServerInterceptor**    
**4、底层机制**    
**（1）通信机制**    
**（2）IO机制**    
**5、参考文档**

<!--more-->


### 1、OKHttp整体流程    

整体流程如图1所示：    

![](/image/2018-04-12-learning-notes-okhttp/okhttp_full_process.png)

从getResponseWithInterceptorChain()开始的拦截器部分如下图所示：

![](/image/2018-04-12-learning-notes-okhttp/okhttp_interceptor.png)

Inteceptor的设计和TCP/IP协议模型的分层思想非常类似。Netty模块的Handler的设计也是一样，RealInterceptorChain类似于Pipeline，Inteceptor类似于Handler。

OkHttp特点：    
（1）支持HTTP2/SPDY黑科技    
（2）socket自动选择最好路线，并支持自动重连    
（3）拥有自动维护的socket连接池，减少握手次数    
（4）拥有队列线程池，轻松写并发    
（5）拥有Interceptors轻松处理请求与响应（比如透明GZIP压缩,LOGGING）    
（6）基于Headers的缓存策略    

### 2、任务队列    

任务分为`同步任务`和`异步任务`。    
同步任务调用RealCall.execute()，异步任务调用RealCall.enqueue()，RealCall里面封装了originalRequest对象。


    public final class Dispatcher {
      //异步任务最大请求数
      private int maxRequests = 64;
      //异步任务同一个host的最大请求数
      private int maxRequestsPerHost = 5;
      //执行异步任务的线程池
      private @Nullable ExecutorService executorService;
      //异步任务如果超出maxRequests和maxRequestsPerHost，后面的任务加入的双端队列
      private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
      //运行中的异步任务双端队列    
      private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

      //运行中的同步任务双端队列
      private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
    ｝

### 3、拦截器Interceptor    

#### （1）RetryAndFollowUpInterceptor    

RetryAndFollowUpInterceptor这个过滤器的主要作用就是用于对请求的重试和重定向的。    
最大重定向次数MAX_FOLLOW_UPS是20，

其中拒绝重试的判断条件有如下几种：    
（1）如果我们在配置OkHttpClient中配置retryOnConnectionFailure属性为false，表明拒绝失败重连    
（2）如果请求已经发送，并且这个请求体是一个UnrepeatableRequestBody类型，则不能重试    
（3）如果是一些严重的协议，安全方面的异常（如ProtocolException、SSLHandshakeException），拒绝重试    
（4）没有更多的可以使用的路由，则不要重试了    

#### （2）BridgeInterceptor    

BridgeInterceptor的作用：    
（1）请求层面：将外部封装的简单的Request设置为可进行网络请求的Request。头部扩充Content-Type／Content-Length／Transfer-Encoding／Host／Connection(Keep-Alive)／Accept-Encoding／Cookie／User-Agent等键值对。    
（2）响应层面：将网络请求得到的Response扩展为上层需要的Response，如Content-Encoding为gzip的处理。

#### （3）CacheInterceptor  

CacheInterceptor的作用：    
（1）请求层面：决定使用本地缓存还是网络请求    
（2）响应层面：将响应结果写入磁盘缓存        

磁盘缓存仅支持GET方法。

**1.关于是否使用本地缓存还是网络请求的缓存策略：**    

![](/image/2018-04-12-learning-notes-okhttp/CacheStrategy.svg)

**2.磁盘缓存逻辑：**    
使用的是DiskLruCache，这一套缓存逻辑和UniversalImageLoader中的磁盘缓存代码逻辑一模一样，只不过IO部分这边由InputStream/OutputStream替换成Okio中的Source/Sink。

![](/image/2018-04-12-learning-notes-okhttp/Cache.svg)

这里主要包括3个部分：    
`LinkedHashMap<String, Entry> lruEntries`是内存索引，key为md5(url)，value为Entry，这个Entry可以理解为每条响应对应存在磁盘中的记录，对应4个文件dirtyFiles和cleanFiles，其中dirtyFiles和cleanFiles分别包含响应头部和响应body。    
    
`journal`可以理解为每次操作的记录，类似于Git log一样。这部分有3个文件，journal、journal.tmp、journal.bkp。其中journal文件是用来记录cache的操作；journal.tmp是一个临时文件，当journal在rebuild时临时起作用的文件；journal.bkp是一个备份文件。通常情况下就只有journal文件。    

`dirtyFiles和cleanFiles`写入真正的响应数据，其中dirtyFiles相当于临时文件，写入过程中是先写入dirtyFiles，当响应的header和body都写进dirtyFiles后，会删除cleanFiles，同时dirtyFiles重命名为cleanFiles。和journal文件操作有点类似。

journal和cleanFiles/dirtyFiles的文件IO操作，都是用的OKio框架。

journal文件内容如下：    

     libcore.io.DiskLruCache
     1
     100
     2
     
     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
     DIRTY 335c4c6028171cfddfbaae1a9c313c52
     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
     REMOVE 335c4c6028171cfddfbaae1a9c313c52
     DIRTY 1ab96a171faeeee38496d8b330771a7a
     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
     READ 335c4c6028171cfddfbaae1a9c313c52
     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6


前5行是journal头部信息：    
第一行是`MAGIC`，值为“libcore.io.DiskLruCache”    
第二行是DiskLruCache的版本号`VERSION_1`为1     
第三行是应用程序的版本号`appVersion`，外部传入    
第四行是`valueCount`，外部传入，用于CLEAN情况且值为2，分别代表的是响应header和body的字节数大小    
第五行是一个空行    

后面是操作记录：    
`DIRTY`表示我们正准备写入一条缓存数据，但不知道结果如何。每次调用DiskLruCache的edit()方法时写入，表示编辑中，对应的Entry会有一个Editor。    
`CLEAN`表示写入缓存成功，调用Editor.commit()的时候写入。    
`REMOVE`表示写入缓存失败，调用Editor.commit()的时候写入。    
`READ`每次调用DiskLruCache的get(key)方法时，就会写入一条记录
每一行第2个参数表示key(即md5(url))，第3～4个参数表示每条响应数据对应的文件中header和body字节数大小，header和body分别写到cleanFiles[0]和cleanFiles[1]文件里。    
由于缓存数据要么写入成功要么失败，因此DIRTY后面要么是CLEAN，要么是REMOVE

关于journal文件的大小限制：    
redundantOpCount可以大约理解成用户操作的记录数。当达到2000时，需要进行清理工作。    

journal清理工作会在线程池中发起一个cleanupRunnable：    
先遍历`旧的journal`文件，对于REMOVE的记录，删除lruEntries中相同key对应的Entry；    
同时将lruEntries中超出maxSize的部分Entry清除；    
然后遍历lruEntries中的Entry，对于每条Entry，如果editor不为空，则往`journal.tmp`写入一条DIRTY记录，否则写入CLEAN记录。    
删除老的journal.bkp文件和老的journal文件，journal.tmp重命名为`新的journal文件`    


磁盘写入和读取对应的核心代码如下：

    //Cache
    public void put(Response response) {
        Entry entry = new Entry(response);
        DiskLruCache.Editor editor = null;
        try {
            editor = cache.edit(key(response.request().url()));
            entry.writeTo(editor);
            editor.commit();
        } catch (IOException e) {
            editor.abort();
        }
    }

    //Cache
    public Response get(Request request) {
        String key = key(request.url());
        DiskLruCache.Snapshot snapshot = cache.get(key);
        Entry entry = new Entry(snapshot.getSource(ENTRY_METADATA));
        Response response = entry.response(snapshot);

        return response;
    }

下面是两篇写的比较好的文章：    
[Android DiskLruCache完全解析，硬盘缓存的最佳方案](https://blog.csdn.net/guolin_blog/article/details/28863651)    
[OkHttp3源码分析[缓存策略]](https://www.jianshu.com/p/9cebbbd0eeab)

#### （4）ConnectInterceptor      

ConnectInterceptor的作用：    
获取连接RealConnection和HttpCodec。    

实现思路：    
（1）先从连接池ConnectionPool中获取Address匹配的可用的有效连接，如果没有则new一个新连接RealConnection。    
（2）对于新创建的RealConnection，根据外部传入的参数决定调用connectTls或connectSocket去建立连接并确定协议。    
（3）对于HTTP1.1协议创建socket／sink／source等相关对象；对于HTTP2/HTTPS需要依次进行new SSLSocket、startHandshake、Verify Certificates并创建sink/source对象等操作。        
（4）将新建立的连接添加到连接池ConnectionPool    

连接池的实现：    

    public final class ConnectionPool {
        //用于垃圾回收的线程池
        private static final Executor executor = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
        //最大空闲连接数，默认为5
        private final int maxIdleConnections;
        //keep-alive持续时间（单位：纳秒），默认为5分钟
        private final long keepAliveDurationNs;
        //连接列表，是一个双端队列
        private final Deque<RealConnection> connections = new ArrayDeque<>();
        final RouteDatabase routeDatabase = new RouteDatabase();
    }

更详细的可以参考：[OkHttp3连接建立过程分析](https://www.wolfcstech.com/2016/10/27/OkHttp3%E8%BF%9E%E6%8E%A5%E5%BB%BA%E7%AB%8B%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/)

#### （5）CallServerInterceptor    

CallServerInterceptor的作用：    
通过HttpCodec执行真正的网络请求，并得到响应结果。    

基本代码：    

    @Override public Response intercept(Chain chain) throws IOException {
        //写入请求header
        httpCodec.writeRequestHeaders(request);
        //写入请求body
        if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            request.body().writeTo(Okio.buffer(new CountingSink(httpCodec.createRequestBody(request, contentLength))));
        }
        //执行请求
        httpCodec.finishRequest();
        //读取响应header
        Response.Builder responseBuilder = httpCodec.readResponseHeaders(false);
        //读取响应body（包了一个Source）
        Response response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
          
        return response;
    }
    
### 4、底层网络通信机制    
#### （1）底层网络通信机制    

完全摒弃了Http，最底层都是使用Socket。如果是HTTP1.1协议则使用普通Socket，如果是HTTPS／HTTP2则使用加密SSLSocket。使用SSLSocket建立连接时，需要先startHandshake，然后验证证书等。    
引进了连接池的概念，相同Address的请求可以复用同一个连接。    

#### （2）IO机制    

Socket流的读写使用的是OKio。

### 5、参考文档

（1）[拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/)    
（2）[OkHttp3源码分析[综述]](https://www.jianshu.com/p/aad5aacd79bf)    
