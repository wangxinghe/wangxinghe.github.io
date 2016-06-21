---
layout: post
comments: true
title: "Android编译及Dex过程源码分析"
description: "Android编译及Dex过程源码分析"
category: Android
tags: [Android]
---

## 0x00 概述

上篇文章讲的是dex的安装过程。

本文主要讲Android Build System的编译和dex过程。

<!--more-->

## 0x01 从BasePlugin入口说起

#### 1.1 BasePlugin入口

Android Studio项目是基于Gradle构建的，module的build.gradle文件首部都会声明apply plugin: 'com.android.application'或apply plugin: 'com.android.library'，对应到源码是`/build-system/gradle/src/main/groovy/com/android/build/gradle/BasePlugin.java`的apply(Project)。而Gradle构建过程都是由一系列task组成。

整个调用流程为：

    BasePlugin.apply(Project) -> createTasks() -> createAndroidTasks(boolean force) -> VariantManager.createAndroidTasks()

#### 1.2 Android Tasks的创建

我们再来看看Android Tasks的创建：

VariantManager是Android Tasks创建的入口类。

这块的调用流程为：

（1）`/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/VariantManager.java -> createAndroidTasks`

（2）`createTasksForVariantData(tasks, variantData)`

（3）`/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/ApplicationTaskManager.java -> createTasksForVariantData(tasks, variantData)`

接下来我们看`/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/ApplicationTaskManager.java`这个类的实现。

从createTasksForVariantData()方法中，我们可以看到依次创建了如下task：

- sourceGenTask，resGenTask，assetGenTask
- checkManifestTask
- process the manifest(s) task
- create the res values task
- compile renderscript files task
- merge the resource folders task
- merge the asset folders task
- create the BuildConfig class task
- process the Android Resources and generate source files task
- process the java resources task
- process the aidl task
- compile task
- NDK tasks
- final packaging task，zipalign task
- lint tasks

以上基本包括了，构建一个可运行的完整APK的所有task。

      ///build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/ApplicationTaskManager.java
      public void createTasksForVariantData(
            @NonNull final TaskFactory tasks,
            @NonNull final BaseVariantData<? extends BaseVariantOutputData> variantData) {
        assert variantData instanceof ApplicationVariantData;
        final ApplicationVariantData appVariantData = (ApplicationVariantData) variantData;

        final VariantScope variantScope = variantData.getScope();

        //create sourceGenTask, resGenTask, assetGenTask and compileTask
        createAnchorTasks(tasks, variantScope);
        //create checkManifestTask
        createCheckManifestTask(tasks, variantScope);

        // Add a task to process the manifest(s)
        // Add a task to create the res values
        // Add a task to compile renderscript files.
        // Add a task to merge the resource folders
        // Add a task to merge the asset folders
        // Add a task to create the BuildConfig class
        // Add a task to process the Android Resources and generate source files
        // Add a task to process the java resources
        // Add a task to process the aidl
        ......

        // Add a compile task
        ThreadRecorder.get().record(ExecutionType.APP_TASK_MANAGER_CREATE_COMPILE_TASK,
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() {
                        AndroidTask<JavaCompile> javacTask = createJavacTask(tasks, variantScope);

                        if (variantData.getVariantConfiguration().getUseJack()) {
                            createJackTask(tasks, variantScope);
                        } else {
                            setJavaCompilerTask(javacTask, tasks, variantScope);
                            createJarTask(tasks, variantScope);
                            createPostCompilationTasks(tasks, variantScope);
                        }
                        return null;
                    }
                });

        // Add NDK tasks
        // Creates the final packaging task, and optionally the zipalign task
        // create the lint tasks.
        ......
    }

#### 1.3 Compile Task And Post Compile Task

我们看下compile task.

