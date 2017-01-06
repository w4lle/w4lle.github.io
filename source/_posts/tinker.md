---
title: Android热补丁之Tinker原理解析
date: 2016-12-16 09:49:06
tags: [Android, 热补丁]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/tinker.jpg
---

> 本文分析版本  [93ecc9351367badc02a91fac25764bee50e6e6a6](https://github.com/Tencent/tinker/tree/93ecc9351367badc02a91fac25764bee50e6e6a6) 
项目地址： [Tinker](https://github.com/Tencent/tinker/)

# 背景

在今年的MDCC大会上，微信开发团队宣布正式开源Tinker，在这之前微信团队已经发出过一些Tinker的相关文章，说实话在开源之前我们还是相当期待Tinker开源的，一方面是因为之前使用的热补丁一直存在一些兼容性问题，另一方面也好奇Tinker的实现方案。

<!-- more -->

在开源后我们团队第一时间着手研究Tinker，在详细阅读了源码之后，我们确定要在之后的一个版本集成Tinker上线，线上效果显示Tinker的修复效果果然牛逼，错误率明显下降的同时也没有报出兼容性的问题。附一张薄荷app使用Tinker修复前后的错误率对比。

![](http://7xs23g.com1.z0.glb.clouddn.com/fatal.png)

# 从接入Tinker入手

想要深入某个框架，前提是要学会使用它。我们就从Tinker的接入入手一步一步解开它的实现原理。参照[wiki](https://github.com/Tencent/tinker/wiki)我们做了如下操作。

实现一个Application

```java
public class OneApplicationForTinker extends TinkerApplication {
    public OneApplicationForTinker() {
        super(
                //tinkerFlags, tinker支持的类型，dex,library，还是全部都支持！
                ShareConstants.TINKER_ENABLE_ALL,
                //ApplicationLike的实现类，只能传递字符串,不能使用class.getName()
                "com.boohee.one.MyApplication",
                //加载Tinker的主类名，对于特殊需求可能需要使用自己的加载类。需要注意的是：
                //这个类以及它使用的类都是不能被补丁修改的，并且我们需要将它们加到dex.loader[]中。
                //一般来说，我们使用默认即可。
                "com.tencent.tinker.loader.TinkerLoader",
                //由于合成过程中我们已经校验了各个文件的Md5，并将它们存放在/data/data/..目录中。
                // 默认每次加载时我们并不会去校验tinker文件的Md5,但是你也可通过开启loadVerifyFlag强制每次加载时校验，
                // 但是这会带来一定的时间损耗。
                false);
    }
}
```

其中的几个参数做了详细说明，Tinker其实提供了注解的方式生成该类，但是我们为了更清楚的了解Tinker的原理，所以并没有使用注解。

然后在``AndroidManifest.xml``中声明该类为``application``

```java
...
<application
        android:name=".tinker.OneApplicationForTinker"
        ...
</application>
```

那我们就知道了，app的入口Application就是该类，该类继承自TinkerApplication。然后我们项目中的MyApplication继承自ApplicationLike，其实看到这里，就大概猜到了OneApplicationForTinker可能是一个代理，App中的Application的真正实现还是MyApplication。

# Application的替换

为了做分析前的铺垫，我们从最开始的接入入手，实现了OneApplicationForTinker，继承自TinkerApplication。我们继续往下看。
看下TinkerApplication的实现

```java
public abstract class TinkerApplication extends Application {
...

    protected TinkerApplication(int tinkerFlags, String delegateClassName,
                                String loaderClassName, boolean tinkerLoadVerifyFlag) {
        this.tinkerFlags = tinkerFlags;
        this.delegateClassName = delegateClassName;
        this.loaderClassName = loaderClassName;
        this.tinkerLoadVerifyFlag = tinkerLoadVerifyFlag;

    }

    private Object createDelegate() {
        try {
            // Use reflection to create the delegate so it doesn't need to go into the primary dex.
            // And we can also patch it
            Class<?> delegateClass = Class.forName(delegateClassName, false, getClassLoader());
            Constructor<?> constructor = delegateClass.getConstructor(Application.class, int.class, boolean.class, long.class, long.class,
                Intent.class, Resources[].class, ClassLoader[].class, AssetManager[].class);
            return constructor.newInstance(this, tinkerFlags, tinkerLoadVerifyFlag,
                applicationStartElapsedTime, applicationStartMillisTime,
                tinkerResultIntent, resources, classLoader, assetManager);
        } catch (Throwable e) {
            throw new TinkerRuntimeException("createDelegate failed", e);
        }
    }

    private synchronized void ensureDelegate() {
        if (delegate == null) {
            delegate = createDelegate();
        }
    }

    /**
     * Hook for sub-classes to run logic after the {@link Application#attachBaseContext} has been
     * called but before the delegate is created. Implementors should be very careful what they do
     * here since {@link android.app.Application#onCreate} will not have yet been called.
     */
    private void onBaseContextAttached(Context base) {
        applicationStartElapsedTime = SystemClock.elapsedRealtime();
        applicationStartMillisTime = System.currentTimeMillis();
        loadTinker();
        ensureDelegate();
        try {
            Method method = ShareReflectUtil.findMethod(delegate, "onBaseContextAttached", Context.class);
            method.invoke(delegate, base);
        } catch (Throwable t) {
            throw new TinkerRuntimeException("onBaseContextAttached method not found", t);
        }
        //重置安全模式次数，大于等于三次会进入安全模式不再加载
        if (useSafeMode) {
            String processName = ShareTinkerInternals.getProcessName(this);
            String preferName = ShareConstants.TINKER_OWN_PREFERENCE_CONFIG + processName;
            SharedPreferences sp = getSharedPreferences(preferName, Context.MODE_PRIVATE);
            sp.edit().putInt(ShareConstants.TINKER_SAFE_MODE_COUNT, 0).commit();
        }
    }

    @Override
    protected final void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        onBaseContextAttached(base);
    }

    private void delegateMethod(String methodName) {
        if (delegate != null) {
            try {
                Method method = ShareReflectUtil.findMethod(delegate, methodName, new Class[0]);
                method.invoke(delegate, new Object[0]);
            } catch (Throwable t) {
                throw new TinkerRuntimeException(String.format("%s method not found", methodName), t);
            }
        }
    }

    @Override
    public final void onCreate() {
        super.onCreate();
        ensureDelegate();
        delegateMethod("onCreate");
    }
}
```

TinkerApplication继承自Application，说明它是正经的Application，而且在manifest文件中声明的也必须是它。然后在Application的各个声明周期方法中反射调用``delegate``同步Application的周期方法回调，其中的``delegate``是我们传过来的我们项目中的Application ``MyApplication``。

其中的loaderTinker()方法是Tinker的加载流程，我们稍后会讲到，在反射调用MyApplication的attachBaseContext之前，loaderTinker()已经被调用完成，也就是说，Tinker是在加载完整个流程之后才去调用的app中的Application的attachBaseContext开始真正的整个App的生命周期。说白了就是采用了代理。

看到这里，如果你看过我之前写的[从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)，就会发现跟这个好像。区别在于，InstantRun是在编译器修改manifest插入IncrementalClassLoader，运行时动态替换成项目中实际使用的MyApplication，进而替换了ClassLoader和资源等，开发者在毫不知情的情况下就完成了替换。

其中大量使用了反射，hook系统api，替换运行时系统中保有的Application的引用，最终完成替换，Tinker团队之前做过测试，100w人会有几十个在替换的时候出现问题，而且如果反射替换Application的问题，那么这个过程是不可逆的。Tinker为了兼容性问题考虑，采用了工程代理的方式，避免进入兼容性的坑。虽然可以用注解的方式生成，但是这种方式相比InstantRun的那一套接入成本还是增大不少，不过为了线上的稳定，这一切都是值得的。

还有一点需要注意的是，TinkerApplication是采用反射调用的MyApplication，为什么一定是反射，我们直接传过去MyApplication的引用直接调用不就好了吗？关于这一点，我们后面会详细说明。

# 补丁加载

在补丁加载之前，我们需要知道补丁文件现在已经下发到app中，并且通过dexDiff合成并且校验然后push到``/data/data/package_name/tinker/``下。大概的文件目录：

```java
root@android:/data/data/tinker.sample.android/tinker # ls
info.lock
patch-bc7c9396
patch.info

root@android:/data/data/tinker.sample.android/tinker/patch-bc7c9396 # ls
dex
odex
patch-bc7c9396.apk
res
```

刚才讲到loadTinker()方法是实现Tinker加载补丁的关键，我们继续看下实现

```java
    private void loadTinker() {
        //disable tinker, not need to install
        if (tinkerFlags == TINKER_DISABLE) {
            return;
        }
        tinkerResultIntent = new Intent();
        try {
            //reflect tinker loader, because loaderClass may be define by user!
            Class<?> tinkerLoadClass = Class.forName(loaderClassName, false, getClassLoader());

            Method loadMethod = tinkerLoadClass.getMethod(TINKER_LOADER_METHOD, TinkerApplication.class, int.class, boolean.class);
            Constructor<?> constructor = tinkerLoadClass.getConstructor();
            tinkerResultIntent = (Intent) loadMethod.invoke(constructor.newInstance(), this, tinkerFlags, tinkerLoadVerifyFlag);
        } catch (Throwable e) {
            //has exception, put exception error code
            ShareIntentUtil.setIntentReturnCode(tinkerResultIntent, ShareConstants.ERROR_LOAD_PATCH_UNKNOWN_EXCEPTION);
            tinkerResultIntent.putExtra(INTENT_PATCH_EXCEPTION, e);
        }
    }
```

其中的``loaderClassName``是我们传过来的``"com.tencent.tinker.loader.TinkerLoader"``，反射调用TinkerLoader的tryLoad()方法拿到加载补丁结果，这里为什么也要用反射，是因为Tinker做了很多扩展性的工作，TinkerLoader只是默认实现，开发者完全可以自己定义加载器完成加载流程。

```java
//TinkerLoader
    /**
     * only main process can handle patch version change or incomplete
     */
    @Override
    public Intent tryLoad(TinkerApplication app, int tinkerFlag, boolean tinkerLoadVerifyFlag) {
        Intent resultIntent = new Intent();

        long begin = SystemClock.elapsedRealtime();
        tryLoadPatchFilesInternal(app, tinkerFlag, tinkerLoadVerifyFlag, resultIntent);
        long cost = SystemClock.elapsedRealtime() - begin;
        ShareIntentUtil.setIntentPatchCostTime(resultIntent, cost);
        return resultIntent;
    }
```

调用tryLoadPatchFilesInternal()方法，然后计算消耗时间。

```java
    private void tryLoadPatchFilesInternal(TinkerApplication app, int tinkerFlag, boolean tinkerLoadVerifyFlag, Intent resultIntent) {
    ...
    ...
        //tinker/patch.info
        File patchInfoFile = SharePatchFileUtil.getPatchInfoFile(patchDirectoryPath);

        //old = 641e634c5b8f1649c75caf73794acbdf
        //new = 2c150d8560334966952678930ba67fa8
        File patchInfoLockFile = SharePatchFileUtil.getPatchInfoLockFile(patchDirectoryPath);

        patchInfo = SharePatchInfo.readAndCheckPropertyWithLock(patchInfoFile, patchInfoLockFile);

        String oldVersion = patchInfo.oldVersion;
        String newVersion = patchInfo.newVersion;

        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_OLD_VERSION, oldVersion);
        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_NEW_VERSION, newVersion);

        boolean mainProcess = ShareTinkerInternals.isInMainProcess(app);
        boolean versionChanged = !(oldVersion.equals(newVersion));

        String version = oldVersion;
        if (versionChanged && mainProcess) {
            version = newVersion;
        }

        //patch-641e634c
        String patchName = SharePatchFileUtil.getPatchVersionDirectory(version);

        //tinker/patch.info/patch-641e634c
        String patchVersionDirectory = patchDirectoryPath + "/" + patchName;
        File patchVersionDirectoryFile = new File(patchVersionDirectory);

        //tinker/patch.info/patch-641e634c/patch-641e634c.apk
        File patchVersionFile = new File(patchVersionDirectoryFile.getAbsolutePath(), SharePatchFileUtil.getPatchVersionFile(version));

        ShareSecurityCheck securityCheck = new ShareSecurityCheck(app);

//校验签名和tinkerId
        int returnCode = ShareTinkerInternals.checkSignatureAndTinkerID(app, patchVersionFile, securityCheck);

        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_PACKAGE_CONFIG, securityCheck.getPackagePropertiesIfPresent());

        final boolean isEnabledForDex = ShareTinkerInternals.isTinkerEnabledForDex(tinkerFlag);

        if (isEnabledForDex) {
            //tinker/patch.info/patch-641e634c/dex
            boolean dexCheck = TinkerDexLoader.checkComplete(patchVersionDirectory, securityCheck, resultIntent);
            if (!dexCheck) {
                //file not found, do not load patch
                return;
            }
        }

        final boolean isEnabledForNativeLib = ShareTinkerInternals.isTinkerEnabledForNativeLib(tinkerFlag);

        if (isEnabledForNativeLib) {
            //tinker/patch.info/patch-641e634c/lib
            boolean libCheck = TinkerSoLoader.checkComplete(patchVersionDirectory, securityCheck, resultIntent);
            if (!libCheck) {
                //file not found, do not load patch
                return;
            }
        }

        //check resource
        final boolean isEnabledForResource = ShareTinkerInternals.isTinkerEnabledForResource(tinkerFlag);
        Log.w(TAG, "tryLoadPatchFiles:isEnabledForResource:" + isEnabledForResource);
        if (isEnabledForResource) {
            boolean resourceCheck = TinkerResourceLoader.checkComplete(app, patchVersionDirectory, securityCheck, resultIntent);
            if (!resourceCheck) {
                //file not found, do not load patch
                return;
            }
        }
        //we should first try rewrite patch info file, if there is a error, we can't load jar
        if (mainProcess && versionChanged) {
            patchInfo.oldVersion = version;
            //update old version to new
            ...
        }
        //是否已经进入安全模式
        if (!checkSafeModeCount(app)) {
            ...
            return;
        }
        //now we can load patch jar
        if (isEnabledForDex) {
            boolean loadTinkerJars = TinkerDexLoader.loadTinkerJars(app, tinkerLoadVerifyFlag, patchVersionDirectory, resultIntent);
        }

        //now we can load patch resource
        if (isEnabledForResource) {
            boolean loadTinkerResources = TinkerResourceLoader.loadTinkerResources(tinkerLoadVerifyFlag, patchVersionDirectory, resultIntent);
        }
        //all is ok!
        ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_OK);
        Log.i(TAG, "tryLoadPatchFiles: load end, ok!");
        return;
    }
```

贴的代码省略了好多判空操作，会判断补丁是否存在，检查补丁信息中的数据是否有效，校验补丁签名以及tinkerId与基准包是否一致。在校验签名时，为了加速校验速度，Tinker只校验 ``*_meta.txt``文件，然后再根据meta文件中的md5校验其他文件。
其中，meta文件有以下几种：

 - package_meta.txt 补丁包的基本信息
 - dex_meta.txt     补丁包中dex文件的信息
 - so_meta.txt      补丁包中so文件的信息
 - res_meta.txt     补丁包中资源文件的信息
 
然后根据开发者配置的Tinker可补丁类型判断是否可以加载dex，res，so。然后分别分发给TinkerDexLoader、TinkerSoLoader、TinkerResourceLoader分别进行校验是否符合加载条件进而进行加载。

## 加载补丁dex

在开始讲load dex之前，先说下Tinker的补丁方案，Tinker采用的是下发差分包，然后在手机端合成全量的dex文件进行加载。而在build.gradle配置中的tinkerPatch

```java
    dex.loader = ["com.tencent.tinker.loader.*",
    "tinker.sample.android.app.SampleApplication",
    "tinker.sample.android.app.BaseBuildInfo"
    ]
```

这个配置中的类不会出现在任何全量补丁dex里，也就是说在合成后，这些类还在老的dex文件中，比如在补丁前dex顺序是这样的：``oldDex1 -> oldDex2 -> oldDex3..``，那么假如修改了dex1中的文件，那么补丁顺序是这样的``newDex1 -> oldDex1 -> oldDex2...``其中合成后的newDex1中的类是oldDex1中除了dex.loader中标明的类之外的所有类，dex.loader中的类依然在oldDex1中。

由于Tinker的方案是基于Multidex实现的修改dexElements的顺序实现的，所以最终还是要修改classLoder中dexPathList中dexElements的顺序。Android中有两种ClassLoader用于加载dex文件，BootClassLoader、PathClassLoader和DexClassLoader都是继承自BaseDexClassLoader

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);

        this.originalPath = dexPath;
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = pathList.findClass(name);

        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }

        return clazz;
    }
    
//DexPathList
    public Class findClass(String name) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        return null;
    }
