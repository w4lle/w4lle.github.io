---
title: Flutter 工程架构
date: 2019-11-23 22:11:04
tags: [Flutter]
thumbnail: https://raw.githubusercontent.com/w4lle/developnote/images/images20191123185156.png
---

本文主要介绍Android视角下在已有 App 中嵌入 Flutter 应用的实践，iOS 的方案思路基本一致。

<!-- more -->



系列文章：

- [Flutter 混合栈管理](http://w4lle.com/2019/11/22/flutter-hybrid-stack/)
- [Flutter 工程架构](http://w4lle.com/2019/11/23/flutter-project-manage/)

# Flutter 工程类型

Flutter工程中，通常有以下几种工程类型，下面分别简单概述下：

**1. Flutter Application**

​	纯净的Flutter App工程，包含标准的Dart层与Native平台层(android/&ios/)

**2. Flutter Module**

​	Flutter模块工程，仅包含Dart层实现，Native平台子工程的作用是构建Flutter产物，是通过Flutter自动生成的隐藏工程（.android/&.ios/）

**3. Flutter Plugin**

​	Flutter平台插件工程，包含Dart层与Native平台层的实现(android/&ios/)，往外提供API接口

**4. Flutter Package**

​	Flutter纯Dart插件工程，仅包含Dart层的实现，往往定义一些公共Widget



接入Flutter工程的两种方式：

1. 源码集成，在原生项目中Flutter作为lib直接嵌入Flutter代码，编译过程需要依赖Flutter环境，每个开发都需要配置Flutter开发环境，适合全新项目
2. Flutter项目作为子项目module，生成aar后由原生App依赖，对于App来说屏蔽了Flutter开发环境，在原有环境中即可打包，对App开发来说屏蔽了Flutter环境，适合已经App嵌入Flutter应用

本文主要介绍第二种方式，由于Flutter官方并未提供完整的解决方案，所以接入过程中会碰到一些问题，这里给出一些解决思路供参考。



创建一个flutter module：

flutter create -t module --org com.example my_flutter

工程结构：

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesD099FCC6-82A2-4170-A8A1-43C90F44CB65.png)



# FlutterModule 构建 Aar

flutter build aar

但是这个命令在Flutter v1.7.8版本中会提示找不到这个命令，估计是dev版本新加入的，还没有stable。

不久前加入的 https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps/_compare/1f606754ee0b18d9970e5fdd7b14d8a6df8d2d72...d648f6b063a0ab7ad4eaf0546e90b1cc75b9d58b

即使没有这个命令，也可以使用./gradlew assembleDebug进行编译，结果是一样的。



如果使用flutter build aar命令，Flutter官方给出的结果是，本地作为localmaven，生成的结构如下：

```shell
build/host/outputs/repo

└── com

└── example

└── my_flutter

└── flutter_release

├── 1.0

│   ├── flutter_release-1.0.aar

│   ├── flutter_release-1.0.aar.md5

│   ├── flutter_release-1.0.aar.sha1

│   ├── flutter_release-1.0.pom

│   ├── flutter_release-1.0.pom.md5

│   └── flutter_release-1.0.pom.sha1

├── maven-metadata.xml

├── maven-metadata.xml.md5

└── maven-metadata.xml.sha1
```

Flutter v1.2版本之后Flutter产物自动会打包到aar中，具体脚本见 flutter.gradle，主要做了三件事情：

1. 选择符合对应架构的Flutter引擎（flutter.so）
2. 解析 .flutter-plugins文件，把Flutter对应的android module动态添加到Native工程的依赖中，即动态添加implementation依赖，这块后面会详细讲
3. Hook mergeAssets/processResources Task，预先执行FlutterTask，调用flutter命令编译Dart层代码构建出flutter_assets产物，并拷贝到assets目录下



基于Flutter v1.7.8，编译产物 Debug版本

![](https://raw.githubusercontent.com/w4lle/developnote/images/images0517FC2D-73DC-4597-A471-8F87E51420B0.png)

Release版本 

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesCB70BFBA-A021-4D0C-9614-395D366BF292.png)

在Flutter v1.7之前，Release的产物如下：

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagese21cd2440bca39a70b30abedaaba4e3be5e090e4.png)

也就是说Flutter的编译产物，**从四个二进制文件变成了一个 libapp.so 文件**。

