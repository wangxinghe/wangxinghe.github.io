---
layout: post
comments: true
title: "Android学习笔记——图片加载"
description: "Android学习笔记——图片加载"
category: Android
tags: [Android]
---

**1、ImageLoader框架**    
**（1）整体流程**    
**（2）数据存取**    
**（3）ImageDecoder**    
**2、各框架比较**    
**3、参考文档**    

<!--more-->

### 1、ImageLoader框架    

#### （1）整体流程    

![](/image/2018-05-17-learning-notes-image-loader/UIL_Flow.png)

**调用方式：**

    ImageLoader imageLoader = ImageLoader.getInstance();
    imageLoader.displayImage(imageUri, imageView);        
    
**缓存机制：**    
主要包括三级缓存：`内存缓存`，`磁盘缓存`，`网络`。    

存数据方向：`网络 -> 磁盘 -> 内存`。    
取数据方向：`内存 -> 磁盘 -> 网络`。    

**数据加载方式：**    

- 先判断内存是否有数据，如果有则依次走post-process -> Bitmap Displayer    
- 如果内存没有数据再判断磁盘是否有数据，如果有则走Image Decoder -> pre-process -> Memory Cache -> post-process -> Bitmap Displayer     
- 如果内存和磁盘都没有数据，则从网络获取，依次走Image Downloader -> Disk Cache -> Image Decoder -> pre-process -> Memory Cache -> post-process -> Bitmap Displayer    

在Bitmap存入Memory Cache和真正显示之前，都会经过Bitmap Processor进行处理（如果外部设置了的话）    

接下来还有几件事情需要搞清楚：    

- Bitmap怎么存到磁盘和内存，以及分别怎么从磁盘和内存取？        
- 存到磁盘和内存前是存原始数据吗？存之前做了哪些优化处理？    

#### （2）数据存取    

存磁盘`DiskCache`：    
**怎么存？**    
图片存储方式：`文件存储`，存储路径：cacheDir + fileName。    
文件名生成方式：imageUri做`HashCode`和`MD5`，分别对应HashCodeFileNameGenerator和MD5FileNameGenerator    
**优化？**    
先decode处理，再存磁盘。


存内存`MemoryCache`：    
图片存储方式：`Map<String, WeakReference<Bitmap>>`    
key生成方式：`[imageUri]_[width]x[height]`，width和height对应ImageView的宽高    
**优化？**    
先decode处理，再存内存。


imageUri为调用ImageLoader时最外层传入的uri

**下图为缓存类结构图：**    

![](/image/2018-05-17-learning-notes-image-loader/MemoryCache-DiskCache.svg)    

默认使用的是LruMemoryCache和UnlimitedDiskCache。

PSSSS，接下来还有几方面需要整理：    
（1）上图几种缓存策略的实现细节        
（2）cacheDir    
（3）ImageDecoder实现细节    


#### （3）ImageDecoder    

decode操作过程，实际上也就是使用BitmapFactory先采样(设置inSampleSize)decode，再Matrix处理。

inSampleSize的计算过程。

		Options options = new Options();
		options.inJustDecodeBounds = true;
        decodingOptions.inSampleSize = scale;
        Bitmap decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
        
        Matrix处理
        



下面是ImageDecodingInfo类的结构：

    public class ImageDecodingInfo {
        //memoryCacheKey
	    private final String imageKey;
        //图片在磁盘中的uri绝对路径，如file://path
    	private final String imageUri;
    	//ImageLoader加载图片时的原始uri，可能是http://，file://，assets://等
	    private final String originalImageUri;
    	private final ImageSize targetSize;

	    private final ImageScaleType imageScaleType;
    	private final ViewScaleType viewScaleType;

	    private final ImageDownloader downloader;
    	private final Object extraForDownloader;

	    private final boolean considerExifParams;
    	private final Options decodingOptions;
    ｝

### 2、各框架比较    

![](/image/2018-05-17-learning-notes-image-loader/image_loader_compare.png)    

### 3、参考文档    

（1）[3分钟全面了解Android主流图片加载库](https://blog.csdn.net/carson_ho/article/details/51939774)    
（2）[Android-Universal-Image-Loader（图片异步加载缓存库）的源码解读](https://blog.csdn.net/u011733020/article/details/51043810)