```

最终在DexPathList的findClass中遍历dexElements，谁在前面用谁。而这个dexElements是在方法makeDexElements中生成的，我们的目的就是hook这个方法把dex插入到dexElements的前面。

继续加载流程，首先调用TinkerDexLoader的checkComplete校验dex_meta.xml文件中记载的dex补丁文件和经过opt优化过的文件是否存在，然后调用loadTinkerJars加载补丁dex。

```java
    /**
     * Load tinker JARs and add them to
     * the Application ClassLoader.
     *
     * @param application The application.
     */
    @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    public static boolean loadTinkerJars(Application application, boolean tinkerLoadVerifyFlag, String directory, Intent intentResult) {

        PathClassLoader classLoader = (PathClassLoader) TinkerDexLoader.class.getClassLoader();

        String dexPath = directory + "/" + DEX_PATH + "/";
        File optimizeDir = new File(directory + "/" + DEX_OPTIMIZE_PATH);

        ArrayList<File> legalFiles = new ArrayList<>();

        final boolean isArtPlatForm = ShareTinkerInternals.isVmArt();
        for (ShareDexDiffPatchInfo info : dexList) {
            //for dalvik, ignore art support dex
            if (isJustArtSupportDex(info)) {
                continue;
            }
            String path = dexPath + info.realName;
            File file = new File(path);

            if (tinkerLoadVerifyFlag) {
                long start = System.currentTimeMillis();
                String checkMd5 = isArtPlatForm ? info.destMd5InArt : info.destMd5InDvm;
                if (!SharePatchFileUtil.verifyDexFileMd5(file, checkMd5)) {
                    //it is good to delete the mismatch file
                    ...
                    return false;
                }
            }
            legalFiles.add(file);
        }
        try {
            SystemClassLoaderAdder.installDexes(application, classLoader, optimizeDir, legalFiles);
        } catch (Throwable e) {
            Log.e(TAG, "install dexes failed");
//            e.printStackTrace();
            intentResult.putExtra(ShareIntentUtil.INTENT_PATCH_EXCEPTION, e);
            ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_DEX_LOAD_EXCEPTION);
            return false;
        }
        Log.i(TAG, "after loaded classloader: " + application.getClassLoader().toString());

        return true;
    }
