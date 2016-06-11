---
layout: post
comments: true
title: "MultiDex安装过程源码分析"
description: "MultiDex安装过程源码分析"
category: Android
tags: [Android]
---

## 1.解决65535问题的通用方法

当项目比较大时，通常我们会遇到65535问题，相信大家都知道需要用到`MultiDex`分包。

通常做法是，在Application中添加如下代码

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
    
然后在build.gradle中添加

    multiDexEnabled true

以上做法确实可以解决许多问题，但是在实际项目中，还是会存在一些问题。

<!--more-->

关于MultiDex和热修复这块，计划是分3－4篇文章来写。因为我一下也吸收不了那么多，这一节主要是分析MultiDex源码。

关于为啥dex方法数限定65535?官方文档说的MultiDex的一些坑？以及更深层次的东西，打算自己研究相关dalvik源码后再写。

## 2.MultiDex源码分析

### 0x00 入口方法MultiDex.install(Context)

    public static void install(Context context) {
        Log.i("MultiDex", "install");
        if(IS_VM_MULTIDEX_CAPABLE) {
            Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
        } else if(VERSION.SDK_INT < 4) {
            throw new RuntimeException("Multi dex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
        } else {
            try {
                ApplicationInfo e = getApplicationInfo(context);
                if(e == null) {
                    return;
                }

                Set var2 = installedApk;
                synchronized(installedApk) {
                    String apkPath = e.sourceDir;
                    if(installedApk.contains(apkPath)) {
                        return;
                    }

                    installedApk.add(apkPath);
                    if(VERSION.SDK_INT > 20) {
                        Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT + ": SDK version higher than " + 20 + " should be backed by " + "runtime with built-in multidex capabilty but it\'s not the " + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
                    }

                    ClassLoader loader;
                    try {
                        loader = context.getClassLoader();
                    } catch (RuntimeException var9) {
                        Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var9);
                        return;
                    }

                    if(loader == null) {
                        Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
                        return;
                    }

                    try {
                        clearOldDexDir(context);
                    } catch (Throwable var8) {
                        Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var8);
                    }

                    File dexDir = new File(e.dataDir, SECONDARY_FOLDER_NAME);
                    List files = MultiDexExtractor.load(context, e, dexDir, false);
                    if(checkValidZipFiles(files)) {
                        installSecondaryDexes(loader, dexDir, files);
                    } else {
                        Log.w("MultiDex", "Files were not valid zip files.  Forcing a reload.");
                        files = MultiDexExtractor.load(context, e, dexDir, true);
                        if(!checkValidZipFiles(files)) {
                            throw new RuntimeException("Zip files were not valid.");
                        }

                        installSecondaryDexes(loader, dexDir, files);
                    }
                }
            } catch (Exception var11) {
                Log.e("MultiDex", "Multidex installation failure", var11);
                throw new RuntimeException("Multi dex installation failed (" + var11.getMessage() + ").");
            }

            Log.i("MultiDex", "install done");
        }
    }
    
依次做了如下事情：

1.先看JVM是否支持MultiDex，若JVM本身就支持则MultiDex库工程将被禁用

2.检查SDK版本号，要求最低版本为4

3.执行具体MultiDex操作：

（1）从ApplicationInfo.sourcecDir中获取APK路径apkPath，其值为/data/app/apkName.apk

（2）检查APK是否已安装，若APK已安装，则不进行后续操作。检查SDK版本号，版本号大于20不能保证MultiDex可正常Work

（3）清空旧的Dex目录。定位到旧Dex路径dexDir，即/data/data/pkgName/files/secondary-dexes，删除dexDir目录下所有文件。具体实现在clearOldDexDir(context)方法中。