从上面代码中可以看出，我们先创建了javac task，这个task是将.java文件编译成.class文件。然后创建jar task和postCompile task，对于Jar task，这个用来将.class打成.jar包，而createPostCompilationTasks()主要是创建编译后的task。

    public void createPostCompilationTasks(TaskFactory tasks, @NonNull final VariantScope scope) {
        checkNotNull(scope.getJavacTask());
        boolean isMinifyEnabled = config.isMinifyEnabled();
        boolean isMultiDexEnabled = config.isMultiDexEnabled() && !isTestForApp;
        boolean isLegacyMultiDexMode = config.isLegacyMultiDexMode();

        AndroidTask<CreateMainDexList> createMainDexListTask = null;
        AndroidTask<RetraceMainDexList> retraceTask = null;

        // Multi-Dex support
        if (isMultiDexEnabled && isLegacyMultiDexMode) {
            // Create a task to collect the list of manifest entry points which are needed in the primary dex
            AndroidTask<CreateManifestKeepList> manifestKeepListTask = androidTasks.create(tasks,
                    new CreateManifestKeepList.ConfigAction(scope, pcData));
            manifestKeepListTask.dependsOn(tasks,
                    variantData.getOutputs().get(0).getScope().getManifestProcessorTask());

            // Create a proguard task to shrink the classes to manifest components
            AndroidTask<ProGuardTask> proguardComponentsTask =
                    androidTasks.create(tasks, new ProGuardTaskConfigAction(scope, pcData));
            proguardComponentsTask.dependsOn(tasks, manifestKeepListTask);
            proguardComponentsTask.optionalDependsOn(tasks,
                    pcData.getClassGeneratingTasks(),
                    pcData.getLibraryGeneratingTasks());

            // Compute the full list of classes for the main dex file
            createMainDexListTask =
                    androidTasks.create(tasks, new CreateMainDexList.ConfigAction(scope, pcData));
            createMainDexListTask.dependsOn(tasks, proguardComponentsTask);

            // If proguard is enabled, create a de-obfuscated list to aid debugging.
            if (isMinifyEnabled) {
                retraceTask = androidTasks.create(tasks,
                        new RetraceMainDexList.ConfigAction(scope, pcData));
                retraceTask.dependsOn(tasks, scope.getObfuscationTask(), createMainDexListTask);
            }

        }


        AndroidTask<Dex> dexTask = androidTasks.create(tasks, new Dex.ConfigAction(scope, pcData));
        scope.setDexTask(dexTask);

        // dependencies, some of these could be null
        dexTask.optionalDependsOn(tasks,
                pcData.getClassGeneratingTasks(),
                pcData.getLibraryGeneratingTasks(),
                createMainDexListTask,
                retraceTask);
    }

上面代码中

（1）如果MultiDexEnabled，则创建CreateManifestKeepList Task，ProGuard Task和CreateMainDexList Task。

其中CreateManifestKeepList Task用来生成manifest_keep.txt文件。ProGuard Task用来对manifest中的组件进行混淆。CreateMainDexList Task用来生成maindexlist.txt文件，如果设置了minifyEnabled true，则创建一个混淆main dex list的task。

（2）创建Dex Task，用来将.class文件生成.dex。

从下面代码可以看出，这部分调用过程为：

a. build-system/gradle-core/src/main/groovy/com/android/build/gradle/tasks/Dex.groovy -> taskAction()
  
b. doTaskAction()

c. build-system/builder/src/main/java/com/android/builder/core/AndroidBuilder.java -> convertByteCode()