这里也涉及到so兼容arm的问题，之前把libflutter.so拷贝到arm目录下即可，现在编译出的libapp.so拷贝过去是有问题的，解决办法可以参考 [这个项目](https://github.com/lizhangqu/plugin-flutter-armeabi)。



如果在Flutter module 中依赖了Flutter Plugin，那么在App中依赖Flutter module编译出的aar时，会报错，例如

```java
ERROR: Unable to resolve dependency for ':app@debug/compileClasspath': Could not resolve io.flutter.plugins.sharedpreferences:shared_preferences:1.0-SNAPSHOT. 

Show Details

Affected Modules: app
```

下面会分析下为什么会报错。

# Flutter Plugin依赖原理

以Flutter Module为例，看下Flutter Plugin依赖的原理。



## flutter package get

当Flutter Module在pub中依赖了Flutter Plugin，并且在 flutter package get后，会从远程pubserver拉取依赖的Flutter Plugin，并生成一个文件记录依赖了哪些Flutter Plugin，名字为： ``.flutter-plugins``，内容为 k-v，如：

``flutter_webview_plugin=/Users/wanglinglong/Develop/flutter/.pub-cache/hosted/pub.flutter-io.cn/flutter_webview_plugin-0.3.5/``

被拉取到本地后的Flutter Plugin目录如下

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesimage2019-8-6%2011-51-21.png)



## 动态依赖

然后在.android/app/settings.gradle 文件中添加配置脚本，使其在Gradle初始化之后执行解析操作:

```java
rootProject.name = 'android_generated'
setBinding(new Binding([gradle: this]))
evaluate(new File(settingsDir, 'include_flutter.groovy'))
```

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesimage2019-8-2%2015-19-48.png)

这个脚本的作用是控制配置(evaluate)顺序，操作是解析.flutter-plugins，得到各个插件的android工程，使其在.android/Flutter 配置完成之后再进行配置解析。Gradle配置阶段的目标是解析每个project中的build.gradle。



然后在.android/Flutter/build.gradle中依赖 

``apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle" ``

这个文件中同样解析.flutter-plugins文件，遍历后一个一个 implementation 到./android/Flutter module工程里完成依赖。

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesimage2019-8-2%2015-22-42.png)

所以在Native层面来看这种依赖其实是本地依赖，只不过Flutter Plugin工程路径都在Flutter的环境变量下的缓存目录中。留个思考题，这里用 ``api`` 代替 ``implementation`` 行不行？



这里的依赖分成两个部分，一部分是原生Native依赖，一部分是Dart依赖。



对于原生部分，Flutter Plugin作为lib被Flutter Module本地依赖，根据Android 编译常识，在打包Aar后，这些**本地依赖的工程lib不能被一起打到Aar中**。

所以当App中依赖Flutter Module的产物Aar时，不能获得Flutter Module 中依赖的Flutter Plugin Android原生依赖，最终会报错。



对于Dart部分，通过pub进行管理依赖，相当于源码依赖，在打包时，这些Plugin的Dart源码部分都参与打包，最终生成Flutter的构建产物。



Flutter Module打包成Aar后，Aar中包含Android原生部分编译产物和Flutter Dart部分编译产物，后面App依赖该Aar后就可以脱离Flutter的编译环境，直接进行apk打包了。



那么如何解决Aar报错的问题？



# Flutter module 依赖 Flutter Plugin

已有方案大致原理都是收集Flutter Plugin 的Aar文件，然后进一步处理。



**方案一：**

使用 fat-aar，大致原理就是将所有的Flutter Plugin打包到同一个Aar中，这个方案比较臃肿，还可能涉及到gradle版本适配，而且可能产生多次依赖的问题。不建议使用。

**方案二：**

遍历所有依赖的Flutter Plugin，搜集Plugin Android工程下的Aar产物并copy到Flutter Module指定目录下，然后再push maven。

**方案三：**

遍历所有依赖的Flutter Plugin，根据Plugin版本信息，挨个打包成上传到maven。



由于公司内之前碰到到相关需求场景，即同一工程下多个Module如何统一管理的问题，最后解决方案就是使用的方案三，这里依然采用方案三，相比其他两种方案更稳定。



iOS 思路相同，打包成framework后上传到CDN，通过Pod进行依赖管理。



看下实现：

首先在.andorid/Flutter/build.gradle中依赖脚本 ``apply from: './publish_flutter_plugin.gradle'``