（4）multi dex文件列表提取。从dex文件提取过程可阅读MultiDexExtractor.load(context, e, dexDir, false)源码

    static List<File> load(Context context, ApplicationInfo applicationInfo, File dexDir, boolean forceReload) throws IOException {
        Log.i("MultiDex", "MultiDexExtractor.load(" + applicationInfo.sourceDir + ", " + forceReload + ")");
        File sourceApk = new File(applicationInfo.sourceDir);
        long currentCrc = getZipCrc(sourceApk);
        List files;
        if(!forceReload && !isModified(context, sourceApk, currentCrc)) {
            try {
                files = loadExistingExtractions(context, sourceApk, dexDir);
            } catch (IOException var9) {
                Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var9);
                files = performExtractions(sourceApk, dexDir);
                putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
            }
        } else {
            Log.i("MultiDex", "Detected that extraction must be performed.");
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
        }

        Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
        return files;
    }

以上dexDir路径为/data/data/pkgName/code_cache/secondary-dexes。

先检查forceReload和sourceApk isModified。（forceReload由提取过程是否出错决定，isModified通过比较sourceApk文件的CRC校验码和时间戳是否变化决定）

若不强制重新提取且sourceApk未经修改。则调用loadExistingExtractions()，直接加载已经存在的从dex文件。

    private static List<File> loadExistingExtractions(Context context, File sourceApk, File dexDir) throws IOException {
        Log.i("MultiDex", "loading existing secondary dex files");
        String extractedFilePrefix = sourceApk.getName() + ".classes";
        int totalDexNumber = getMultiDexPreferences(context).getInt("dex.number", 1);
        ArrayList files = new ArrayList(totalDexNumber);

        for(int secondaryNumber = 2; secondaryNumber <= totalDexNumber; ++secondaryNumber) {
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";
            File extractedFile = new File(dexDir, fileName);
            if(!extractedFile.isFile()) {
                throw new IOException("Missing extracted secondary dex file \'" + extractedFile.getPath() + "\'");
            }

            files.add(extractedFile);
            if(!verifyZipFile(extractedFile)) {
                Log.i("MultiDex", "Invalid zip file: " + extractedFile);
                throw new IOException("Invalid ZIP file.");
            }
        }

        return files;
    }
    
以上代码段具体实现是，从dexDir路径下提取如下zip文件列表：

/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip

/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes3.zip

......

/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classesN.zip

若以上loadExistingExtractions过程中出现异常（缺少某个从dex，或zip文件格式不对），或者强制重新提取或sourceApk被修改，则调用performExtractions(File sourceApk, File dexDir)，重新抽取从dex列表

    private static List<File> performExtractions(File sourceApk, File dexDir) throws IOException {
        String extractedFilePrefix = sourceApk.getName() + ".classes";
        prepareDexDir(dexDir, extractedFilePrefix);
        ArrayList files = new ArrayList();
        ZipFile apk = new ZipFile(sourceApk);

        try {
            int e = 2;

            for(ZipEntry dexFile = apk.getEntry("classes" + e + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + e + ".dex")) {
                String fileName = extractedFilePrefix + e + ".zip";
                File extractedFile = new File(dexDir, fileName);
                files.add(extractedFile);
                Log.i("MultiDex", "Extraction is needed for file " + extractedFile);
                int numAttempts = 0;
                boolean isExtractionSuccessful = false;

                while(numAttempts < 3 && !isExtractionSuccessful) {
                    ++numAttempts;
                    extract(apk, dexFile, extractedFile, extractedFilePrefix);
                    isExtractionSuccessful = verifyZipFile(extractedFile);
                    Log.i("MultiDex", "Extraction " + (isExtractionSuccessful?"success":"failed") + " - length " + extractedFile.getAbsolutePath() + ": " + extractedFile.length());
                    if(!isExtractionSuccessful) {
                        extractedFile.delete();
                        if(extractedFile.exists()) {
                            Log.w("MultiDex", "Failed to delete corrupted secondary dex \'" + extractedFile.getPath() + "\'");
                        }
                    }
                }

                if(!isExtractionSuccessful) {
                    throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + e + ")");
                }

                ++e;
            }
        } finally {
            try {
                apk.close();
            } catch (IOException var16) {
                Log.w("MultiDex", "Failed to close resource", var16);
            }

        }

        return files;
    }

以上代码段具体实现是：

