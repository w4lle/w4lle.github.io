---
title: 一键接入Tinker
date: 2017-01-05 14:15:49
tags: [Android, 热补丁]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/tinker2.jpeg
---

# 背景

Tinker开源挺长时间了，使用的开发者也越来越多，对于一些小白开发者来说对接Tinker的成本还是挺高的，其中主要因素还是不能理解为什么Application要修改成ApplicationLike，以及改造后对项目中使用Application的地方也要同步修改。

在上篇文章[Android热补丁之Tinker原理解析](http://w4lle.github.io/2016/12/16/tinker/)中我们已经讲解了这样做的目的以及Tinker的加载补丁的流程，本篇文章主要讲一下一键接入Tinker的实现思路。

# InstantRun

我们的目的是要实现不修改Application达到替换Application的效果，在这篇文章[从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)中，详细讲述了如何动态替换Application，总结起来就两步：

 1. 打包时替换Application标签，插入BootstrapApplication
 2. 运行时hook系统api，将BootstrapApplication换回MyApplication

那么，我们依然可以用这套方案来实现Tinker的一键接入，动态替换Application。

# 实现

有了思路我们就可以敲代码了。

## 打包

打包时我们要改变Manifest中Application的标签值，可以通过自定义Gradle插件来实现，关键代码
 
```java
@TaskAction
    def updateManifest() {
        def ns = new Namespace("http://schemas.android.com/apk/res/android", "android")
        def xml = new XmlParser().parse(new InputStreamReader(new FileInputStream(manifestPath), "utf-8"))

        def application = xml.application[0]
        if (application) {
            def metaDataTags = application['meta-data']

            String rawApplicationName = application.attributes()[ns.name]
            metaDataTags.findAll {
                it.attributes()[ns.name].equals(TINKER_APPLICATION)
            }.each {
                it.parent().remove(it)
            }
            application.appendNode('meta-data', [(ns.name): TINKER_APPLICATION, (ns.value): rawApplicationName])
            application.attributes()[ns.name] = TINKER_APPLICATION_VALUE

            def printer = new XmlNodePrinter(new PrintWriter(manifestPath, "utf-8"))
            printer.preserveWhitespace = true
            printer.print(xml)
        }
    }
```

打包出的apk中的AndroidManifest.xml文件基本是这样的

```java
<application android:name="com.w4lle.onekeytinker.BootstrapApplication">
    ...
    <meta-data android:name="ONEKEY_TINKER_APPLICATION" android:value="com.w4lle.onekeytinker.App"/>
  </application>
```

其中的App是项目中原有的Application，BootstrapApplication是后期我们插入的Application。自定义Gradle插件时可以封装一个Extension配置参数，把Tinker的相关配置封装起来，一些不变的默认配置项都可以写到里面，这样项目的gradle配置可以更简洁。另外说一句，这个Gradle插件的顺序应该是打包工具生成Manifest之后，Tinker相关Task之前。

## 运行时替换Application

这一步的主要工作也是分两步，第一就是解析Manifest文件，拿到realApplication(App)和BootstrapApplication；然后hook 系统完成替换。

InstantRun中的替换实现

```java
public static void monkeyPatchApplication(@Nullable Context context,
                                              @Nullable Application bootstrap,
                                              @Nullable Application realApplication) {
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

主要做了两件事：

 1. 替换Application
    - baseContext.mPackageInfo.mApplication 代码3处
    - baseContext.mPackageInfo.mActivityThread.mInitialApplication 代码2处
    - baseContext.mPackageInfo.mActivityThread.mAllApplications 代码1处
 2. 替换mLoadedApk对象，代码4处
 
详细请查看[从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)

做完上面这两步这样就可以实现一键接入了。

# 兼容性

在上篇文章中我们提到，由于该方案大量hook系统api，在国内Android碎片化如此严重的市场环境下，该方案兼容性有一些问题，大概有 1/1w的概率会出现替换失败的问题，如果替换失败，那么在系统中运行的Application还是BootstrapApplication，而我们App中的Application已经没有了Application的生命周期和作用。

所以我们要在失败catch中调用下Application的生命周期方法以保证程序能够正常初始化启动起来。

```java
  try {
    ...
  } catch {
    e = true;
    realApplication.onCreate();
  }

  public void onConfigurationChanged(Configuration paramConfiguration)
  {
    if (e && realApplication != null) {
      realApplication.onConfigurationChanged(paramConfiguration);
      return;
    }
    super.onConfigurationChanged(paramConfiguration);
  }

  public void onLowMemory()
  {
    if (e && realApplication != null) {
      realApplication.onLowMemory();
      return;
    }
    super.onLowMemory();
  }

  @TargetApi(14)
  public void onTrimMemory(int paramInt)
  {
    if (e && realApplication != null) {
      realApplication.onTrimMemory(paramInt);
      return;
    }
    super.onTrimMemory(paramInt);
  }

  public void onTerminate()
  {
    if (e && realApplication != null) {
      realApplication.onTerminate();
      return;
    }
    super.onTerminate();
  }
```

这么做虽然能保证App能启动，但是实际上还会有隐性问题存在。比如App中有如下代码``((App) getApplication()).xxx();``，那么在替换失败的情况下可能就会崩了， 因为``getApplication()``得到的是BootstrapApplication，强转为``App``类型肯定就挂了。

# 总结

整体思路大概讲清楚了，虽然这种方案接入成本低，但是兼容性问题是个很麻烦的事情，说不定啥时候就崩了。推荐大家还是使用Tinker自有的接入方案。

# 参考
[从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)
[TinkerPatch](http://www.tinkerpatch.com/)