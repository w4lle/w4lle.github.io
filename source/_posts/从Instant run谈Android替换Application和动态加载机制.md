---
title: 从Instant run谈Android替换Application和动态加载机制
date: 2016-05-02 19:33:05
tags: [Android, 热补丁]
---

# 背景

``Android studio 2.0 Stable``版本中集成了``Install run``即时编译技术，官方描述可以大幅加速编译速度，我们团队在第一时间更新并使用，总体用下来感觉，恩...也就那样吧，还不如不用的快。所以就去看了下``Install run``的实现方式，其中有一个整体框架的基础，也就是今天的文章的主题，Android替换Application和动态加载机制。

# Instant run

``Instant run``的大概实现原理可以看下这篇[Instant Run 浅析](http://jiajixin.cn/2015/11/25/instant-run/)，我们需要知道``Instant run``使用的``gradle plugin2.0.0``，源码在[这里](https://bintray.com/android/android-tools/com.android.tools.build.gradle/view)，文中大概讲了下``Instant run``的实现原理，但是并没有深入细节，特别是替换Application和动态加载机制。

关于动态加载，实际上``Instant run``提供了两种动态加载的机制：
1.修改java代码需要重启应用加载补丁dex，而在Application初始化时替换了Application，新建了一个自定义的ClassLoader去加载所有的dex文件。我们称为**重启更新机制**
2.修改代码不需要重启，新建一个``ClassLoader``去加载修改部分。我们称为**热更新机制**

## Application入口

在编译时``Instant run``用到了``Transform API``修改字节码文件。其中``AndroidManifest.xml``文件也被修改，如下：
``/app/build/intermediates/bundles/production/instant-run/AndroidManifest.xml``，其中的``Application``标签
```xml
<application
        name="com.aa.bb.MyApplication"
        android:name="com.android.tools.fd.runtime.BootstrapApplication"
		... />
```
多了一个``com.android.tools.fd.runtime.BootstrapApplication``，在刚刚提到的``gradle plugin``中的``instant-run-server``目录下找到该文件。

实际上``BootstrapApplication``是我们app的实际入口，我们自己的``Application``即``MyApplication``采用反射机制调用。
我们知道``Application``是``ContextWrapper``的子类

```java
// android.app.Application
public class Application extends ContextWrapper {
    // ...
    public application() {
        super(null);
    }
    // ...
}
// android.content.ContextWrapper
public class ContextWrapper extends Context {
    Context mBase;
    // ...
    public ContextWrapper(Context base) {
        mBase = base;
    }
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    // ...
    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }
    @Override
    public Resources getResources()
    {
        return mBase.getResources();
    }
    // ...
}
```

ContextWrapper一方面继承了Context，一方面又包含(composite)了一个Context对象（称为mBase），对Context的实现为转发给mBase对象处理。上面的代码表示，在``attachBaseContext``方式调用之前Application是没有用的，因为mBase是空的。所以我们看下``BootstrapApplication``的``attachBaseContext``方法

```java
protected void attachBaseContext(Context context) {
        if (!AppInfo.usingApkSplits) {
            createResources(apkModified);
            //新建一个ClassLoader并设置为原ClassLoader的parent
            setupClassLoaders(context, context.getCacheDir().getPath(), apkModified);
        }
		//通过Manifest中我们的实际Application即MyApplication名反射生成对象
        createRealApplication();
		//调用attachBaseContext完成初始化
        super.attachBaseContext(context);

        if (realApplication != null) {
        //反射调用实际Application的attachBaseContext方法
            try {
                Method attachBaseContext =
                        ContextWrapper.class.getDeclaredMethod("attachBaseContext", Context.class);
                attachBaseContext.setAccessible(true);
                attachBaseContext.invoke(realApplication, context);
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        }
    }
```

初始化ClassLoader

```java
//BootstrapApplication.setupClassLoaders
private static void setupClassLoaders(Context context, String codeCacheDir, long apkModified) {
		// /data/data/package_name/files/instant-run/dex/目录下的dex列表
        List<String> dexList = FileManager.getDexList(context, apkModified);
            ClassLoader classLoader = BootstrapApplication.class.getClassLoader();
            String nativeLibraryPath = (String) classLoader.getClass().getMethod("getLdLibraryPath")
                                .invoke(classLoader);
            IncrementalClassLoader.inject(
                    classLoader,
                    nativeLibraryPath,
                    codeCacheDir,
                    dexList);
        }
    }
    
//IncrementalClassLoader.inject
public static ClassLoader inject(
            ClassLoader classLoader, String nativeLibraryPath, String codeCacheDir,
            List<String> dexes) {
        //新建一个自定义ClassLoader，dexPath为参数中的dexList
        IncrementalClassLoader incrementalClassLoader =
                new IncrementalClassLoader(classLoader, nativeLibraryPath, codeCacheDir, dexes);
        //设置为原ClassLoader的parent
        setParent(classLoader, incrementalClassLoader);
		return incrementalClassLoader;
    }
```

## 动态加载

新建一个自定义的``ClassLoader``名为IncrementalClassLoader，该``ClassLoader``很简单，就是``BaseDexClassLoader``的一个子类，并且将``IncrementalClassLoader``设置为原ClassLoader的parent，熟悉JVM加载机制的同学应该都知道，由于ClassLoader采用双亲委托模式，即委托父类加载类，父类找不到再自己去找。这样``IncrementalClassLoader``就变成了整个App的所有类的加载的ClassLoader，并且dexPath是``/data/data/package_name/files/instant-run/dex``目录下的dex列表，这意味着什么呢？

```java

//``BaseDexClassLoader``的``findClass``
protected Class<?> findClass(String name) throws ClassNotFoundException {
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
```

可以看到，查找Class的任务通过pathList完成；这个pathList是一个DexPathList类的对象，它的findClass方法如下：

```java
public Class findClass(String name, List<Throwable> suppressed) {
   for (Element element : dexElements) {
       DexFile dex = element.dexFile;

       if (dex != null) {
           Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
           if (clazz != null) {
               return clazz;
           }
       }
   }
   if (dexElementsSuppressedExceptions != null) {
       suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
   }
   return null;
}
```

这个DexPathList内部有一个叫做dexElements的数组，然后findClass的时候会遍历这个数组来查找Class。看到了吗，这个dexElements就是从dexPath来的，也就说是``IncrementalClassLoader``用来加载dexPath(/data/data/package_name/files/instant-run/dex/)下面的dex文件。感兴趣的同学可以看下，我们app中的所有第三方库和自己项目中的代码，都被打包成若干个slice dex分片，该目录下有几十个dex文件。每当修改代码用``Instant run``完成编译，该目录下的dex文件就会有一个或者几个的更新时间发生改变。

正常情况下，apk被安装之后，APK文件的代码以及资源会被系统存放在固定的目录（比如/data/app/package_name/base-1.apk )系统在进行类加载的时候，会自动去这一个或者几个特定的路径来寻找这个类。而使用``Install run``则完全不管之前的加载路径，所有的分片dex文件和资源都在dexPath下，用``IncrementalClassLoader``去加载。也就是加载不存在APK固定路径之外的类，即动态加载。