实际上另起一个进程进行dex化操作。

    //build-system/gradle-core/src/main/groovy/com/android/build/gradle/tasks/Dex.groovy
    @TaskAction
    void taskAction(IncrementalTaskInputs inputs) {
        Collection<File> _inputFiles = getInputFiles()
        File _inputDir = getInputDir()
        ......
        doTaskAction(_inputFiles, _inputDir, !forceFullRun.get())
    }

    private void doTaskAction(
            @Nullable Collection<File> inputFiles,
            @Nullable File inputDir,
            boolean incremental) {
        ......
        getBuilder().convertByteCode(
                inputFiles,
                getLibraries(),
                outFolder,
                getMultiDexEnabled(),
                getMainDexListFile(),
                getDexOptions(),
                getAdditionalParameters(),
                tmpFolder,
                incremental,
                getOptimize(),
                new LoggedProcessOutputHandler(getILogger())

    //build-system/builder/src/main/java/com/android/builder/core/AndroidBuilder.java
    //Converts the bytecode to Dalvik format
    public void convertByteCode(
            @NonNull Collection<File> inputs,
            @NonNull Collection<File> preDexedLibraries,
            @NonNull File outDexFolder,
                     boolean multidex,
            @Nullable File mainDexList,
            @NonNull DexOptions dexOptions,
            @Nullable List<String> additionalParameters,
            @NonNull File tmpFolder,
            boolean incremental,
            boolean optimize,
            @NonNull ProcessOutputHandler processOutputHandler)
            throws IOException, InterruptedException, ProcessException {
        BuildToolInfo buildToolInfo = mTargetInfo.getBuildTools();
        DexProcessBuilder builder = new DexProcessBuilder(outDexFolder);

        builder.setVerbose(mVerboseExec)
                .setIncremental(incremental)
                .setNoOptimize(!optimize)
                .setMultiDex(multidex)
                .setMainDexList(mainDexList)
                .addInputs(preDexedLibraries)
                .addInputs(verifiedInputs.build());

        if (additionalParameters != null) {
            builder.additionalParameters(additionalParameters);
        }

        JavaProcessInfo javaProcessInfo = builder.build(buildToolInfo, dexOptions);
        ProcessResult result = mJavaProcessExecutor.execute(javaProcessInfo, processOutputHandler);
    }

## 0x02 Class文件Dex过程分析

入口方法为dalvik/dx/src/com/android/dx/command/Main.java的main()。从下面代码可以看出，其调用流程为：

1、 `dalvik/dx/src/com/android/dx/command/Main.java -> main()`

2、`dalvik/dx/src/com/android/dx/command/dexer/Main.java -> main(String[])`

3、`run(Arguments)`

4、`runMultiDex()`

    //dalvik/dx/src/com/android/dx/command/Main.java
    public static void main(String[] args) {
        ......
        if (arg.equals("--dex")) {
            com.android.dx.command.dexer.Main.main(without(args, i));
            break;
        }
        ......
    }

    //dalvik/dx/src/com/android/dx/command/dexer/Main.java
    public static void main(String[] argArray) throws IOException {
        Arguments arguments = new Arguments();
        arguments.parse(argArray);

        int result = run(arguments);
        if (result != 0) {
            System.exit(result);
        }
    }
    
    public static int run(Arguments arguments) throws IOException {    
        args = arguments;
        ......    
        if (args.multiDex) {
            return runMultiDex();
        } else {
            return runMonoDex();
        }
    }

下面重点分析runMultiDex()的逻辑：

遍历所有文件列表，依次对每个文件进行处理。

**1、对于.apk, .zip, .jar文件：**

读取解压后的ZipEntry列表。若为.class文件，则按照下面介绍的Class文件处理方法；若为classes.dex文件，则添加到libraryDexBuffers列表中；若为资源文件，则添加到outputResources的Map中。

**2、对于.class文件：**

#### 2.1 整个过程可以概括为：

（1）将class进行类转换处理 

（2）将转换后的类写入到dex中(multidex会创建多个dex)

（3）将dex转换成byte[]

（4）将byte[]列表依次写入到classes.dex, classes2.dex, classes3.dex...

#### 2.2 相应的里面涉及到几个线程池：

（1）类转化线程池classTranslatorPool。用于将原始class文件转换成ClassDefItem

（2）dex写入线程池classDefItemConsumer。用于将转换后的类依次写到dex中

（3）dex字节码化线程池dexOutPool。用于将dex列表转化为byte[]字节数组，并添加到字节码列表中

#### 2.3 关于multi-dex生成dex过程涉及的几个变量：

（1）numMethodIds

已经写入到dex中的方法数。

（2）maxMethodIdsInClass

我理解的是，这个是还未放到线程池classTranslatorPool中进行转换处理的Class类的最大方法数。其预估值为constantPoolSize + cf.getMethods().size() + MAX_METHOD_ADDED_DURING_DEX_CREATION

（3）maxMethodIdsInProcess

类转化到生成dex中间过程中的最大方法数。

初始值为0，我理解的是，当这个类放到线程池classTranslatorPool中进行转换处理时，maxMethodIdsInProcess值加maxMethodIdsInClass；当将转换后的类放到线程池classDefItemConsumer中，进行dex写入操作时，maxMethodIdsInProcess值减maxMethodIdsInClass。

这样当前预估最大方法数，就可以理解为numMethodIds + maxMethodIdsInClass + maxMethodIdsInProcess，即尚未进行转换处理的Class最大方法数 ＋ 转换／写入过程中的最大方法数 ＋ dex中已有方法数。

#### 2.4 关于multi-dex生成dex过程

若当前预估最大方法数超过单个Dex允许最大方法数时：

（1）若maxMethodIdsInProcess > 0，说明当前还有类在进行类转换，还未写入到dex中，此时应阻塞，等待dex写入操作完成。
若所有类转换操作已完成，且dex中已经填充满，此时将填充满的dex转换成byte[]，并放到List<byte[]> dexOutputArrays列表中。
然后创建新的dex。若预估的最大方法数超出了dex的容量，则跳出循环。

从上面可以看出，multi-dex过程中，需要等到dex写入操作完成，才能继续进行后面的类转化操作。

（2）将上面得到的List<byte[]>列表，依次写到classes.dex，classes2.dex，classes3.dex...

代码如下：

    //dalvik/dx/src/com/android/dx/command/dexer/Main.java
    private static int runMultiDex() throws IOException {
        dexOutPool = Executors.newFixedThreadPool(args.numThreads);

        if (!processAllFiles()) {
            return 1;
        }

        ......
        
        if (args.jarOutput) {
            for (int i = 0; i < dexOutputArrays.size(); i++) {
                outputResources.put(getDexFileName(i),
                        dexOutputArrays.get(i));
            }

            if (!createJar(args.outName)) {
                return 3;
            }
        } else if (args.outName != null) {
            File outDir = new File(args.outName);
            assert outDir.isDirectory();
            for (int i = 0; i < dexOutputArrays.size(); i++) {
                OutputStream out = new FileOutputStream(new File(outDir, getDexFileName(i)));
                try {
                    out.write(dexOutputArrays.get(i));
                } finally {
                    closeOutput(out);
                }
            }
        }

        return 0;
    }

    private static boolean processAllFiles() {
        createDexFile();

        ......
        
        try {
            if (args.mainDexListFile != null) {
                // with --main-dex-list
                FileNameFilter mainPassFilter = args.strictNameCheck ? new MainDexListFilter() :
                    new BestEffortMainDexListFilter();

                // forced in main dex
                for (int i = 0; i < fileNames.length; i++) {
                    processOne(fileNames[i], mainPassFilter);
                }

                if (dexOutputFutures.size() > 0) {
                    throw new DexException("Too many classes in " + Arguments.MAIN_DEX_LIST_OPTION
                            + ", main dex capacity exceeded");
                }

                if (args.minimalMainDex) {
                    // start second pass directly in a secondary dex file.

                    // Wait for classes in progress to complete
                    synchronized(dexRotationLock) {
                        while(maxMethodIdsInProcess > 0 || maxFieldIdsInProcess > 0) {
                            try {
                                dexRotationLock.wait();
                            } catch(InterruptedException ex) {
                                /* ignore */
                            }
                        }
                    }

                    rotateDexFile();
                }

                // remaining files
                for (int i = 0; i < fileNames.length; i++) {
                    processOne(fileNames[i], new NotFilter(mainPassFilter));
                }
            } else {
                // without --main-dex-list
                for (int i = 0; i < fileNames.length; i++) {
                    processOne(fileNames[i], ClassPathOpener.acceptAll);
                }
            }
        } catch (StopProcessing ex) {
            /*
             * Ignore it and just let the error reporting do
             * their things.
             */
        }

        return true;
    }

    //dalvik/dx/src/com/android/dx/cf/direct/ClassPathOpener.java
    private static void processOne(String pathname, FileNameFilter filter) {
        ClassPathOpener opener;

        opener = new ClassPathOpener(pathname, true, filter, new FileBytesConsumer());

        if (opener.process()) {
          updateStatus(true);
        }
    }

    private boolean processOne(File file, boolean topLevel) {
        try {
            if (file.isDirectory()) {
                return processDirectory(file, topLevel);
            }

            String path = file.getPath();

            if (path.endsWith(".zip") ||
                    path.endsWith(".jar") ||
                    path.endsWith(".apk")) {
                return processArchive(file);
            }
            if (filter.accept(path)) {
                byte[] bytes = FileUtils.readFile(file);
                return consumer.processFileBytes(path, file.lastModified(), bytes);
            } else {
                return false;
            }
        } catch (Exception ex) {
            consumer.onException(ex);
            return false;
        }
    }


    private static class FileBytesConsumer implements ClassPathOpener.Consumer {

        @Override
        public boolean processFileBytes(String name, long lastModified,
                byte[] bytes)   {
            return Main.processFileBytes(name, lastModified, bytes);
        }
    }
    
    private static boolean processFileBytes(String name, long lastModified, byte[] bytes) {

        boolean isClass = name.endsWith(".class");
        boolean isClassesDex = name.equals(DexFormat.DEX_IN_JAR_NAME);
        boolean keepResources = (outputResources != null);

        if (!isClass && !isClassesDex && !keepResources) {
            if (args.verbose) {
                DxConsole.out.println("ignored resource " + name);
            }
            return false;
        }

        if (args.verbose) {
            DxConsole.out.println("processing " + name + "...");
        }

        String fixedName = fixPath(name);

        if (isClass) {

            if (keepResources && args.keepClassesInJar) {
                synchronized (outputResources) {
                    outputResources.put(fixedName, bytes);
                }
            }
            if (lastModified < minimumFileAge) {
                return true;
            }
            processClass(fixedName, bytes);
            // Assume that an exception may occur. Status will be updated
            // asynchronously, if the class compiles without error.
            return false;
        } else if (isClassesDex) {
            synchronized (libraryDexBuffers) {
                libraryDexBuffers.add(bytes);
            }
            return true;
        } else {
            synchronized (outputResources) {
                outputResources.put(fixedName, bytes);
            }
            return true;
        }
    }


    private static boolean processClass(String name, byte[] bytes) {
        if (! args.coreLibrary) {
            checkClassName(name);
        }

        try {
            new DirectClassFileConsumer(name, bytes, null).call(
                    new ClassParserTask(name, bytes).call());
        } catch (ParseException ex) {
            // handled in FileBytesConsumer
            throw ex;
        } catch(Exception ex) {
            throw new RuntimeException("Exception parsing classes", ex);
        }

        return true;
    }


    private static class DirectClassFileConsumer implements Callable<Boolean> {

        private Boolean call(DirectClassFile cf) {

            int maxMethodIdsInClass = 0;
            int maxFieldIdsInClass = 0;

            if (args.multiDex) {

                // Calculate max number of indices this class will add to the
                // dex file.
                // The possibility of overloading means that we can't easily
                // know how many constant are needed for declared methods and
                // fields. We therefore make the simplifying assumption that
                // all constants are external method or field references.

                int constantPoolSize = cf.getConstantPool().size();
                maxMethodIdsInClass = constantPoolSize + cf.getMethods().size()
                        + MAX_METHOD_ADDED_DURING_DEX_CREATION;
                maxFieldIdsInClass = constantPoolSize + cf.getFields().size()
                        + MAX_FIELD_ADDED_DURING_DEX_CREATION;
                synchronized(dexRotationLock) {

                    int numMethodIds;
                    int numFieldIds;
                    // Number of indices used in current dex file.
                    synchronized(outputDex) {
                        numMethodIds = outputDex.getMethodIds().items().size();
                        numFieldIds = outputDex.getFieldIds().items().size();
                    }
                    // Wait until we're sure this class will fit in the current
                    // dex file.
                    while(((numMethodIds + maxMethodIdsInClass + maxMethodIdsInProcess
                            > args.maxNumberOfIdxPerDex) ||
                           (numFieldIds + maxFieldIdsInClass + maxFieldIdsInProcess
                            > args.maxNumberOfIdxPerDex))) {

                        if (maxMethodIdsInProcess > 0 || maxFieldIdsInProcess > 0) {
                            // There are classes in the translation phase that
                            // have not yet been added to the dex file, so we
                            // wait for the next class to complete.
                            try {
                                dexRotationLock.wait();
                            } catch(InterruptedException ex) {
                                /* ignore */
                            }
                        } else if (outputDex.getClassDefs().items().size() > 0) {
                            // There are no further classes in the translation
                            // phase, and we have a full dex file. Rotate!
                            rotateDexFile();
                        } else {
                            // The estimated number of indices is too large for
                            // an empty dex file. We proceed hoping the actual
                            // number of indices needed will fit.
                            break;
                        }
                        synchronized(outputDex) {
                            numMethodIds = outputDex.getMethodIds().items().size();
                            numFieldIds = outputDex.getFieldIds().items().size();
                        }
                    }
                    // Add our estimate to the total estimate for
                    // classes under translation.
                    maxMethodIdsInProcess += maxMethodIdsInClass;
                    maxFieldIdsInProcess += maxFieldIdsInClass;
                }
            }

            // Submit class to translation phase.
            Future<ClassDefItem> cdif = classTranslatorPool.submit(
                    new ClassTranslatorTask(name, bytes, cf));
            Future<Boolean> res = classDefItemConsumer.submit(new ClassDefItemConsumer(
                    name, cdif, maxMethodIdsInClass, maxFieldIdsInClass));
            addToDexFutures.add(res);

            return true;
        }
    }
    
## 0x03 总结

构建一个apk的入口为BasePlugin.apply(Project)，构建过程由各种Android Tasks组成。    
其中Compile Task，用于将.java文件编译成.class文件。    
对于PostCompile Task，如果支持multi dex，则先生成manifest_keep.txt，maindexlist.txt文件并对组件类进行混淆操作。最后另起一个进程将.class写入dex包。

具体来说，关于.class文件到dex写入过程主要为：

（1）将class进行类转换处理 

（2）将转换后的类写入到dex中(multidex会创建多个dex)

（3）将dex转换成byte[]

（4）将byte[]列表依次写入到classes.dex, classes2.dex, classes3.dex...

本文并没有讲述manifest_keep.txt，maindexlist.txt生成原理，需要等到后面研究了之后再写。

可能也没有讲清楚或有遗漏，欢迎指正。