```java
def flutterUpload = gradle.rootProject.project(':flutter').tasks.getByName(uploadTaskName)
gradle.rootProject.ext.pluginList.each { name ->
    project.afterEvaluate {
        project.apply plugin: 'com.u51.publish'
        // 修改插件库对插件库的本地依赖为pom依
        modifyPom { pom ->
            pom.dependencies.findAll { dep ->
                if (rootProject.ext.pluginList.contains(dep.artifactId)) {
                    dep.groupId = rootProject.ext.groupId
                    dep.version = rootProject.ext.pluginMap[dep.artifactId]
                }
            }
        }
        // 设定插件本地库发布坐标版本号
        project.publish {
            groupId rootProject.ext.groupId
            version rootProject.ext.pluginMap[name]
            artifactId name
            compileEnvCheck false
            sources true
        }
        // flutter库 upload在插件本地库均upload完成后
        if (pluginUploadTask != null&&node==null) {
            flutterUpload.dependsOn(pluginUploadTask)
        }
    }
}
```



另外这种方案兼容持续集成环境，对于Native层面来讲，无需做任何改动。

# Flutter Plugin 依赖 Flutter Plugin

上面提到的是Flutter module依赖Flutter Plugin的情况，那么如果是Flutter Plugin工程依赖Flutter Plugin工程有没有问题？

先看下Flutter Plugin的项目结构 ``flutter create --template=plugin -i swift -a kotlin flutter_plugin``

在example下运行 flutter run命令，就可以跑起来了。



看下编译产物，其中Aar中只有原生的编译产物，并无Dart编译产物。



所以Flutter Plugin需要被pub依赖，pub将发布到远程的库下载到本地，然后在工程中

- 原生部分，Flutter Module通过上述方案动态添加maven 依赖
- Dart部分，Flutter Module通过pub依赖找到Dart源码，在Flutter Module中import引入，相当于源码依赖，共同参与编译，生成最终的 Dart产物



上面的场景都是Flutter Module(Flutter Application)依赖Flutter Plugin的情况，那么Flutter Plugin能不能依赖Flutter Plugin，会不会有问题？



首先Dart部分由于是pub依赖，相当于源码依赖，是没有问题的。



原生部分，由于Flutter Plugin在原生部分没有引入include_flutter.groovy(Android)，所以宿主Flutter Pugin无法动态include到子Flutter Plugin，找不到依赖。

解决办法就是再给它加上，方法同上。



# 工程结构

总结来说，对于 Flutter 来讲只有 Flutter Module 和 Flutter Application编译会生成Dart编译产物，而Flutter Application我们基本不用考虑。



所以很自然的得出结论，我们可以把Flutter Module作为隔离层，作为Flutter层的聚合统一输出给 App，App 工程只需通过 maven or Pod 依赖Flutter Module即可。

其余的Flutter相关依赖操作都交给Flutter Module，通过Pub进行管理即可。

这样通过若干个独立工程 Flutter Module、App、Flutter Plugin，就把整个构建流程建立起来了。



更进一步的，如果业务需要，可以在Flutter层进行更细化的划分，把Native组件化的思路对应到Flutter层。

每个Flutter Plugin可以作为独立的业务模块或基础组件，业务组件向下依赖基础组件，在打包时通过Pub传递依赖一起参与打包。

我们期望业务组件之间不互相依赖，就像之前的文章 [51信用卡 Android 架构演进实践](http://w4lle.com/2018/11/16/51credit-android-architecture/) 中讲的那样，业务组件不互相依赖只是约定，但不是强制约束，所以要通过一些手段达到强制约束的目的，由于架构还没有演进到这一步，所以没还没有调研方案，后面有需要再研究。



另外还有持续集成 Flutter 环境怎么配置？

以及Flutter版本管理的问题，Flutter版本升级API可能有变动，怎么保证开发同学使用同一版本的Flutter？



参考主流方案使用flutterw进行管理，不详细阐述。

整体结构借用一张图

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesflutter_hybrid_arch.jpg)



## Pub 私服