先准备好dexDir，prepareDexDir(File dexDir, final String extractedFilePrefix)。从dexDir路径下过滤掉所有不是以apkName.apk.classes开头的文件，并删除这些文件。

从dex文件列表提取。从sourceApk的文件中找到classes2.dex...classesN.dex的ZipEntry入口，依次调用extract(apk, dexFile, extractedFile, extractedFilePrefix)：

    private static void extract(ZipFile apk, ZipEntry dexFile, File extractTo, String extractedFilePrefix) throws IOException, FileNotFoundException {
        InputStream in = apk.getInputStream(dexFile);
        ZipOutputStream out = null;
        File tmp = File.createTempFile(extractedFilePrefix, ".zip", extractTo.getParentFile());
        Log.i("MultiDex", "Extracting " + tmp.getPath());

        try {
            out = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(tmp)));

            try {
                ZipEntry classesDex = new ZipEntry("classes.dex");
                classesDex.setTime(dexFile.getTime());
                out.putNextEntry(classesDex);
                byte[] buffer = new byte[16384];

                for(int length = in.read(buffer); length != -1; length = in.read(buffer)) {
                    out.write(buffer, 0, length);
                }

                out.closeEntry();
            } finally {
                out.close();
            }

            Log.i("MultiDex", "Renaming to " + extractTo.getPath());
            if(!tmp.renameTo(extractTo)) {
                throw new IOException("Failed to rename \"" + tmp.getAbsolutePath() + "\" to \"" + extractTo.getAbsolutePath() + "\"");
            }
        } finally {
            closeQuietly(in);
            tmp.delete();
        }

    }
    
分析下extract(apk, dexFile, extractedFile, extractedFilePrefix)：

参数如下：

- apk:  Apk文件/data/app/apkName.apk

- dexFile:  Apk文件zip解压后得到的从dex文件，classes2.dex...classesN.dex

- extractedFile:  dexFile写入的目标文件/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip等

- extractedFilePrefix:  前缀apkName.apk.classes

简言之，这个方法做的事情就是，将Apk文件解压后得到的classes2.dex, ..., classesN.dex文件的内容依次写入到/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip, ..., /data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classesN.zip压缩文件的classes.dex文件中。

若该方法执行异常，则最多可以重试3次。

最终performExtractions做的事情就是，将/data/app.apkName.apk文件解压得到的classes2.dex, ..., classesN.dex文件的内容依次写入/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip, ..., /data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classesN.zip压缩文件的classes.dex文件中，并返回这个zip列表。

得到从dex的zip列表后，SP中存入timestamp, crc, dex.number等Apk相关信息

得到从dex的zip列表后，调用installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files)进行安装

installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files)具体实现为：

根据SDK >= 19，SDK < 14， 14 <= SDK < 19三种情况V19, V14, V4分别调用不同实现。

以V19为例，安装过程实际做的事情是：

通过反射方法，找到BaseDexClassLoader实例中的DexPathList类型的pathList实例变量。

调用pathList实例的makeDexElements(ArrayList<File> files,File optimizedDirectory)方法，将上面得到的zip文件列表封装成Element数组

修改pathList实例的dexElements数组变量的值，在原有基础上扩充上面的Element数组。

异常处理

makeDexElements的C实现中，会执行里相关dex2opt优化操作，这个是个耗时操作，更详细的可以看dalvik源码部分。

## 小结

本文主要讲述MultiDex的安装过程：

将/data/app/apkName.apk路径下解压得到的classes2.dex, ..., classesN.dex，依次写入到/data/data/pkgName/code_cache/secondary-dexes/apkName.apk.classes2.zip等zip文件的classes.dex中，并返回这个zip列表。然后针对这个zip列表执行安装过程，具体过程是，将这个要安装的zip列表加入BaseDexClassLoader的pathList实例的dexElements数组中，其中会针对各dex文件进行dex2opt优化。一旦加入到了dexElements数组中，程序启动的时候，ClassLoader会加载dexElements数组中的元素，从而实现multi dex的安装。

这么看来，本文并没有讲述如果进行multi dex拆分。

未完待续。