但是仅仅有ClassLoader是不够的。因为每个被修改的类都被改了名字，类名在原名后面添加``$override``，目录在``app/build/intermediates/transforms/instantRun/debug/folders/4000``。AndroidManifest中并没有注册这些被改了名字的Activity。> 因此正常情况下系统无法加载我们插件中的类；因此也没有办法创建Activity的对象。
> 解决这个问题有两个思路，要么全盘接管这个类加载的过程；要么告知系统我们使用的插件存在于哪里，让系统帮忙加载；这两种方式或多或少都需要干预这个类加载的过程。
> 引用自 -- [Android 插件化原理解析——插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)



## 动态加载的两种方案

先来看下系统如何完成类的加载过程。
``Activity``的创建过程

```java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
StrictMode.incrementExpectedActivityCount(activity.getClass());
r.intent.setExtrasClassLoader(cl);
```

通过``ClassLoader``和类名加载，反射调用生成``Activity``对象，其中的``ClassLoader``从``LoadedApk``的一个对象``r.packageInfo``中获得的。``LoadedApk``对象是APK文件在内存中的表示。 Apk文件的相关信息，诸如Apk文件的代码和资源，甚至代码里面的``Activity``，``Service``等组件的信息我们都可以通过此对象获取。
r.packageInfo的来源：

```java
private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
        ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
        boolean registerPackage) {
        // 获取userid信息
    final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
    synchronized (mResourcesManager) {
    // 尝试获取缓存信息
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            // Caching not supported across users
            ref = null;
        } else if (includeCode) {
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
                // 缓存没有命中，直接new
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

        // 省略。。更新缓存
        return packageInfo;
    }
}
```

重要的是这个缓存``mPackage``，``LoadedApk``对象``packageInfo``就是从这个缓存中取的，所以我们只要在``mPackage``修改里面的``ClassLoader``控制类的加载就能完成动态加载。

在《[Android 插件化原理解析——插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)》一文中，作者已经提出两种动态加载的解决方案：

