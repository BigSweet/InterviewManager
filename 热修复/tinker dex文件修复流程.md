上篇文章写了tinker的使用方法
[tinker使用](https://blog.csdn.net/qq_15527709/article/details/61921447)
本篇文章简单的分析一下dex文件的修复流程
在几年前我对热修复的理解为
新apk和旧的apk通过dexbiff算法对比生成差异包,差异包通过比对文件的MD5值，把修改过的文件打进差异包，差异包下发到服务器，下载到手机指定的路径,
通过这个下载的路径生成dexclassloader，获取dexclassloader中的的pathlist，在获取Element数组，然后获取系统的pathclassloader和他的element数组，把这个差异包的Element数组塞到系统默认的element数组里面。通过反射把新的element数组设置给pathclassloader然后下次启动的时候就会加载差异包的dex类

倒是最近发现阅读tinker的源码的时候，发现还是有一些不一样的地方
首先从入口代码开始看
```
TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
```
这个tinker加载patch包的代码
继续跟踪
```
 public static void onReceiveUpgradePatch(Context context, String patchLocation) {
        Tinker.with(context).getPatchListener().onPatchReceived(patchLocation);
    }
    getPatchListenr是一个接口，直接看它的实现类DefaultPatchListener
    
    @Override
    public int onPatchReceived(String path) {
        final File patchFile = new File(path);
        final String patchMD5 = SharePatchFileUtil.getMD5(patchFile);
        final int returnCode = patchCheck(path, patchMD5);
        if (returnCode == ShareConstants.ERROR_PATCH_OK) {
            runForgService();
            TinkerPatchService.runPatchService(context, path);
        } else {
            Tinker.with(context).getLoadReporter().onLoadPatchListenerReceiveFail(new File(path), returnCode);
        }
        return returnCode;
    }
    
   有一个MD5的校验
   接下来看这一行
    TinkerPatchService.runPatchService(context, path);

  public static void runPatchService(final Context context, final String path) {
        Intent intent = new Intent(context, TinkerPatchService.class);
        intent.putExtra(PATCH_PATH_EXTRA, path);
        intent.putExtra(RESULT_CLASS_EXTRA, resultServiceClass.getName());
        try {
            context.startService(intent);
        } catch (Throwable thr) {
            ShareTinkerLog.e(TAG, "run patch service fail, exception:" + thr);
        }
    }
    
```
直接开启了一个server，而且这个server是有自己独立的进程的在manifest里面可以看到 android:process=":patch"
```
 <service
            android:name="com.tencent.tinker.lib.service.TinkerPatchService"
            android:exported="false"
            android:permission="android.permission.BIND_JOB_SERVICE"
            android:process=":patch" />
```
在这个server的onHandleIntent中加载了补丁包
```
 @Override
    protected void onHandleIntent(Intent intent) {
        increasingPriority();
        doApplyPatch(this, intent);
    }
 private static void doApplyPatch(Context context, Intent intent) {
       ....
 		result = upgradePatchProcessor.tryPatch(context, path, patchResult);
        AbstractResultService.runResultService(context, patchResult, getPatchResultExtra(intent));

        sIsPatchApplying.set(false);
    }
继续看tryPatch，位于UpgradePatch类中

	   @Override
    public boolean tryPatch(Context context, String tempPatchPath, PatchResult patchResult) {
    	...
    	  //we use destPatchFile instead of patchFile, because patchFile may be deleted during the patch process
        if (!DexDiffPatchInternal.tryRecoverDexFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile, patchResult)) {
            ShareTinkerLog.e(TAG, "UpgradePatch tryPatch:new patch recover, try patch dex failed");
            return false;
        }
        ...
    }
	 因为只是简单看一下dex文件的修复，所以只截取了相关的代码
	 关键代码为
	 DexDiffPatchInternal.tryRecoverDexFiles
	
	protected static boolean tryRecoverDexFiles(Tinker manager, ShareSecurityCheck checker, Context context,
                                                String patchVersionDirectory, File patchFile, PatchResult patchResult) {
       ...
        boolean result = patchDexExtractViaDexDiff(context, patchVersionDirectory, dexMeta, patchFile, patchResult);
        long cost = SystemClock.elapsedRealtime() - begin;
        ShareTinkerLog.i(TAG, "recover dex result:%b, cost:%d", result, cost);
        return result;
    }
	->patchDexExtractViaDexDiff
	
	 private static boolean patchDexExtractViaDexDiff(Context context, String patchVersionDirectory, String meta, final File patchFile, PatchResult patchResult) {
       ...
        return dexOptimizeDexFiles(context, legalFiles, optimizeDexDirectory, patchFile, patchResult);
    }
    
    ->dexOptimizeDexFiles
 private static boolean dexOptimizeDexFiles(Context context, List<File> dexFiles, String optimizeDexDirectory, final File patchFile, final PatchResult patchResult) {
        final Tinker manager = Tinker.with(context);

        optFiles.clear();

        if (dexFiles != null) {
            File optimizeDexDirectoryFile = new File(optimizeDexDirectory);

            if (!optimizeDexDirectoryFile.exists() && !optimizeDexDirectoryFile.mkdirs()) {
                ShareTinkerLog.w(TAG, "patch recover, make optimizeDexDirectoryFile fail");
                return false;
            }
            // add opt files
            for (File file : dexFiles) {
                String outputPathName = SharePatchFileUtil.optimizedPathFor(file, optimizeDexDirectoryFile);
                optFiles.add(new File(outputPathName));
            }

            ShareTinkerLog.i(TAG, "patch recover, try to optimize dex file count:%d, optimizeDexDirectory:%s", dexFiles.size(), optimizeDexDirectory);
           ...
            TinkerDexOptimizer.optimizeAll(
                  context, dexFiles, optimizeDexDirectoryFile,
                  useDLC,
                  new TinkerDexOptimizer.ResultCallback() {
                      long startTime;

                      @Override
                      public void onStart(File dexFile, File optimizedDir) {
                          startTime = System.currentTimeMillis();
                          ShareTinkerLog.i(TAG, "start to parallel optimize dex %s, size: %d", dexFile.getPath(), dexFile.length());
                      }

                      @Override
                      public void onSuccess(File dexFile, File optimizedDir, File optimizedFile) {
                          ShareTinkerLog.i(TAG, "success to parallel optimize dex %s, opt file:%s, opt file size: %d, use time %d",
                              dexFile.getPath(), optimizedFile.getPath(), optimizedFile.length(), (System.currentTimeMillis() - startTime));
                          if (!optimizedFile.exists()) {
                              synchronized (anyOatNotGenerated) {
                                  anyOatNotGenerated[0] = true;
                              }
                          }
                      }

                      @Override
                      public void onFailed(File dexFile, File optimizedDir, Throwable thr) {
                          ShareTinkerLog.i(TAG, "fail to parallel optimize dex %s use time %d",
                              dexFile.getPath(), (System.currentTimeMillis() - startTime));
                          failOptDexFile.add(dexFile);
                          throwable[0] = thr;
                      }
                  }
            );
		...
        return true;
    }

->optimizeAll
public static boolean optimizeAll(Context context, Collection<File> dexFiles, File optimizedDir,
                                      boolean useDLC, ResultCallback cb) {
        return optimizeAll(context, dexFiles, optimizedDir, false, useDLC, null, cb);
    }

    public static boolean optimizeAll(Context context, Collection<File> dexFiles, File optimizedDir,
                                      boolean useInterpretMode, boolean useDLC,
                                      String targetISA, ResultCallback cb) {
        ArrayList<File> sortList = new ArrayList<>(dexFiles);
        // sort input dexFiles with its file length in reverse order.
       ...
        for (File dexFile : sortList) {
            OptimizeWorker worker = new OptimizeWorker(context, dexFile, optimizedDir, useInterpretMode,
                  useDLC, targetISA, cb);
            if (!worker.run()) {
                return false;
            }
        }
        return true;
    }
    ->OptimizeWorker.run
 boolean run() {
            try {
             ...
                if (!ShareTinkerInternals.isArkHotRuning()) {
                    if (useInterpretMode) {
                        interpretDex2Oat(dexFile.getAbsolutePath(), optimizedPath);
                    } else if (Build.VERSION.SDK_INT >= 26
                            || (Build.VERSION.SDK_INT >= 25 && Build.VERSION.PREVIEW_SDK_INT != 0)) {
                        NewClassLoaderInjector.triggerDex2Oat(context, optimizedDir,
                                                              useDLC, dexFile.getAbsolutePath());
                        // Android Q is significantly slowed down by Fallback Dex Loading procedure, so we
                        // trigger background dexopt to generate executable odex here.
                        triggerPMDexOptOnDemand(context, dexFile.getAbsolutePath(), optimizedPath);
                    } else {
                        DexFile.loadDex(dexFile.getAbsolutePath(), optimizedPath, 0);
                    }
                }
                ...
            return true;
        }
        
        因为现在andorid 版本都比较新，所以直接看(Build.VERSION.SDK_INT >= 26）里面的代码
        ->NewClassLoaderInjector.triggerDex2Oat
        public static void triggerDex2Oat(Context context, File dexOptDir, boolean useDLC,
                                      String... dexPaths) throws Throwable {
        final ClassLoader triggerClassLoader = createNewClassLoader(context.getClassLoader(), dexOptDir, useDLC, dexPaths);
    }
->createNewClassLoader
这里就是dex文件修复的关键代码了
private static ClassLoader createNewClassLoader(ClassLoader oldClassLoader,
                                                    File dexOptDir,
                                                    boolean useDLC,
                                                    String... patchDexPaths) throws Throwable {
        final Field pathListField = findField(
                Class.forName("dalvik.system.BaseDexClassLoader", false, oldClassLoader),
                "pathList");
        final Object oldPathList = pathListField.get(oldClassLoader);
        final StringBuilder dexPathBuilder = new StringBuilder();
        final boolean hasPatchDexPaths = patchDexPaths != null && patchDexPaths.length > 0;
        if (hasPatchDexPaths) {
            for (int i = 0; i < patchDexPaths.length; ++i) {
                if (i > 0) {
                    dexPathBuilder.append(File.pathSeparator);
                }
                dexPathBuilder.append(patchDexPaths[i]);
            }
        }
        final String combinedDexPath = dexPathBuilder.toString();
        final Field nativeLibraryDirectoriesField = findField(oldPathList.getClass(), "nativeLibraryDirectories");
        List<File> oldNativeLibraryDirectories = null;
        if (nativeLibraryDirectoriesField.getType().isArray()) {
            oldNativeLibraryDirectories = Arrays.asList((File[]) nativeLibraryDirectoriesField.get(oldPathList));
        } else {
            oldNativeLibraryDirectories = (List<File>) nativeLibraryDirectoriesField.get(oldPathList);
        }
        final StringBuilder libraryPathBuilder = new StringBuilder();
        boolean isFirstItem = true;
        for (File libDir : oldNativeLibraryDirectories) {
            if (libDir == null) {
                continue;
            }
            if (isFirstItem) {
                isFirstItem = false;
            } else {
                libraryPathBuilder.append(File.pathSeparator);
            }
            libraryPathBuilder.append(libDir.getAbsolutePath());
        }

        final String combinedLibraryPath = libraryPathBuilder.toString();

        ClassLoader result = null;
        if (useDLC && Build.VERSION.SDK_INT >= 27) {
            result = new DelegateLastClassLoader(combinedDexPath, combinedLibraryPath, ClassLoader.getSystemClassLoader());
            final Field parentField = ClassLoader.class.getDeclaredField("parent");
            parentField.setAccessible(true);
            parentField.set(result, oldClassLoader);
        } else {
            result = new TinkerClassLoader(combinedDexPath, dexOptDir, combinedLibraryPath, oldClassLoader);
        }

        // 'EnsureSameClassLoader' mechanism which is first introduced in Android O
        // may cause exception if we replace definingContext of old classloader.
        if (Build.VERSION.SDK_INT < 26) {
            findField(oldPathList.getClass(), "definingContext").set(oldPathList, result);
        }

        return result;
    }
   

```


到这里就开始分开分析了
首先看前面的代码
```
 final StringBuilder dexPathBuilder = new StringBuilder();
        final boolean hasPatchDexPaths = patchDexPaths != null && patchDexPaths.length > 0;
        if (hasPatchDexPaths) {
            for (int i = 0; i < patchDexPaths.length; ++i) {
                if (i > 0) {
                    dexPathBuilder.append(File.pathSeparator);
                }
                dexPathBuilder.append(patchDexPaths[i]);
            }
        }

        final String combinedDexPath = dexPathBuilder.toString();
```
patchDexPaths是patch包的路径
最后转换成了combinedDexPath用File.pathSeparator分隔

```
final Field nativeLibraryDirectoriesField = findField(oldPathList.getClass(), "nativeLibraryDirectories");
        List<File> oldNativeLibraryDirectories = null;
        if (nativeLibraryDirectoriesField.getType().isArray()) {
            oldNativeLibraryDirectories = Arrays.asList((File[]) nativeLibraryDirectoriesField.get(oldPathList));
        } else {
            oldNativeLibraryDirectories = (List<File>) nativeLibraryDirectoriesField.get(oldPathList);
        }
        final StringBuilder libraryPathBuilder = new StringBuilder();
        boolean isFirstItem = true;
        for (File libDir : oldNativeLibraryDirectories) {
            if (libDir == null) {
                continue;
            }
            if (isFirstItem) {
                isFirstItem = false;
            } else {
                libraryPathBuilder.append(File.pathSeparator);
            }
            libraryPathBuilder.append(libDir.getAbsolutePath());
        }

        final String combinedLibraryPath = libraryPathBuilder.toString();
```
这里和上面一样是将老的oldPathList的路径通过StringBuilder append到一起组合成了combinedLibraryPath
最后通过反射把新的classloader替换了旧的
```
result = new DelegateLastClassLoader(combinedDexPath, combinedLibraryPath, ClassLoader.getSystemClassLoader());
            final Field parentField = ClassLoader.class.getDeclaredField("parent");
            parentField.setAccessible(true);
            parentField.set(result, oldClassLoader);
```
关于这个DelegateLastClassLoader可以在这里看到详细的介绍
[DelegateLastClassLoader](https://developer.android.google.cn/reference/dalvik/system/DelegateLastClassLoader.html?hl=zh-tw)

最后回到doApplyPatch方法中
加载完成之后开启了一个AbstractResultService.runResultService


	在这个server的onHandleIntent调用了onPatchResult
	 @Override
	protected void onHandleIntent(Intent intent) {
	    if (intent == null) {
	        ShareTinkerLog.e(TAG, "AbstractResultService received a null intent, ignoring.");
	        return;
	    }
	    PatchResult result = (PatchResult) ShareIntentUtil.getSerializableExtra(intent, RESULT_EXTRA);
	
	    onPatchResult(result);
	}
	public abstract void onPatchResult(PatchResult result);
	 这个类是一个抽象类，实现类为DefaultTinkerResultService
	 
	  @Override
	public void onPatchResult(PatchResult result) {
	 
	    TinkerServiceInternals.killTinkerPatchServiceProcess(getApplicationContext());
	    if (result.isSuccess) {
	        deleteRawPatchFile(new File(result.rawPatchFilePath));//删除原来的路径的文件
	        if (checkIfNeedKill(result)) {
	            android.os.Process.killProcess(android.os.Process.myPid());
	        } else {
	            ShareTinkerLog.i(TAG, "I have already install the newly patch version!");
	        }
	    }
	}
最后把这个补丁包文件删除，然后杀掉开启了的新进程
到这里dex文件修复的流程就跟踪完了，
那核心方法就是通过反射把原来的类加载器换成了新的，和我文章开头的理解还是有一点不一样的，不过在我的印象中，好像tinker的某一个版本是用的我文章开头使用的方法后续因为android版本的原因才替换的？
参考资料
[混合编译对tinker的影响](https://github.com/WeMobileDev/article/blob/master/Android_N%E6%B7%B7%E5%90%88%E7%BC%96%E8%AF%91%E4%B8%8E%E5%AF%B9%E7%83%AD%E8%A1%A5%E4%B8%81%E5%BD%B1%E5%93%8D%E8%A7%A3%E6%9E%90.md)
[tinker](https://github.com/Tencent/tinker)