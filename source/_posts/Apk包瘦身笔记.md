---
title: Apk包瘦身笔记
date: 2016-04-10 19:11:51
tags: [apk瘦身, 优化]
---

# 背景

随着业务功能的不断更迭，apk安装包的体积越来越大，最新的薄荷app android版本的安装包大小已经达到了22.5M，过大的安装包对于用户的体验来说会造成不好的影响，所以减小apk包的大小就显得尤为重要。

# 问题原因

[NimbleDroid](https://nimbledroid.com/)网站可以分析出apk安装包的各种大文件排行和各种依赖包的方法数，我们比较关注的是大文件的大小。上图
![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/large_file1.png)

不看不知道，一看真是吓一跳，超过100k的图片文件就那么多，而且超过300k的就有7个之多。
还有两个MP3文件也很大。

# 解决办法

## 资源混淆

资源混淆的方案采用的微信团队的[AndResGuard](https://github.com/shwenzhang/AndResGuard?spm=a313e.7916648.0.0.wwht7B)。使用方法输入一个apk文件，压缩混淆后输出一个apk包，采用引用gradle插件的方式集成到项目中，但是插件并不支持，所以就需要适配多渠道打包。薄荷项目中多渠道打包使用的是美团的方案，参考[Android批量打包提速](http://www.open-open.com/lib/view/open1418262988402.html)
方法如下

```gradle
    project.afterEvaluate {
        //在Release后执行资源混淆，然后多渠道打包
        //打包命令 ./gradlew resguard
        tasks.getByName("resguard") {
            it.doLast {
                print 'channel package task begin'
                def fileName = "one_v${defaultConfig.versionName}_${releaseTime()}_signed_7zip_aligned.apk"
                def rApk = new File("app/build/outputs/apk/AndResProguard/" + fileName)
                if (rApk.exists()) {
                    print 'resGuard apk exits begin channel package'
                    packageChannel(rApk.absolutePath)
                }
            }
        }
    }
```
执行完resguard混淆压缩后生成apk文件，调用python脚本执行多渠道打包。
还有一点需要注意的是，使用该方案要注意动态查找id导致找不到id的问题。需要添加白名单，项目中有明确的说明。但是对于引用第三方项目较多的来说，挨个添加id就很不现实了。
所以在白名单中添加所有的id
```java
 //for all id
            "com.boohee.one.R.id.*"
```

集成改方案后apk包大概较小了4M。并且资源文件也已经混淆，资源路径长路径变成段路径
``res/drawable/icon -> res/drawable/a``
对于反编译apk来说难度增加了。

## 删除无用代码和资源

在Android Studio中使用lint分析无用的文件
Analyze -> Run Inspection by Name

```
unusedResources
```
会列出无用资源列表，特别是图片全部干掉。
另外一点，由于项目更新迭代快，业务功能更改比较频繁，这就导致了项目中的无用代码特别多，但是依靠lint检查就比较有限，需要我们根据项目功能手动删除。删除无用代码后又会引出很多无用的资源文件，lint检查后再删除一遍。
做完这一步大概减少了1.5M左右。但是还有好多无用的代码还没有删除，以后有时间再处理。

## 去除无用的语言资源
具体项目具体执行，对于没有国际化的项目比如薄荷，在build.gradle中添加一句

```java
android {
    defaultConfig {
        resConfigs "zh"
    }
}
```
只使用中文资源，不光对本项目生效，还会对依赖的项目生效，比如support包。这样打包后只会打包中文资源，大概减小了1M左右。

## 删除多余的so库

薄荷一直以来至保留armable，这一点可以根据具体项目来定。

## 混淆
Proguard是编译时对java代码进行压缩，混淆，优化，预编译等操作的集成化工具。达到删除冗余，增加安全防护，减小大小的功效。薄荷一直以来并没有加入代码混淆，一方面原因是项目越来越大，混淆的话如果测试不充分很容易出问题，另一方面，薄荷一直提倡开源，并且核心并不在app端，并且有很多初学者会反编译薄荷app学习一些开发技巧。但是，对于提升用户体验来说这些都显得微不足道。我们也在着手做相关的混淆工作。

## 其余一些建议

* tiny图片处理 目前所知图片压缩效果最好的网站。蘑菇街写了tiny的gradle[插件](https://github.com/mogujie/TinyPIC_Gradle_Plugin)，薄荷还没有实践
* 删除音频文件 薄荷项目中有一些音频文件，好在下个版本改业务功能即将下线，所以这些文件可以直接删除；另外一种方法是可以走网络获取，在线化。
* 用更小的库
* png转成webp 可能会有点兼容性问题

# 参考
* [APK瘦身记，如何实现高达53%的压缩效果](http://jaq.alibaba.com/community/art/show?spm=a313e.7916646.24000001.3.T8AIXY&articleid=219)
* [安装包立减1M--微信Android资源混淆打包工具](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)
* [APK瘦身实践](http://www.jayfeng.com/2015/12/29/APK%E7%98%A6%E8%BA%AB%E5%AE%9E%E8%B7%B5/)]
* [如何将apk大小减少6M的](http://blog.csdn.net/UsherFor/article/details/46827587)