参考 [pub_server](https://github.com/dart-lang/pub_server) 搭建私服。

# 工程模板

当依照上面的思路愉快的开发时，会发现当触发`` flutter package get`` 命令，.android/&.ios/ 目录就都被还原了，里面的修改都被删除了。

这段操作在 flutter_tools 中进行，官方这么做的目的可能是不希望开发者直接修改.android&.ios/目录，因为这两个目录仅仅是参与编译的中间过渡。

而通过上面的分析，这两个目录是一定要修改的，不然跑不起来。



除此之外还有一个问题，上面提到远程依赖和本地依赖切换，在开发过程中选择本地依赖进行打包，此时Flutter是跑在App中的，需要调试时发现没有入口。

这里可以通过在Flutter Module 中 ``flutter attach``，然后重启App就可以连上Flutter的调试服务，才可以使用hot reload的功能，但是整个流程还是很长，并且hot reload并不是全能的，有些情况下并不能生效，还是要触发全量编译，有没有办法缩短构建路径？



上面的两个问题其实都可以通过自定义工程模板来解决，首先我们修改flutter_tools中的关键代码，使得不覆盖目录。

```dart
packages/flutter_tools/lib/src/commands/packages.dart
  @override
  Future<FlutterCommandResult> runCommand() async {
    // await rootProject.ensureReadyForPlatformSpecificTooling(checkProjects: true);
    logger.startProgress(
      '这是修改过的 flutter_tools 不会去修改.android 和 .ios 目录下的代码',
    );
    refreshPluginsList(rootProject);
  	...
	}

```

然后我们修改flutter_tools/template下的模板代码，使其满足我们的需求。这样新建的Flutter Module都是通过模板工程创建的，开发环境一致。

更进一步的，我们可以把**完整可运行**的Module模板放到仓库中使用submodule进行管理，这样每次模板更新后，Module可以实时更新。这部分可以放在flutterw中进行管理。



这里的完整可运行包含两个部分：

- 修改 .android/Flutter/ Module，使其可以满足编译构建需要
- 修改 .android/app，使其**拥有和工程App同样的运行时环境**，包括登陆注册环境、Hybrid Plugin环境(非Flutter Plugin，这里可以理解为不依赖Flutter环境的、更加通用的Hybrid插件系统)、调试环境、网络环境等等

当然这个方案也并不完美，因为涉及到修改flutter_tools，虽然改动不大，但每次Flutter版本还是要做版本兼容。



更好的方案是重写Dart构建流程，将flutter_tools中的流程串起来而不是修改它们，这样就不用做版本迁移了。



好在本身Flutter开发环境就是由flutterw动态配置的，我们把FlutterSDK放在内部仓库进行维护，通过统一管理入口进行管理，对开发来讲透明、无感知。



另外在App中，我们可以通过一些配置来切换远程依赖和本地依赖。

```java
gradle.properties
# flutter模块项目开关
includeFlutterProject=false
flutterProjectPath=../U51XiaoLanBenFlutter

setting.gradle
// flutter lib
if (includeFlutterProject == 'true') {
    setBinding(new Binding([gradle: this]))
    evaluate(new File(
            settingsDir,
            "${flutterProjectPath}/.android/include_flutter.groovy"
    ))
}

build.gradle
if (includeFlutterProject == 'true') {
	implementation project(':flutter')
  implementation project(':flutter_shell')
} else {
	implementation 'com.u51.android:xiaolanben-flutter:0.0.1-SNAPSHOT'
}
```

这样可以更方便的进行调试。



# 总结

本篇文章以Android为例子，主要介绍了在接入Flutter过程中碰到的一些问题及解决方案，实际上对于iOS来说也思路基本一致，我们的iOS方案也已经落地。



我们目前只在一个App中进行了实践，实际上对于另外的一个App工程，业务相同的Flutter Plugin应该做到通用性，只需要新建一个Flutter Module做中转就可以集成。

各个App的Flutter运行时环境应该是统一的，目前来看，这个工程架构可以满足多业务组开发并行Flutter的情况。



**注：另外这篇文章成文时间较久了，大概几个月前写的，方案对于目前的Flutter v1.9版本依然是适用的。**



参考：

- https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps
- [Flutter原理与实践](https://tech.meituan.com/2018/08/09/waimai-flutter-practice.html)
- [Flutter混合开发组件化与工程化架构](http://zhengxiaoyong.com/2018/12/16/Flutter%E6%B7%B7%E5%90%88%E5%BC%80%E5%8F%91%E7%BB%84%E4%BB%B6%E5%8C%96%E4%B8%8E%E5%B7%A5%E7%A8%8B%E5%8C%96%E6%9E%B6%E6%9E%84/)