```

根据传过来的tinkerLoadVerifyFlag选项控制是否每次加载都要验证dex的md5值，一般来说不需要，默认也是false，会节省加载时间。然后调用SystemClassLoaderAdder去加载。

```java
//SystemClassLoaderAdder
    public static void installDexes(Application application, PathClassLoader loader, File dexOptDir, List<File> files)
        throws Throwable {

        if (!files.isEmpty()) {
            ClassLoader classLoader = loader;
            if (Build.VERSION.SDK_INT >= 24) {
                classLoader = AndroidNClassLoader.inject(loader, application);
            }
            //because in dalvik, if inner class is not the same classloader with it wrapper class.
            //it won't fail at dex2opt
            if (Build.VERSION.SDK_INT >= 23) {
                V23.install(classLoader, files, dexOptDir);
            } else if (Build.VERSION.SDK_INT >= 19) {
                V19.install(classLoader, files, dexOptDir);
            } else if (Build.VERSION.SDK_INT >= 14) {
                V14.install(classLoader, files, dexOptDir);
            } else {
                V4.install(classLoader, files, dexOptDir);
            }

            if (!checkDexInstall()) {
                throw new TinkerRuntimeException(ShareConstants.CHECK_DEX_INSTALL_FAIL);
            }
        }
    }
