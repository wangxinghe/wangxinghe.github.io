---
layout: post
comments: true
title: "MainDexList产生过程源码分析"
description: "MainDexList产生过程源码分析"
category: Android
tags: [Android]
---

## 0x00 前言

之前的两篇文章分别分析了 [MultiDex的编译及Dex过程](http://mp.weixin.qq.com/s?__biz=MzA5MTE1NTE5NQ==&mid=2653352671&idx=1&sn=8a79b8170e1554dab6d8c59466c0c6bd#rd) 和 [MultiDex的安装过程](http://mp.weixin.qq.com/s?__biz=MzA5MTE1NTE5NQ==&mid=2653352668&idx=1&sn=faeefc89e5e1f0d0bb3c7828de32d7aa#rd)

一个讲解了MultiDex的编译及Dex分包过程，我们知道了主dex是由maindexlist.txt决定的。

一个讲解了MultiDex中多个dex是怎么安装到程序中，让程序得以正常运行的。

本文重点讲解maindexlist.txt是怎么产生的。

<!--more-->

## 0x01 源码分析

对于一个使用了MultiDex的Android工程，编译后在/build/intermediates/multi-dex/{variant_path}/路径下面，可以看到如下几个文件。

- componentClasses.jar
- components.flags
- manifest_keep.txt
- maindexlist.txt

这几个文件决定了主Dex中的该放哪些类。

下面依次对这几个文件的产生过程进行分析。

### 1.CreateMainDexList

这小节讲的是maindexlist.txt的产生逻辑。

调用顺序为：

（1）/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/tasks/multidex/CreateMainDexList.groovy -> output()

（2）/build-system/builder/src/main/java/com/android/builder/core/AndroidBuilder.java -> createMainDexList(allClassesJarFile, jarOfRoots)

（3）/dalvik/dx/src/com/android/multidex/ClassReferenceListBuilder.java -> addRoots(ZipFile jarOfRoots)

源码如下：

    //build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/tasks/multidex/CreateMainDexList.groovy
    @Override
    void execute(CreateMainDexList createMainDexList) {
        createMainDexList.androidBuilder = scope.globalScope.androidBuilder
        createMainDexList.setVariantName(scope.getVariantConfiguration().getFullName())

        def files = inputFiles
        createMainDexList.allClassesJarFile = files.call().first()
        ConventionMappingHelper.map(createMainDexList, "componentsJarFile") {
            scope.getProguardComponentsJarFile()
        }
        createMainDexList.mainDexListFile = scope.manifestKeepListFile
        createMainDexList.outputFile = scope.getMainDexListFile()
    }

    @TaskAction
    void output() {
        if (getAllClassesJarFile() == null) {
            throw new NullPointerException("No input file")
        }

        // manifest components plus immediate dependencies must be in the main dex.
        File _allClassesJarFile = getAllClassesJarFile()
        Set<String> mainDexClasses = callDx(_allClassesJarFile, getComponentsJarFile())

        // add additional classes specified via a jar file.
        File _includeInMainDexJarFile = getIncludeInMainDexJarFile()
        if (_includeInMainDexJarFile != null) {
            mainDexClasses.addAll(callDx(_allClassesJarFile, _includeInMainDexJarFile))
        }

        if (mainDexListFile != null) {
            Set<String> mainDexList = new HashSet<String>(Files.readLines(mainDexListFile, Charsets.UTF_8))
            mainDexClasses.addAll(mainDexList)
        }

        String fileContent = Joiner.on(System.getProperty("line.separator")).join(mainDexClasses)

        Files.write(fileContent, getOutputFile(), Charsets.UTF_8)
    }

CreateMainDexList中涉及到如下几个成员变量：

- @InputFile allClassesJarFile: /build/intermediates/multi-dex/{variant_path}/allclasses.jar
- @InputFile componentsJarFile: /build/intermediates/multi-dex/{variant_path}/componentClasses.jar
- @OutputFile outputFile: /build/intermediates/multi-dex/{variant_path}/maindexlist.txt
- @InputFile includeInMainDexJarFile: 额外指定的Jar包
- @InputFile mainDexListFile: /build/intermediates/multi-dex/{variant_path}/manifest_keep.txt
    
这些参数来源于：

/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/scope/VariantScope.java
    
通过分析componentClasses.jar和额外指定的Jar包includeInMainDexJarFile中的Class，从allClassesJarFile中找出前面Class直接或间接引用到的Class，然后加上manifest_keep.txt中keep住的类一起作为maindexlist.txt的内容。将maindexlist.txt里的Class进行Dex操作就产生了classes.dex。

参考网上一些文章以及变量命名，allClassesJarFile代表的应该是包含了所有class的一个Jar包，路径应该是/build/intermediates/multi-dex/{variant_path}/allclasses.jar，然而我在项目下并没发现该文件。

类引用关系的实现，可参考：
/dalvik/dx/src/com/android/multidex/ClassReferenceListBuilder.java中的
addRoots(ZipFile jarOfRoots)

关于componentClasses.jar，我理解的是，产生maindexlist.txt的根jar包（jarOfRoots），就像GC操作的Root一样，关于其产生过程，文章后面进行了分析。

更上一层的调用入口是

/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/scope/TaskManager.java -> createPostCompilationTasks()

### 2.CreateManifestKeepList

这小节讲的是manifest_keep.txt的产生逻辑。

调用顺序为：

/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/tasks/multidex/CreateManifestKeepList.groovy -> execute() -> generateKeepListFromManifest()

源码如下：

    @Override
    void execute(CreateManifestKeepList manifestKeepListTask) {
            manifestKeepListTask.setVariantName(scope.getVariantConfiguration().getFullName())

            // since all the output have the same manifest, besides the versionCode,
            // we can take any of the output and use that.
            final BaseVariantOutputData output = scope.variantData.outputs.get(0)
            ConventionMappingHelper.map(manifestKeepListTask, "manifest") {
                output.getScope().getManifestOutputFile()
            }

            manifestKeepListTask.proguardFile = scope.variantConfiguration.getMultiDexKeepProguard()
            manifestKeepListTask.outputFile = scope.getManifestKeepListFile();

            //variant.ext.collectMultiDexComponents = manifestKeepListTask
        }
        
    @TaskAction
    void generateKeepListFromManifest() {
        SAXParser parser = SAXParserFactory.newInstance().newSAXParser()

        Writer out = new BufferedWriter(new FileWriter(getOutputFile()))
        try {
            parser.parse(getManifest(), new ManifestHandler(out))

            // add a couple of rules that cannot be easily parsed from the manifest.
            out.write(
                """-keep public class * extends android.app.backup.BackupAgent {
                    <init>();
                }
                -keep public class * extends java.lang.annotation.Annotation {
                    *;
                }
            """)

            if (proguardFile != null) {
                out.write(Files.toString(proguardFile, Charsets.UTF_8))
            }
        } finally {
            out.close()
        }
    }

    private static String DEFAULT_KEEP_SPEC = "{ <init>(); }"
    private static Map<String, String> KEEP_SPECS = [
        'application'       : """{
                            <init>();
            void attachBaseContext(android.content.Context);
        }""",
        'activity'          : DEFAULT_KEEP_SPEC,
        'service'           : DEFAULT_KEEP_SPEC,
        'receiver'          : DEFAULT_KEEP_SPEC,
        'provider'          : DEFAULT_KEEP_SPEC,
        'instrumentation'   : DEFAULT_KEEP_SPEC,
    ]

    private class ManifestHandler extends DefaultHandler {
        private Writer out

        ManifestHandler(Writer out) {
            this.out = out
        }

        @Override
        void startElement(String uri, String localName, String qName, Attributes attr) {
            String keepSpec = (String)CreateManifestKeepList.KEEP_SPECS[qName]
            if (keepSpec) {

                boolean keepIt = true
                if (CreateManifestKeepList.this.filter) {
                    // for ease of use, turn 'attr' into a simple map
                    Map<String, String> attrMap = [:]
                    for (int i = 0; i < attr.getLength(); i++) {
                        attrMap[attr.getQName(i)] = attr.getValue(i)
                    }
                    keepIt = CreateManifestKeepList.this.filter(qName, attrMap)
                }

                if (keepIt) {
                    out.write((String)"-keep class ${attr.getValue('android:name')} $keepSpec\n")
                }
            }
        }
    }
        
CreateMainDexList中涉及到如下几个成员变量：

- @InputFile manifest: /build/intermediates/manifests/full/{variant_path}/AndroidManifest.xml
- @OutputFile outputFile: /build/intermediates/multi-dex/{variant_path}/manifest_keep.txt
- @InputFile proguardFile

生成manifest_keep.txt的过程是：

遍历并解析/build/intermediates/manifests/full/{variant_path}/AndroidManifest.xml文件，将其中application, activity, service, receiver, provider和instrumentation等标签对应的组件类添加到manifest_keep.txt文件中。

具体解析规则如下：

对于application，keepSpec为

    {
        <init>();
        void attachBaseContext(android.content.Context);
    }
    
对于activity, service, receiver, provider和instrumentation，keepSpec为

    { <init>(); }
    
遍历/build/intermediates/manifests/full/{variant_path}/AndroidManifest.xml中注册的组件，针对每个组件，解析其对应的attr属性，将属性列表转换成Map形式。然后根据`-keep class ${attr.getValue('android:name')} $keepSpec\n`格式生成对应的keep语句，以activity为例，其keep语句为`-keep class ***package.***Activity.class { <init>(); }\n`，依次将keep语句写到manifest_keep.txt中。

此外还需增加下面语句：

    -keep public class * extends android.app.backup.BackupAgent {
        <init>();
    }
    -keep public class * extends java.lang.annotation.Annotation {
        *;
    }

因此manifest_keep.txt中的内容为AndroidManifest.xml中注册的Application、四大组件、Instrumentation、BackupAgent和Annotation对应的类。有心的同学可以去Android项目验证下是否确实如此。manifest_keep.txt文件中的类是不能混淆的。

### 3.ProGuard混淆和components.flags

这小节讲的是Proguard混淆和components.flags的产生逻辑。

调用顺序为：

（1）/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/TaskManager.java -> createPostCompilationTasks(TaskFactory, VariantScope)

（2）/build-system/gradle-core/src/main/groovy/com/android/build/gradle/tasks/factory/ProGuardTaskConfigAction.java -> execute()

（3）/external/proguard/src/proguard/gradle/ProGuardTask.java -> proguard() -> getConfiguration()

（4）/external/proguard/src/proguard/ProGuard.java

（5）/external/proguard/src/proguard/Configuration.java; /external/proguard/src/proguard/ConfigurationParser.java

源码如下：

    //build-system/gradle-core/src/main/groovy/com/android/build/gradle/tasks/factory/ProGuardTaskConfigAction.java
    @Override
    public void execute(ProGuardTask proguardComponentsTask) {
        proguardComponentsTask.dontobfuscate();
        proguardComponentsTask.dontoptimize();
        proguardComponentsTask.dontpreverify();
        proguardComponentsTask.dontwarn();
        proguardComponentsTask.forceprocessing();

        try {
            proguardComponentsTask.configuration(scope.getManifestKeepListFile());

            proguardComponentsTask.libraryjars(new Callable<File>() {
                @Override
                public File call() throws Exception {
                    Preconditions.checkNotNull(
                            scope.getGlobalScope().getAndroidBuilder().getTargetInfo());
                    File shrinkedAndroid = new File(
                            scope.getGlobalScope().getAndroidBuilder().getTargetInfo()
                                    .getBuildTools()
                                    .getLocation(),
                            "lib" + File.separatorChar + "shrinkedAndroid.jar");

                    // TODO remove in 1.0
                    // STOPSHIP
                    if (!shrinkedAndroid.isFile()) {
                        shrinkedAndroid = new File(
                                scope.getGlobalScope().getAndroidBuilder().getTargetInfo()
                                        .getBuildTools().getLocation(),
                                "multidex" + File.separatorChar + "shrinkedAndroid.jar");
                    }

                    return shrinkedAndroid;
                }
            });

            proguardComponentsTask.injars(new Callable<File>() {
                @Override
                public File call() throws Exception {
                    return inputFiles.call().iterator().next();
                }
            });

            proguardComponentsTask.outjars(scope.getProguardComponentsJarFile());

            proguardComponentsTask.printconfiguration(
                    scope.getGlobalScope().getBuildDir() + "/" + FD_INTERMEDIATES
                            + "/multi-dex/" + scope.getVariantConfiguration().getDirName()
                            + "/components.flags");
        } catch (ParseException e) {
            throw new RuntimeException(e);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    //external/proguard/src/proguard/gradle/ProGuardTask.java
    @TaskAction
    public void proguard()
    throws ParseException, IOException
    {
        // Let the logging manager capture the standard output and errors from
        // ProGuard.
        LoggingManager loggingManager = getLogging();
        loggingManager.captureStandardOutput(LogLevel.INFO);
        loggingManager.captureStandardError(LogLevel.WARN);

        // Run ProGuard with the collected configuration.
        new ProGuard(getConfiguration()).execute();

    }

针对混淆ProGuard，先进行如下配置：

- 添加dontobfuscate
- 添加dontoptimize
- 添加dontpreverify
- 添加dontwarn
- 添加forceprocessing
- 添加配置文件manifest_keep.txt
- 添加libraryjars: 如/Users/wangxinghe/Library/Android/sdk/build-tools/23.0.1/lib/shrinkedAndroid.jar
- 添加injars: /build/intermediates/transforms/jarMerging/{variant_path}/jars/1/1f/combined.jar
- 添加outjars: /build/intermediates/multi-dex/{variant_path}/componentClasses.jar
- 添加配置输出文件printConfiguration: /build/intermediates/multi-dex/{variant_path}/components.flags

然后调用/external/proguard/src/proguard/gradle/ProGuardTask.java－>getConfiguration()方法，通过/external/proguard/src/proguard/ConfigurationParser.java解析以上配置，得到/external/proguard/src/proguard/Configuration.java实例。

然后调用/external/proguard/src/proguard/ProGuard.java->execute()方法，根据上面得到的Configuration实例配置，进行具体的ProGuard操作，同时将相关配置写到/build/intermediates/multi-dex/{variant_path}/components.flags文件中。

关于componentClasses.jar的生成，根据components.flags文件的描述，我个人理解的是，是由manifest_keep.txt文件和通过JarMergingTask任务生成的combined.jar文件产生的。至于combined.jar怎么产生的，根据前两篇文章关于jar的ZipEntry源码分析经验，将combined.jar扩展名改为.zip并解压后，发现是一个所有类及R合并后的class集合。

## 0x02 总结

理解有偏差的地方，多多指教～

## 欢迎大家关注我的公众号：学姐的IT专栏

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)

