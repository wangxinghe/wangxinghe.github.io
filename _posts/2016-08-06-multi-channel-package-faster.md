---
layout: post
comments: true
title: "Android打包系列——多渠道快速打包"
description: "Android打包系列——多渠道快速打包"
category: Android
tags: [Android]
---

## 0x00前言

上一篇文章总结了下多渠道打包的知识点，又研究了下多渠道快速打包的几种方法。

虽说这些知识已经出来一段时间了，由于楼主也在成长积累阶段，所以还是觉得有总结下的必要。

能力有限，楼主暂时只能做一些知识点的总结，后续研究一些新的东西会及时分享出来。

<!--more-->

## 0x01提高多渠道打包速度

之所以打包的时候带入渠道号，主要还是为了统计各个渠道的相关数据。

普通的多渠道打包速度是非常慢的，可这根本难不倒网络上的极客们。

下面分别介绍下几种多渠道打包方式的特点。

### 1. 普通多渠道打包方式

普通的多渠道打包需要经历如下几个步骤：

- 解压apk文件
- 替换AndroidManifest.xml中的meta-data
- 压缩apk文件
- 签名

读取渠道号：直接通过Android的API读取meta-data

这种方式的特点是：生成每一个渠道包，都需要经过解压缩、压缩、签名这些步骤。而这些操作都非常耗时，因此会导致打包效率不高。据网上的经验是，打100个渠道包需要大约1个小时。

这种打包方式的具体操作，可以参考楼主的上一篇文章。

### 2. 美团多渠道打包

美团的多渠道打包需要经历如下几个步骤：

- 解压apk文件
- 在META-INF目录下创建一个名称为渠道号的空文件
- 压缩apk文件

读取渠道号：定位到安装包目录data/app/<package>.apk，解压apk文件，读取META-INF下的空文件的文件名

这种方式的特点是：生成一个渠道包，需要经过解压缩、创建空文件、压缩这些步骤。和普通多渠道打包相比，不需要重复签名，因此效率大大提高。一般来说这种方式就可以满足需求了。

楼主参考网上的例子，自己学着写了个多渠道打包的Python脚本。

    #!/usr/bin/python
    # -*- coding: utf-8 -*-
    import os
    import shutil
    import zipfile

    # 步骤:
    # 找到源apk文件
    # 遍历channel.txt中的所有渠道
    # 创建新的渠道包文件,并复制源文件内容到新创建的渠道包文件
    # 在新的渠道包文件中/META-INF/目录下创建标识渠道的空文件


    def get_original_apk(path):
        for f in os.listdir(path):
            if f.endswith(".apk"):
                return f


    def multi_channel_pkg():
        # 获取源apk,前缀及后缀
        original_apk = get_original_apk(".")

        if original_apk is None:
            return

        index = original_apk.find(".apk")
        original_apk_name = os.path.splitext(original_apk)[0:index]
        apk_name_prefix = original_apk_name[0]
        apk_name_suffix = original_apk_name[1]

        if os.path.exists("./output"):
            shutil.rmtree("./output")

        os.mkdir("./output")

        # 遍历渠道文件
        with open("./channel.txt") as f:
            for line in f:
                channel_name = line.strip("\n")

                # 拷贝源apk内容到渠道apk
                dest_apk = "./output/{}-{}{}".format(apk_name_prefix, channel_name, apk_name_suffix)
                shutil.copy(original_apk, dest_apk)

                # 创建空的渠道文件
                f_empty_channel = open(channel_name, 'w')
                f_empty_channel.close()

                # 往渠道apk中添加空的渠道文件
                dest_channel_path = "./META-INF/" + channel_name
                f = zipfile.ZipFile(dest_apk, 'a')
                f.write(channel_name, dest_channel_path)
                f.close()

                # 删除本地空的渠道文件
                os.remove(channel_name)


    multi_channel_pkg()