```

看到这里，如果你之前看过Multidex.install()方法的实现，就会感觉很相似。只不过热修复是把dex插到dexElements的前面，Multidex是把其余的dex插到后面。相同的就是都是分版本加载，我们分别来看，由于v14以下(Android4.0以前)太过古老，我们就不看了，从v14开始。

### v14

14 <= SDK < 19
Android 4.0 <= Android系统 < Android 4.4

```java
    /**
     * Installer for platform versions 14, 15, 16, 17 and 18.
     */
    private static final class V14 {

        private static void install(ClassLoader loader, List<File> additionalClassPathEntries,
                                    File optimizedDirectory)
            throws IllegalArgumentException, IllegalAccessException,
            NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
            Field pathListField = ShareReflectUtil.findField(loader, "pathList");
            Object dexPathList = pathListField.get(loader);
            ShareReflectUtil.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList,
                new ArrayList<File>(additionalClassPathEntries), optimizedDirectory));
        }

        private static Object[] makeDexElements(
            Object dexPathList, ArrayList<File> files, File optimizedDirectory)
            throws IllegalAccessException, InvocationTargetException,
            NoSuchMethodException {
            Method makeDexElements =
                ShareReflectUtil.findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class);

            return (Object[]) makeDexElements.invoke(dexPathList, files, optimizedDirectory);
        }
    }