> 『激进方案』中我们自定义了插件的ClassLoader，并且绕开了Framework的检测；利用ActivityThread对于LoadedApk的缓存机制，我们把携带这个自定义的ClassLoader的插件信息添加进mPackages中，进而完成了类的加载过程。

>『保守方案』中我们深入探究了系统使用ClassLoader findClass的过程，发现应用程序使用的非系统类都是通过同一个PathClassLoader加载的；而这个类的最终父类BaseDexClassLoader通过DexPathList完成类的查找过程；我们hack了这个查找过程，从而完成了插件类的加载。

激进方案由于是一个插件一个``Classloader``也叫多``ClassLoader``方案，代表作[DroidPlugin](https://github.com/Qihoo360/DroidPlugin)；保守方案也叫做单``ClassLoader``方案，代表作，Small、众多热更新框架如[nuwa](https://github.com/jasonross/Nuwa)等。

## Instant run的重启更新机制

绕了一大圈，终于能接着往下看了。接上面，我们继续看``BootstrapApplication``的``onCreate``方法

```java
public void onCreate() {
        MonkeyPatcher.monkeyPatchApplication(
                    BootstrapApplication.this, BootstrapApplication.this,
                    realApplication, externalResourcePath);
            MonkeyPatcher.monkeyPatchExistingResources(BootstrapApplication.this,
                    externalResourcePath, null);
        super.onCreate();
        ...
		//手机客户端app和Android Studio建立Socket通信，AS是客户端发消息，app		//是服务端接收消息作出相应操作。Instant run的通信方式。不在本文范围内
        Server.create(AppInfo.applicationId, BootstrapApplication.this);

        if (realApplication != null) {
        	//还记得这个realApplication吗，我们app中实际的Application
            realApplication.onCreate();
        }
    }
```

上面代码，手机客户端app和Android Studio建立Socket通信，AS是客户端发消息，app是服务端接收消息作出相应操作，这是Instant run的通信方式，不在本文范围内。然后反射调用实际``Application``的``onCreate``方法。
那么前面的两个``MonkeyPatcher``的方法是干嘛的呢

先看``MonkeyPatcher.monkeyPatchApplication``

```java
public static void monkeyPatchApplication(@Nullable Context context,
                                              @Nullable Application bootstrap,
                                              @Nullable Application realApplication,
                                              @Nullable String externalResourceFile) {
        try {
            // Find the ActivityThread instance for the current thread
            Class<?> activityThread = Class.forName("android.app.ActivityThread");
            Object currentActivityThread = getActivityThread(context, activityThread);

            // Find the mInitialApplication field of the ActivityThread to the real application
            Field mInitialApplication = activityThread.getDeclaredField("mInitialApplication");
            mInitialApplication.setAccessible(true);
            Application initialApplication = (Application) mInitialApplication.get(currentActivityThread);
            if (realApplication != null && initialApplication == bootstrap) {
            //**2.替换掉ActivityThread.mInitialApplication**
                mInitialApplication.set(currentActivityThread, realApplication);
            }

            // Replace all instance of the stub application in ActivityThread#mAllApplications with the
            // real one
            if (realApplication != null) {
                Field mAllApplications = activityThread.getDeclaredField("mAllApplications");
                mAllApplications.setAccessible(true);
                List<Application> allApplications = (List<Application>) mAllApplications
                        .get(currentActivityThread);
                for (int i = 0; i < allApplications.size(); i++) {
                    if (allApplications.get(i) == bootstrap) {
                    //**1.替换掉ActivityThread.mAllApplications**
                        allApplications.set(i, realApplication);
                    }
                }
            }

            // Figure out how loaded APKs are stored.

            // API version 8 has PackageInfo, 10 has LoadedApk. 9, I don't know.
            Class<?> loadedApkClass;
            try {
                loadedApkClass = Class.forName("android.app.LoadedApk");
            } catch (ClassNotFoundException e) {
                loadedApkClass = Class.forName("android.app.ActivityThread$PackageInfo");
            }
            Field mApplication = loadedApkClass.getDeclaredField("mApplication");
            mApplication.setAccessible(true);
            Field mResDir = loadedApkClass.getDeclaredField("mResDir");
            mResDir.setAccessible(true);

            // 10 doesn't have this field, 14 does. Fortunately, there are not many Honeycomb devices
            // floating around.
            Field mLoadedApk = null;
            try {
                mLoadedApk = Application.class.getDeclaredField("mLoadedApk");
            } catch (NoSuchFieldException e) {
                // According to testing, it's okay to ignore this.
            }

            // Enumerate all LoadedApk (or PackageInfo) fields in ActivityThread#mPackages and
            // ActivityThread#mResourcePackages and do two things:
            //   - Replace the Application instance in its mApplication field with the real one
            //   - Replace mResDir to point to the external resource file instead of the .apk. This is
            //     used as the asset path for new Resources objects.
            //   - Set Application#mLoadedApk to the found LoadedApk instance
            for (String fieldName : new String[]{"mPackages", "mResourcePackages"}) {
                Field field = activityThread.getDeclaredField(fieldName);
                field.setAccessible(true);
                Object value = field.get(currentActivityThread);

                for (Map.Entry<String, WeakReference<?>> entry :
                        ((Map<String, WeakReference<?>>) value).entrySet()) {
                    Object loadedApk = entry.getValue().get();
                    if (loadedApk == null) {
                        continue;
                    }

                    if (mApplication.get(loadedApk) == bootstrap) {
                        if (realApplication != null) {
                        //**3.替换掉mApplication**
                            mApplication.set(loadedApk, realApplication);
                        }
                        if (externalResourceFile != null) {
                        //替换掉资源目录
                            mResDir.set(loadedApk, externalResourceFile);
                        }

                        if (realApplication != null && mLoadedApk != null) {
                        //**4.替换掉mLoadedApk**
                            mLoadedApk.set(realApplication, loadedApk);
                        }
                    }
                }
            }
        } catch (Throwable e) {
            throw new IllegalStateException(e);
        }
    }
```

这里做了三件事情：

1.替换Application对象

``BootstrapApplication``的作用就是加载``realApplication``也就是``MyApplication``，所以我们就要把所有Framework层的``BootstrapApplication``对象替换为``MyApplication``对象。包括：

```java
baseContext.mPackageInfo.mApplication 代码3处
baseContext.mPackageInfo.mActivityThread.mInitialApplication 代码2处
baseContext.mPackageInfo.mActivityThread.mAllApplications 代码1处
```

2.替换资源相关对象mResDir，前面我们已经说过，正常情况下寻找资源都是在``/data/app/package_name/base-1.apk``目录下，而``Instant run``将资源也抽出来放在``/data/data/package_name/files/instant-run/``，加载目录也更改为后者

3.替换``mLoadedApk``对象
还记得前面的讲的``LoadedApk``吗，这里面有加载类的``ClassLoader``，由于``BootstrapApplication``在``attachBaseContext``方法中就将其已经替换为了``IncrementalClassLoader``，所以代码4处反射将``BootstrapApplication``的``mLoadedApk``赋值给了``MyApplication``，那么接下来MyApplication的所有类的加载都将由``IncrementalClassLoader``来负责。

``MonkeyPatcher.monkeyPatchExistingResources``更新资源补丁，不在本文范围内就不讲了。

这些工作做完之后调用``MyApplication``的``onCreate``方法``BootstrapApplication``就将控制权交给了``MyApplication``，这样在整个运行环境中，``MyApplication``就是正牌``Application``了，完成``Application``的替换。

总结一下，刚才我们说了已经有两个动态加载的方案，激进方案和保守方案,而``Instant run``的重启更新机制更像后者--保守方案即单``ClassLoader``方案，首先，该种方案只有一个``ClassLoader``，只不过是通过替换``Application``达到的替换``mLoadedApk``进而替换``ClassLoader``的目的，并没有涉及到缓存``mPackage``然后dexList也是它自己维护的。

## Instant run 热更新机制

Instant run哪里用到的热更新机制呢？还记得刚才我们提到的Socket通信吗，其中S端也就是手机客户端，接收到热更新的消息会执行下面的方法：

```java
 private int handleHotSwapPatch(int updateMode, @NonNull ApplicationPatch patch) {
        try {
            String dexFile = FileManager.writeTempDexFile(patch.getBytes());
            String nativeLibraryPath = FileManager.getNativeLibraryFolder().getPath();
            //新建一个ClassLoader，dexFile是刚更新的插件
            DexClassLoader dexClassLoader = new DexClassLoader(dexFile,
                    mApplication.getCacheDir().getPath(), nativeLibraryPath,
                    getClass().getClassLoader());

            // we should transform this process with an interface/impl
            Class<?> aClass = Class.forName(
                    "com.android.tools.fd.runtime.AppPatchesLoaderImpl", true, dexClassLoader);
            try {
                PatchesLoader loader = (PatchesLoader) aClass.newInstance();
                String[] getPatchedClasses = (String[]) aClass
                        .getDeclaredMethod("getPatchedClasses").invoke(loader);
                //loader是PatchesLoader的一个实例，调用load方法加载插件
                if (!loader.load()) {
                    updateMode = UPDATE_MODE_COLD_SWAP;
                }
            } catch (Exception e) {
                updateMode = UPDATE_MODE_COLD_SWAP;
            }
        } catch (Throwable e) {
            updateMode = UPDATE_MODE_COLD_SWAP;
        }
        return updateMode;
    }
```

可以看到根据单个dexFile新建了一个``ClassLoader``，然后调用``loader.load()``方法，``loader``是``PatchesLoader``接口的实例，``PatchesLoader``接口的一个实现类``AppPatchesLoaderImpl``，该类中记录了哪些修改的类。看一下``load``方法

```java
@Override
    public boolean load() {
        try {
        //遍历已记录的所有修改的类
            for (String className : getPatchedClasses()) {
                ClassLoader cl = getClass().getClassLoader();
                //我们刚才说的修改的类名后面都有$override
                Class<?> aClass = cl.loadClass(className + "$override");
                Object o = aClass.newInstance();
                //1.**反射修改原类中的$change字段为修改后的值**
                Class<?> originalClass = cl.loadClass(className);
                Field changeField = originalClass.getDeclaredField("$change");
                // force the field accessibility as the class might not be "visible"
                // from this package.
                changeField.setAccessible(true);
                // If there was a previous change set, mark it as obsolete:
                Object previous = changeField.get(null);
                if (previous != null) {
                    Field isObsolete = previous.getClass().getDeclaredField("$obsolete");
                    if (isObsolete != null) {
                        isObsolete.set(null, true);
                    }
                }
                changeField.set(null, o);
            }
        } catch (Exception e) {
            return false;
        }
        return true;
    }
```

``Instant run``的热更新原理可以概述为：
1.第一次运行，应用``transform API``修改字节码。
输出目录在``app/build/intermediates/transforms/instantRun/debug/folders/1/``，给所有的类添加``$change``字段，``$change``为``IncrementalChange``类型，``IncrementalChange``是个接口。如果``$change``不为空，去调用``$change``的``access$dispatch``方法，参数为方法签名字符串和方法参数数组，否则调用原逻辑。
load方法中会去加载全部补丁类，并赋值给对应原类的``$change``。
这也验证了我们说它是多``ClassLoader``方案。

2.所有修改的类有``gradle plugin``自动生成，类名在原名后面添加$override，复制修改后类的大部分方法，实现IncrementalChange 接口的access$dispatch方法，该方法会根据传递过来的方法签名，调用本类的同名方法。

那么也就是说只要把原类的``$change``字段设置为该类，那就会调用该类的``access$dispatch``方法，就会使用修改后的方法了。上面代码1处就通过反射修改了原类中的``$change``为修改后补丁类中的值。``AppPatchesLoaderImpl``记录了所有被修改的类，也会被打进补丁dex。

总结一下，可以看到``Instant run``热更新是多``ClassLoader``加载方案，每个插件dex都有一个``ClassLoader``，如果插件需要升级，直接重新创建一个自定的``ClassLoader``加载新的插件。但是目前来看，``Instant run``修改java代码大部分情况下都是重启更新机制，可能热更新机制还有bug。资源更新是热更新，重启对应Activity就可以。

# 总结

``Instant run``看下来真的有好多东西，其中就以替换``Application``和动态加载尤为重要，关于动态加载，完全可以根据``Instant run``的实现方式完成一个热修复和重启修复相结合的更新框架，用于线上bug的修复和功能更新，并且可以支持资源文件的更新，是无侵入性的更新框架，最重要的一点，这是官方支持的。但是，性能肯定会有所影响，实际开发中使用``Instant run``编译其实还有很多的问题，而且app初始化时使用的很多反射，这也直接导致app的启动速度降低好多。

另外一点关于Application的替换是基于[bazel](https://github.com/bazelbuild/bazel)(一种构建工具，类似于burk)中的[StubApplication](https://github.com/bazelbuild/bazel/blob/master/src/tools/android/java/com/google/devtools/build/android/incrementaldeployment/StubApplication.java)


# 参考

* [Android的Proxy/Delegate Application框架](http://blogs.360.cn/blog/proxydelegate-application/)
* [Android 插件化原理解析——插件加载机制](http://weishu.me/2016/04/05/understand-plugin-framework-classloader/)
* [Instant Run 浅析](http://jiajixin.cn/2015/11/25/instant-run/)
* [Instant Run原理解析](http://www.jianshu.com/p/0400fb58d086)