具体可以参考[https://github.com/wangxinghe/multi_channel_pack](https://github.com/wangxinghe/multi_channel_pack)

### 3. 一种更快速的打包

继美团多渠道打包方案之后，万能的网友又想出了一种更快速的打包方式。

说实话，这种打包方式其实也是很容易想到的，只不过大多数人知识面太窄，所以觉得高大上。

由于apk文件实质上就是个zip包，因此可以利用zip包的文件结构，将渠道信息带进去即可。

一个完整的zip包的文件结构如下图所示。

![image/2016-08-06-multi-channel-package-faster/zip_format.png](image/2016-08-06-multi-channel-package-faster/zip_format.png)

该结构的末尾部分，结构如下表所示。

![image/2016-08-06-multi-channel-package-faster/eocd.png](image/2016-08-06-multi-channel-package-faster/eocd.png)

很显然，我们能看到最后两个字段是comment length和comment。

正常情况下，comment部分为空，而comment length存储comment的字节长度，占2 bytes.

这种多渠道打包方式：只需要写入渠道号到apk文件末尾即可。

读取渠道号：直接读取data/app/<package>.apk文件末尾的渠道号

这种方式的特点：没有解压缩、压缩、重签名等步骤，比美团的打包效率还要高。

下面我列出自己的代码实现片段：

    public static void pack(String outputDir, File originalApkFile, File channelFile) {
        List<String> channels = getAllChannels(channelFile);
        for (String channel : channels) {
            packOneApk(outputDir, originalApkFile, channel);
        }
    }

    public static String readPackChannel(File channelApkFile) {
        String channel = "";
        try {
            RandomAccessFile raf = new RandomAccessFile(channelApkFile, "r");
            //read magic words
            long index = raf.length() - MAGIC.length;
            raf.seek(index);
            byte magic[] = new byte[MAGIC.length];
            raf.read(magic);
            if (!isEqual(magic, MAGIC)) {
                return channel;
            }

            //read channel length
            index -= 2;
            raf.seek(index);
            short channelLength = raf.readShort();
            if (channelLength <= 0) {
                return channel;
            }

            //read channel
            index -= channelLength;
            raf.seek(index);
            byte channelByte[] = new byte[channelLength];
            raf.read(channelByte);
            channel = new String(channelByte, Charset.forName("UTF-8"));
        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }

        return channel;
    }

    private static void packOneApk(String outputDir, File originalApkFile, String channel) {
        writeComment(outputDir, originalApkFile, channel);
    }

    private static void writeComment(String outputDir, File originalApkFile, String channel) {
        try {
            String originalApkFileName = originalApkFile.getName();
            int index = originalApkFileName.indexOf(".apk");
            if (index == -1) {
                return;
            }

            String apkPrefix = originalApkFileName.substring(0, index - 1);
            String destApkFileName = apkPrefix + "-" + channel + ".apk";
            File destApkFile = new File(outputDir, destApkFileName);
            boolean success = destApkFile.createNewFile();
            if (!success) {
                return;
            }

            copyFile(originalApkFile, destApkFile);
            //comment length is 2 bytes
            //comment format: channel + channel.length(2 bytes) + magic
            RandomAccessFile raf = new RandomAccessFile(destApkFileName, "rw");
            raf.seek(destApkFile.length() - 2);
            writeShort((short) (channel.getBytes("UTF-8").length + 2 + MAGIC.length), raf);
            raf.write(channel.getBytes("UTF-8"));
            writeShort((short)channel.getBytes("UTF-8").length, raf);
            raf.write(MAGIC);
            raf.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void copyFile(File originalApkFile, File destApkFile) {
        BufferedInputStream bis = null;
        BufferedOutputStream bos = null;
        try {
            bis = new BufferedInputStream(new FileInputStream(originalApkFile));
            bos = new BufferedOutputStream(new FileOutputStream(destApkFile));
            byte buffer[] = new byte[1024];
            int length;
            while ((length = bis.read(buffer)) != -1) {
                bos.write(buffer, 0, length);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (bos != null) {
                    bos.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                if (bis != null) {
                    bis.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

具体的可以看我的github。

这个打包过程已经做成了Android Studio的插件形式，只不过楼主还没研究过插件开发，有空打算研究下。

## 0x02总结

思路并不难，关键还是要不断学习，拓宽自己的知识面。

多研究下Apk包的文件结构。其实后面介绍的两种打包方式本质上都是从文件结构入手的。

另外还可以学习下Python、插件开发。

## 0x03参考文档

- [https://github.com/mcxiaoke/packer-ng-plugin
](https://github.com/mcxiaoke/packer-ng-plugin
)
- [https://github.com/seven456/MultiChannelPackageTool](https://github.com/seven456/MultiChannelPackageTool)
- [http://tech.meituan.com/mt-apk-packaging.html](http://tech.meituan.com/mt-apk-packaging.html)
- [https://github.com/GavinCT/AndroidMultiChannelBuildTool](https://github.com/GavinCT/AndroidMultiChannelBuildTool)
- [http://chuansong.me/n/2715890](http://chuansong.me/n/2715890)
- [https://en.wikipedia.org/wiki/Zip\_\(file_format\)](https://en.wikipedia.org/wiki/Zip\_\(file_format\))