```

反射找到classLoder中的pathList，然后反射调用pathList中的makeDexElements方法，穿进去的参数分别是补丁dexList和优化过的opt目录，在Tinker中是dex补丁目录的同级目录``odex/``。

其中有个ShareReflectUtil.expandFieldArray我们看下实现

```java
    public static void expandFieldArray(Object instance, String fieldName, Object[] extraElements)
        throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
        Field jlrField = findField(instance, fieldName);

        Object[] original = (Object[]) jlrField.get(instance);
        Object[] combined = (Object[]) Array.newInstance(original.getClass().getComponentType(), original.length + extraElements.length);

        // NOTE: changed to copy extraElements first, for patch load first

        System.arraycopy(extraElements, 0, combined, 0, extraElements.length);
        System.arraycopy(original, 0, combined, extraElements.length, original.length);

        jlrField.set(instance, combined);
    }
```

注意传进来的值分别是pathList,"dexElements"和新生成的dexElements数组，找到pathList的原始oldDexElements，然后生成一个新的数组combined，长度是oldDexElements.length + newDexElements.length。然后将newDexElements拷贝到combined的前面，将oldDexElements拷贝的combined的剩余位置，我们称之为dex前置。

刚才我们说Tinker是将dex前置，Multidex是将dex后置，我们顺便看下Multidex.install()中expandFieldArray的实现吧。

```java
//Multidex.java
    private static void expandFieldArray(Object instance, String fieldName,
            Object[] extraElements) throws NoSuchFieldException, IllegalArgumentException,
            IllegalAccessException {
        Field jlrField = findField(instance, fieldName);
        Object[] original = (Object[]) jlrField.get(instance);
        Object[] combined = (Object[]) Array.newInstance(
                original.getClass().getComponentType(), original.length + extraElements.length);
        System.arraycopy(original, 0, combined, 0, original.length);
        System.arraycopy(extraElements, 0, combined, original.length, extraElements.length);
        jlrField.set(instance, combined);
    }
```

它是先把oldDexElements拷贝到了前面，在把newDexElements拷贝到了后面，我们称之为dex后置。

实际上，对于Multidex的项目，不论Tinker是否加载了补丁，都应该在ApplicationLike的onBaseContextAttached方法中执行``MultiDex.install(base);``。

### v19

19 <= SDK < 23
Android 4.4 <= Android系统 < Android 6.0

跟v14的区别不大，只是在makeDexElements方法中多加了一个参数suppressedExceptions异常数组，另外在makeDexElements的catch异常中多加了一次重试

```java
catch (NoSuchMethodException e) {
                Log.e(TAG, "NoSuchMethodException: makeDexElements(ArrayList,File,ArrayList) failure");
                try {
                    makeDexElements = ShareReflectUtil.findMethod(dexPathList, "makeDexElements", List.class, File.class, List.class);
                } catch (NoSuchMethodException e1) {
                    Log.e(TAG, "NoSuchMethodException: makeDexElements(List,File,List) failure");
                    throw e1;
                }
            }
```
是因为Tinker发现线上有的Rom将改方法参数类型给改了，本来是``makeDexElements(ArrayList,File,ArrayList)``，给改成了``makeDexElements(List,File,List)``，做了个兼容处理。

### v23

23 <= SDK < 24
Android 6.0 <= Android系统 < Android 7.0

Android6.0以后把makeDexElements给改了，改成了``makePathElements(List,File,List)``，如果找不到的话再找一下``makeDexElements(List,File,List)``。其余没啥区别。

### v24

SDK >=24
Android 系统 >= Android7.0

```java
    if (Build.VERSION.SDK_INT >= 24) {
        classLoader = AndroidNClassLoader.inject(loader, application);
    }
```

哎，这个好像跟上面不太一样啊，这是为啥呢。
[Android N混合编译与对热补丁影响解析](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=0)中详细解释了混合编译对热不定的影响。我做下简单的总结。

我们知道，在Dalvik虚拟机中，总是在运行时通过JIT（Just-In—Time）把字节码文件编译成机器码文件再执行，这样跑起来程序就很慢，所在ART上，改为AOT（Ahead-Of—Time）提前编译，即在安装应用或OTA系统升级时提前把字节码编译成机器码，这样就可以直接执行了，提高了运行效率。但是AOT有个缺点就是每次执行的时间都太长了，并且占用的ROM空间又很大，所以在Android N上Google做了混合编译同时支持JIT和AOT。混合编译的作用简单来说，在应用运行时分析运行过的代码以及“热代码”，并将配置存储下来。在设备空闲与充电时，ART仅仅编译这份配置中的“热代码”。

简单来说，就是在应用安装和首次运行不做AOT编译，先让用户愉快的玩耍起来，然后把在运行中JIT解释执行的那部分代码收集起来，在手机空闲的时候通过dex2aot编译生成一份名为app image的base.art文件，然后在下次启动的时候一次性把app image加载进来到缓存，预先加载代替用时查找以提升应用的性能。

这种方式对热补丁的影响就是，app image中已经存在的类会被插入到ClassLoader的ClassTable，再次加载类时，直接从ClassTable中取而不会走DefineClass。假设base.art文件在补丁前已经存在，这里存在三种情况：

 1. 补丁修改的类都不appimage中；这种情况是最理想的，此时补丁机制依然有效；
 2. 补丁修改的类部分在appimage中；这种情况我们只能更新一部分的类，此时是最危险的。一部分类是新的，一部分类是旧的，app可能会出现地址错乱而出现crash。
 3. 补丁修改的类全部在appimage中；这种情况只是造成补丁不生效，app并不会因此造成crash。
 
Tinker的解决方案是，完全废弃掉PathClassloader，而采用一个新建Classloader来加载后续的所有类，即可达到将cache无用化的效果。基本原理我们清楚了，让我们来看下代码吧。

```java
//AndroidNClassLoader.java
    public static AndroidNClassLoader inject(PathClassLoader originClassLoader, Application application) throws Exception {
        AndroidNClassLoader classLoader = createAndroidNClassLoader(originClassLoader);
        reflectPackageInfoClassloader(application, classLoader);
        return classLoader;
    }
    
        private static AndroidNClassLoader createAndroidNClassLoader(PathClassLoader original) throws Exception {
        //let all element ""
        AndroidNClassLoader androidNClassLoader = new AndroidNClassLoader("",  original);
        Field originPathList = findField(original, "pathList");
        Object originPathListObject = originPathList.get(original);
        //should reflect definingContext also
        Field originClassloader = findField(originPathListObject, "definingContext");
        originClassloader.set(originPathListObject, androidNClassLoader);
        //copy pathList
        Field pathListField = findField(androidNClassLoader, "pathList");
        //just use PathClassloader's pathList
        pathListField.set(androidNClassLoader, originPathListObject);
        return androidNClassLoader;
    }

