---
layout: post
comments: true
title: "Android学习笔记——okdownload"
description: "Android学习笔记——okdownload"
category: Android
tags: [Android]
---

源码：
[https://github.com/lingochamp/okdownload](https://github.com/lingochamp/okdownload)

wiki：
[https://github.com/lingochamp/okdownload/wiki](https://github.com/lingochamp/okdownload/wiki)

<!--more-->

整个代码风格和okhttp有点相似。

关于这个源码的分析，我结合之前写的一篇涉及“关于模块设计”主题的文章 [学习方法总结](http://mouxuejie.com/blog/2018-05-12/learning-summary/) 来分析。

### （1）明确实现的功能

参考1：okhttp对外接口调用：    

    OkHttpClient client = new OkHttpClient();

    Request request = new Request.Builder()
          .url(url)
          .build();

    Call call = client.newCall(request);
    call.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            ...
        }
        @Override
        public void onResponse(Call call, Response response) throws IOException {
            ...
        }
    });

参考2: volley对外接口调用：    

    RequestQueue queue = Volley.newRequestQueue(this);
    String url ="http://www.google.com";

    StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
            new Response.Listener<String>() {
        @Override
        public void onResponse(String response) {
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
        }
    });

    queue.add(stringRequest);

okdownload的wiki文档里，关于各种不同情况的下载，对外接口并不是统一的，我觉得OnDownload这个类应该是一个对外类。
        
    // 在源码基础上略有修改
    DownloadTask task = new DownloadTask.Builder(url, parentFile)
                .setFilename(filename)
                // the minimal interval millisecond for callback progress
                .setMinIntervalMillisCallbackProcess(16)
                // ignore the same task has already completed in the past.
                .setPassIfAlreadyCompleted(false)
                .build();
    // 异步
    OkDownload.with().enqueue(task, downloadlistener);
    // 同步
    OkDownload.with().execute(task, downloadlistener);    
    
### （2）分析实现这个功能需要哪些角色

OkDownload用到了Builder模式。涉及如下角色：

    // 下载分发器
    private DownloadDispatcher downloadDispatcher;
    // 回调分发器
    private CallbackDispatcher callbackDispatcher;
    // 下载存储类
    private DownloadStore downloadStore;
    // 下载连接
    private DownloadConnection.Factory connectionFactory;
    // 文件处理
    private ProcessFileStrategy processFileStrategy;
    // 下载策略
    private DownloadStrategy downloadStrategy;
    // 下载输出流
    private DownloadOutputStream.Factory outputStreamFactory;


下载分发器 DownloadDispatcher    

    private final List<DownloadCall> readyAsyncCalls;
    private final List<DownloadCall> runningAsyncCalls;
    private final List<DownloadCall> runningSyncCalls;
    private final List<DownloadCall> finishingCalls;

    private volatile ExecutorService executorService;
    private int maxParallelRunningCount = 5;
    
    public void enqueue(DownloadTask[] tasks);
    public void enqueue(DownloadTask task);
    public void execute(DownloadTask task);
    
    public boolean cancel(IdentifiedTask task);
    public void cancel(IdentifiedTask[] tasks);
    public boolean cancel(int id);
    public void cancelAll();

回调分发器 CallbackDispatcher

    private final DownloadListener transmit;
    private final Handler uiHandler;
    
    public DownloadListener dispatch();
    
    public class DefaultTransmitListener implements interface DownloadListener {
        void taskStart(DownloadTask task);
        void connectTrialStart(DownloadTask task, Map<String, List<String>> requestHeaderFields);
        void connectTrialEnd(DownloadTask task, int responseCode, Map<String, List<String>> responseHeaderFields);
        void downloadFromBeginning(DownloadTask task, BreakpointInfo info, ResumeFailedCause cause);
        void downloadFromBreakpoint(DownloadTask task, BreakpointInfo info);
        void connectStart(DownloadTask task, int blockIndex, Map<String, List<String>> requestHeaderFields);
        void connectEnd(DownloadTask task, int blockIndex, int responseCode, Map<String, List<String>> responseHeaderFields);
        void fetchStart(DownloadTask task, int blockIndex, long contentLength);
        void fetchProgress(DownloadTask task, int blockIndex, long increaseBytes);
        void fetchEnd(DownloadTask task, int blockIndex, long contentLength);
        void taskEnd(DownloadTask task, EndCause cause, Exception realCause);
    }