```

我们按步骤进行：

 1. 新建一个AndroidNClassLoader 它的parent是originPathClassLoader。注意，PathClassLoader的optimizedDirectory只能是null，这个后面还有用。
 2. 找到originPathClassLoader中的pathList 和 pathList中的类型为ClassLoader的definingContext。
 3. 替换definingContext为AndroidNClassLoader
 4. 将AndroidNClassLoader中的pathList替换为originPathClassLoader的pathList。
 
有的同学可能会问，Android 的ClassLoader采用双亲委托模型，只有parent找不到的情况下才会去找AndroidNClassLoader，那我新建这个AndroidNClassLoader有什么用，最终还是会去originPathClassLoader中取找。其实不是这样的，我们已经将originPathClassLoader中pathList中的definingContext(是个ClassLoader)替换为了AndroidNClassLoader了。这个definingContext会在生成DexFile的时候传递进去，而ClassLoader的findClass()方法会调用pathList的findClass方法，如下：

```java
//DexPathList.java
    public Class findClass(String name) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        return null;
    }
```
最终还是调用的dexFile.loadClassBinaryName()方法，其中的第二个参数其实就已经是AndroidNClassLoader了。

还记得刚才说的AndroidNClassLder的optimizedDirectory是null吗

```java
//DexPathList.java
    private static Element[] makeDexElements(ArrayList<File> files,
            File optimizedDirectory) {
            ...
            dex = loadDexFile(file, optimizedDirectory);
            ....
    }
    
        private static DexFile loadDexFile(File file, File optimizedDirectory)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0);
        }
    }
```

看到这里我们明白了，optimizedDirectory是用来缓存我们需要加载的dex文件的，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile对象。意思也就是说我不需要用缓存，不需要用app image加载。

接续往下走

```java
        private static void reflectPackageInfoClassloader(Application application, ClassLoader reflectClassLoader) throws Exception {
        String defBase = "mBase";
        String defPackageInfo = "mPackageInfo";
        String defClassLoader = "mClassLoader";

        Context baseContext = (Context) findField(application, defBase).get(application);
        Object basePackageInfo = findField(baseContext, defPackageInfo).get(baseContext);
        Field classLoaderField = findField(basePackageInfo, defClassLoader);
        Thread.currentThread().setContextClassLoader(reflectClassLoader);
        classLoaderField.set(basePackageInfo, reflectClassLoader);
    }