下载存储类 DownloadStore

    默认实现类是BreakpointStoreOnSQLite
    protected final BreakpointSQLiteHelper helper;
    protected final BreakpointStoreOnCache onCache;

    BreakpointInfo get(int id);
    BreakpointInfo createAndInsert(DownloadTask task);
    int findOrCreateId(DownloadTask task);
    boolean update(BreakpointInfo breakpointInfo);
    void remove(int id);
    
下载连接 DownloadConnection，使用简单工厂模式

    void addHeader(String name, String value);
    boolean setRequestMethod(@NonNull String method);
    Connected execute();
    void release();
    
    interface Factory {
        DownloadConnection create(String url) throws IOException;
    }

文件处理 ProcessFileStrategy

    public MultiPointOutputStream createProcessStream(@NonNull DownloadTask task,
                                                               @NonNull BreakpointInfo info,
                                                               @NonNull DownloadStore store) {
        return new MultiPointOutputStream(task, info, store);
    }

    public void completeProcessStream(@NonNull MultiPointOutputStream processOutputStream,
                                      @NonNull DownloadTask task) {
    }

    public void discardProcess(@NonNull DownloadTask task);
    
    而MultiPointOutputStream是支持分段处理的输出流，FileChannel支持seek
    public void write(int blockIndex, byte[] bytes, int length);
    public void cancel();
    public void done(int blockIndex);
    public void cancelAsync();

下载策略 DownloadStrategy

    // 1 connection: [0, 1MB)
    private static final long ONE_CONNECTION_UPPER_LIMIT = 1024 * 1024; // 1MiB
    // 2 connection: [1MB, 5MB)
    private static final long TWO_CONNECTION_UPPER_LIMIT = 5 * 1024 * 1024; // 5MiB
    // 3 connection: [5MB, 50MB)
    private static final long THREE_CONNECTION_UPPER_LIMIT = 50 * 1024 * 1024; // 50MiB
    // 4 connection: [50MB, 100MB)
    private static final long FOUR_CONNECTION_UPPER_LIMIT = 100 * 1024 * 1024; // 100MiB
    
    public int determineBlockCount(@NonNull DownloadTask task, long totalLength);
    public boolean isUseMultiBlock(final boolean isAcceptRange);
    
下载输出流 DownloadOutputStream，使用简单工厂模式

    void write(byte[] b, int off, int len);
    void close();
    void flushAndSync();
    void seek(long offset);
    void setLength(long newLength);
    interface Factory {
        DownloadOutputStream create(Context context, File file, int flushBufferSize);
        DownloadOutputStream create(Context context, Uri uri, int flushBufferSize);
        boolean supportSeek();
    }

    默认实现类DownloadUriOutputStream，成员变量
    private final FileChannel channel;
    private final ParcelFileDescriptor pdf;
    private final BufferedOutputStream out;
    private final FileOutputStream fos;

### （3）理清各角色之间的关系

![](/image/2018-09-20-okdownload/okdownload.svg)    

### （4）细化各角色的具体工作

DownloadConnection：使用的okhttp

DownloadStore：内存存储和db

支持分段下载，分段规则由DownloadStrategy决定

    public class BreakpointInfo {
        private final int id;
        private final String url;
        private String etag;
        private File targetFile;
        private final List<BlockInfo> blockInfoList;
    }
    
    public class BlockInfo {
        @IntRange(from = 0)
        private final long startOffset;
        @IntRange(from = 0)
        private final long contentLength;
        private final AtomicLong currentOffset;
    }