```

作用是替换掉了mPackageInfo中的ClassLoader，mPackageInfo是LoadedApk的对象，代表了APK文件在内存中的表示，诸如Apk文件的代码和资源，甚至代码里面的Activity，Service等组件的信息我们都可以通过此对象获取。

```java
//ActivityThread.java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
StrictMode.incrementExpectedActivityCount(activity.getClass());
r.intent.setExtrasClassLoader(cl);
```

到这里就完成了AndroidNClassLoader的创建与替换，接下来的加载过程使用了v23的加载流程，就不细说了。

### 总结

至此，整个dex加载流程就分析完了。我们看到Tinker在兼容性上做了充足的工作，整个加载流程虽然跟其他基于Multidex的热补丁框架差不多，但是在兼容性上做了更完备的处理。

## 加载补丁资源

Tinker的资源更新采用的InstantRun的资源补丁方式，全量替换资源。由于App加载资源是依赖Context.getResources()方法返回的Resources对象，Resources 内部包装了 AssetManager，最终由 AssetManager 从 apk 文件中加载资源。我们要做的就是新建一个AssetManager()，hook掉其中的addAssetPath()方法，将我们的资源补丁目录传递进去，然后循环替换Resources对象中的AssetManager对象，达到资源替换的目的。看下代码实现。

首先依然先根据res_meta.xml文件中记载的信息检查文件(res/resources.apk)是否存在，实现在TinkerResourceLoader.checkComplete()方法，然后调用``TinkerResourcePatcher.isResourceCanPatch(context);``判断是否支持反射更新资源，看下具体的实现

```java
    public static void isResourceCanPatch(Context context) throws Throwable {
        // Create a new AssetManager instance and point it to the resources installed under /sdcard
        AssetManager assets = context.getAssets();
        // Baidu os
        if (assets.getClass().getName().equals("android.content.res.BaiduAssetManager")) {
            Class baiduAssetManager = Class.forName("android.content.res.BaiduAssetManager");
            newAssetManager = (AssetManager) baiduAssetManager.getConstructor().newInstance();
        } else {
            newAssetManager = AssetManager.class.getConstructor().newInstance();
        }
        addAssetPathMethod = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
        addAssetPathMethod.setAccessible(true);

        // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
        // in L, so we do it unconditionally.
        ensureStringBlocksMethod = AssetManager.class.getDeclaredMethod("ensureStringBlocks");
        ensureStringBlocksMethod.setAccessible(true);

        // Iterate over all known Resources objects
        if (SDK_INT >= KITKAT) {
            //pre-N
            // Find the singleton instance of ResourcesManager
            Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
            Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance");
            mGetInstance.setAccessible(true);
            Object resourcesManager = mGetInstance.invoke(null);
            try {
                Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
                fMActiveResources.setAccessible(true);
                ArrayMap<?, WeakReference<Resources>> arrayMap =
                    (ArrayMap<?, WeakReference<Resources>>) fMActiveResources.get(resourcesManager);
                references = arrayMap.values();
            } catch (NoSuchFieldException ignore) {
                // N moved the resources to mResourceReferences
                Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
                mResourceReferences.setAccessible(true);
                //noinspection unchecked
                references = (Collection<WeakReference<Resources>>) mResourceReferences.get(resourcesManager);
            }
        } else {
            Class<?> activityThread = Class.forName("android.app.ActivityThread");
            Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
            fMActiveResources.setAccessible(true);
            Object thread = getActivityThread(context, activityThread);
            @SuppressWarnings("unchecked")
            HashMap<?, WeakReference<Resources>> map =
                (HashMap<?, WeakReference<Resources>>) fMActiveResources.get(thread);
            references = map.values();
        }
        // check resource
        if (references == null || references.isEmpty()) {
            throw new IllegalStateException("resource references is null or empty");
        }
        try {
            assetsFiled = Resources.class.getDeclaredField("mAssets");
            assetsFiled.setAccessible(true);
        } catch (Throwable ignore) {
            // N moved the mAssets inside an mResourcesImpl field
            resourcesImplFiled = Resources.class.getDeclaredField("mResourcesImpl");
            resourcesImplFiled.setAccessible(true);
        }
    }
```

按照步骤来吧，首先新建一个AssetManager对象，其中对BaiduROM做了兼容(BaiduAssetManager)，拿到其中的addAssetPath方法的反射addAssetPathMethod，然后拿到ensureStringBlocks的反射，然后区分版本拿到Resources的集合。

 - SDK >= 19，从ResourcesManager中拿到mActiveResources变量，是个持有Resources的ArrayMap，赋值给references，Android N中该变量叫做mResourceReferences
 - SDK < 19，从ActivityThread中获取mActiveResources，是个HashMap持有Resources，赋值给references

如果references为空，说明该系统不支持资源补丁，throw 一个IllegalStateException被上层调用catch。

然后调用monkeyPatchExistingResources方法(这个方法的名字跟InstantRun的资源补丁方法名是一样的)，将补丁资源路径(res/resources.apk)传递进去，代码就不贴了，简单描述为反射调用新建的AssetManager的addAssetPath将路径穿进去，然后主动调用ensureStringBlocks方法确保资源的字符串索引创建出来；然后循环遍历持有Resources对象的references集合，依次替换其中的AssetManager为新建的AssetManager，最后调用Resources.updateConfiguration将Resources对象的配置信息更新到最新状态，完成整个资源替换的过程。
 
目前来看InstantRun的资源更新方式最简便而且兼容性也最好，市面上大多数的热补丁框架都采用这套方案。Tinker的这套方案虽然也采用全量的替换，但是在下发patch中依然采用差量资源的方式获取差分包，下发到手机后再合成全量的资源文件，有效的控制了补丁文件的大小。

## 加载补丁so

依然根据so_meta.txt中的补丁信息校验so文件是否都存在。然后将so补丁列表存放在结果中libs的字段。

so的更新方式跟dex和资源都不太一样，因为系统提供给了开发者自定义so目录的选项

```java
public final class System {
    ...
    public static void load(String pathName) {
        Runtime.getRuntime().load(pathName, VMStack.getCallingClassLoader());
    }
    ...
}
```

Tinker加载SO补丁提供了两个入口，分别是TinkerInstaller和TinkerApplicationHelper。他们两个的区别是TinkerInstaller只有在Tinker.install过之后才能使用,否则会抛出异常。

```java
//TinkerInstaller
    public static boolean loadLibraryFromTinker(Context context, String relativePath, String libname) throws UnsatisfiedLinkError {
        final Tinker tinker = Tinker.with(context);

        libname = libname.startsWith("lib") ? libname : "lib" + libname;
        libname = libname.endsWith(".so") ? libname : libname + ".so";
        String relativeLibPath = relativePath + "/" + libname;

        //TODO we should add cpu abi, and the real path later
        if (tinker.isEnabledForNativeLib() && tinker.isTinkerLoaded()) {
            TinkerLoadResult loadResult = tinker.getTinkerLoadResultIfPresent();
            if (loadResult.libs != null) {
                for (String name : loadResult.libs.keySet()) {
                    if (name.equals(relativeLibPath)) {
                        String patchLibraryPath = loadResult.libraryDirectory + "/" + name;
                        File library = new File(patchLibraryPath);
                        if (library.exists()) {
                            //whether we check md5 when load
                            boolean verifyMd5 = tinker.isTinkerLoadVerify();
                            if (verifyMd5 && !SharePatchFileUtil.verifyFileMd5(library, loadResult.libs.get(name))) {
                                tinker.getLoadReporter().onLoadFileMd5Mismatch(library, ShareConstants.TYPE_LIBRARY);
                            } else {
                                System.load(patchLibraryPath);
                                TinkerLog.i(TAG, "loadLibraryFromTinker success:" + patchLibraryPath);
                                return true;
                            }
                        }
                    }
                }
            }
        }

        return false;
    }
```

简单来说就是遍历检查的结果列表libs，找到要加载的类，调用System.load方法进行加载。

# 遇到的问题

在集成Tinker的过程中，遇到了一个问题(环境是Dalvik，ART没问题)，在前面我们提到了dex.loader的配置，我把项目中用于下载补丁文件的工具类A加到了其中，然后下发补丁报错，出现Class ref in pre-verified class resolved to unexpected  implementation的crash。Qzone的那套热补丁为了消除这个错误采用插庄的方式来规避，Tinker采用全量dex的方式来规避该问题，那为什么还会出现呢。

根据log找到了报错点是在工具类A中的一个直接引用类B的方法中报错。错误原因在加载补丁dex一节其实已经提到一些，我们引用过来，这个配置(dex.loader)中的类不会出现在任何全量补丁dex里，也就是说在合成后，这些类还在老的dex文件中，比如在补丁前dex顺序是这样的：``oldDex1 -> oldDex2 -> oldDex3..``，那么假如修改了dex1中的文件，那么补丁顺序是这样的``newDex1 -> oldDex1 -> oldDex2...``其中合成后的newDex1中的类是oldDex1中除了dex.loader中标明的类之外的所有类，dex.loader中的类依然在oldDex1中。
也就是说A类是在dex.loader配置中的，补丁后，A依然在oldDex1中，而A的直接引用类B却出现在了newDex1中，并且在之前A类已经被打上了preverify标志，所在A再去newDex1中加载B的话就会报该错误。

那有的同学可能会问了，TinkerApplication也在oldDex1中的，而我们的ApplicationLike在补丁后也出现在了newDex1中，TinkerApplication反射调用ApplicationLike的生命周期方法为什么没有出现crash呢？还记得文章前面的有一个反射么，我们说了要注意后面会讲到，就是在这里用到的。

校验preverify的方法，正常的类加载会走到这里。

```java
ClassObject* dvmResolveClass(const ClassObject* referrer, u4 classIdx,
    bool fromUnverifiedConstant)
{
....
       if (!fromUnverifiedConstant &&
            IS_CLASS_FLAG_SET(referrer, CLASS_ISPREVERIFIED))
...
}
```

而反射走了完全不同的路径，不会走到dvmResolveClass方法，也就不会报错了。关于这个方法，我们下篇文章会详细讲解。反射最直接的目的也是为了隔离开这两个类，也就是隔离开了Tinker组件和app。如图

![](http://7xs23g.com1.z0.glb.clouddn.com/ref.png)

通过反射，将Tinker组建和App隔离开，并且先后顺序是先Tinker后App，这样可以防止App中的代码提前加载，确保App中所有的代码都可以具有被热修复的能力包括ApplicationLike。

然后又有同学问了，为啥Dalvik有问题，ART没问题呢？那是因为在ART虚拟机原生支持从APK文件加载多个dex文件。在应用安装时执行dex2oat扫描 classes(..N).dex文件，并将它们编译成单个oat文件，供 Android设备执，也就不存在MultiDex的问题了。

这个问题的[issue](https://github.com/Tencent/tinker/issues/124)

# 总结

到这里，Tinker的基本补丁加载流程就分析完了，本文只对补丁加载流程加以分析，对dexDiff差分以及补丁加载没有做说明，如果你对这部分感兴趣可以参考这篇文章[Tinker Dexdiff算法解析](https://www.zybuluo.com/dodola/note/554061)。另外ART下的内联影响和OTA升级没有做过多说明，Tinker官方已经有相关文章。

我们简单对Tinker做下总结。
优点：

 - 支持类、资源、so修复
 - 兼容性处理的很好，全平台支持
 - 由于不用插庄，所以性能损耗很小
 - 完善的开发文档和官方技术支持
 - gradle支持，再自己定义下可以一键打补丁包
 - dexDiff算法使得补丁文件较小
 - 扩展性良好，代码中处处为开发者留出开放接口，简直业界良心
 - 支持多次补丁

缺点：

 - 不支持及时生效，下发补丁需要重启生效，MultiDex方案决定的
 - 占用ROM空间较大，这点空间在如今的手机大ROM下也不算个事
 - 对加固支持不太好

总结下来Tinker是一种基于单ClassLoader加载多dex方案的热补丁框架，兼容性做的比较好，功能强大。如果你正在考虑接入热补丁，那么强烈推荐你使用Tinker，地精修补匠，带你无限刷新！

# 参考

[微信Tinker的一切都在这里，包括源码(一)](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286384&idx=1&sn=f1aff31d6a567674759be476bcd12549&scene=4#wechat_redirect)
[Android N混合编译与对热补丁影响解析](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=0)
[Tinker Dexdiff算法解析](https://www.zybuluo.com/dodola/note/554061)
[从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)
[Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)
[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)

