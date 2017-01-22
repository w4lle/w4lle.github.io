---
title: Gradle模块化配置
date: 2017-01-22 09:53:37
tags: [Android]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/gradle.jpg
---

本文以[AndResGuard](https://github.com/shwenzhang/AndResGuard)和[Tinker](https://github.com/Tencent/tinker)为例讲解下如何模块化配置Gradle，以及一键打Tinker补丁包的实现方法。
<!--more-->

# 背景

随着项目越来越大，引用第三方库的gradle愈来愈多，app的build.gradle文件也越来越长，拿薄荷App为例已经达到了1000行左右，一大坨代码赛在一起看起来很不爽。并且将第三方库的配置移植到新项目也很困难，稍微不注意就会出错。

# AndResGuard配置

首先新建resguard.gradle文件，配置内容如下：

```java
apply plugin: 'AndResGuard'
andResGuard {
    use7zip = true
    useSign = true
    keepRoot = false
    whiteList{...}
    compressFilePattern{...}
    sevenzip{...}
}
```

然后在主项目的build.gradle中加入

```java
apply from: '../resguard.gradle'
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath ('com.tencent.mm:AndResGuard-gradle-plugin:1.1.16')
    }
}
```

这样就配置好了，子Project依赖resguard脚本，sync下就能看到resguard Task了。那么移植到新项目只需要把resguard.gradle拷贝过去就ok了。

# Tinker配置

由于要配合AndResGuard使用，Tinker的gradle配置相对来说复杂些，然后我又在中间插入了自动备份和多渠道打包的task，然后就可以一键打Release包和补丁包。

首先新建tinker.gralde文件，首先把Tinker的官方基本配置copy进来此处代码略。先看下Project的build.gradle文件

```java
    apply from: '../tinker.gradle'
    apply from: '../utils.gradle'
    dependencies {
        classpath ("com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}")
    }
```

Project只需要配置这两行代码就可以了。其中的utils.gradle是一个工具脚本文件，其中有一些常用的工具方法。如下：

```java
//utils.gradle
ext {
    releaseTime = this.&releaseTime
    packageChannel = this.&packageChannel
}

def releaseTime() {
    return new Date().format("yyyy-MM-dd", TimeZone.getTimeZone("UTC"))
}

//多渠道打包，使用的美团方案
def packageChannel(String releaseApk) {
    print "begin package Channel! \n "
    try {
        def stdout = new ByteArrayOutputStream()
        exec {
            commandLine 'python', rootProject.getRootDir().getAbsolutePath() + "/app/multi_build.py", releaseApk
            standardOutput = stdout
        }
        return stdout.toString().trim()
    }
    catch (ignored) {
        print "package Channel Error! \n "
        return "UnKnown";
    }
}
```

其中的多渠道打包方案仍然使用的美团之前的往META-INF插入空文件的做法，是因为AndResGuard赞不支持v2签名，暂时无法使用新一代的v2签名多渠道打包方案[walle](https://github.com/Meituan-Dianping/walle)。

继续看tinker.gradle，首先需要配置依赖库和打包时的插入代码比如版本号。

```java
android {
    defaultConfig {
        //每次发补丁需要更改此处 增加patch版本号。升级app 版本要还原0.0
        buildConfigField "String", "PATCH_VERSION", "\"0.0\""
        buildConfigField "String", "PLATFORM", "\"all\""
    }
}

dependencies {
    provided("com.tencent.tinker:tinker-android-anno:${cfg.TINKER_VERSION}")
    compile("com.tencent.tinker:tinker-android-lib:${cfg.TINKER_VERSION}")
}
```

这样配置完Tinker就可以可用了，然后我们要处理和AndResGuard配置使用以及自动备份自动打渠道包，步骤如下：

 1. 使用resguard打正式包后需要备份apk、R、mapping、Resource mapping；并且打渠道包
 2. 打补丁包需要自动填入resguard新打出包的路径buildApkPath，apk、R、mapping、Resource mapping文件的对应路径。
 3. 打补丁包时resguard新打出的包不需要备份
 4. 需要区分debug和release切换备份路径

对应以上几点，给出关键代码：

```java
def cfg = rootProject.ext
def bakPath = file("${buildDir.parent}/tinkerBackup/")
def bakResguard = file("${bakPath}")
def debugBakPath = file("${buildDir.absolutePath}/tinkerBackup")
ext {
    appName = "nicepro"
    ...
    省略tinker配置
}
    def isNeedBackup = true
    task AtinkerPatchPrepare << {
        isNeedBackup = false
        def backUpVersion = ""
        if (new File("${bakResguard}/version.txt").exists()) {
            backUpVersion = new File("${bakResguard}/version.txt").getText()
        }
        if (!backUpVersion.isEmpty()) {
            print "app version : + ${backUpVersion} \n"
            project.tinkerPatch.oldApk = "${bakResguard}/${backUpVersion}.apk"
            project.tinkerPatch.buildConfig.applyResourceMapping = "${bakResguard}/${backUpVersion}_R.txt"
            project.andResGuard.mappingFile = file("${bakResguard}/resource_mapping_${backUpVersion}.txt")
        }
    }

    /**
     * bak apk and mapping
     */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = 'release'
//        def taskName = 'debug'

        if (taskName.equals("debug")) {
            bakResguard = file("${buildDir.absolutePath}/tinkerBackup")
        }

        tasks.all {
            if (variant.buildType.name == taskName) {

                def backUpVersion = ""
                if (new File("${bakResguard}/version.txt").exists()) {
                    backUpVersion = new File("${bakResguard}/version.txt").getText()
                }
                if ("tinkerPatch${taskName.capitalize()}".equalsIgnoreCase(it.name)) {
                    def resguardTask
                    tasks.all {
                        if (it.name.equalsIgnoreCase("resguard${taskName.capitalize()}")) {
                            resguardTask = it
                        }
                    }

                    it.doFirst({
                        // change build apk path
                        it.buildApkPath = "${buildDir}/outputs/apk/AndResGuard_${appName}_${cfg.versionName}_${releaseTime()}/${appName}_${cfg.versionName}_${releaseTime()}_signed_7zip_aligned.apk"
                        project.android.ext.tinkerOldApkPath = "${bakResguard}/${backUpVersion}.apk"
                        project.android.ext.tinkerApplyResourcePath = "${bakResguard}/${backUpVersion}_R.txt"
                    })
                    it.dependsOn AtinkerPatchPrepare
                    it.dependsOn resguardTask
                }

                if ("resguard${taskName.capitalize()}".equalsIgnoreCase(it.name)) {
                    if (!backUpVersion.isEmpty() && new File("${backUpVersion}_mapping.txt").exists()) {
                        ext.andResMappingFile = new File("${backUpVersion}_mapping.txt")
                    } else {
                        ext.andResMappingFile = null
                    }
                    it.doLast {
                        if (!isNeedBackup) {
                            return 0
                        }
                        def date = new Date().format("MMdd-HH-mm-ss")
                        def orgAndresPrefix = "AndResGuard_${appName}_${cfg.versionName}_${releaseTime()}"
                        def orgApkPrefix = "${appName}_${cfg.versionName}_${releaseTime()}"
                        def targetApkPrefix = "${appName}_${taskName.capitalize()}_${cfg.versionName}_${date}"

                        //backUpVersion
                        File version = new File("${bakResguard}/version.txt")
                        if (!version.parentFile.exists()) {
                            version.parentFile.mkdir()
                        }
                        version.write("${targetApkPrefix}")

                        copy {
                            from "${buildDir}/outputs/apk/${orgAndresPrefix}/${orgApkPrefix}_signed_7zip_aligned.apk"
                            into file(bakResguard.absolutePath)
                            rename { String fileName ->
                                try {
                                    fileName.replace("${orgApkPrefix}_signed_7zip_aligned.apk", "${targetApkPrefix}.apk")
                                } catch (Exception e) {
                                    print "rename apk mapping error"
                                    e.printStackTrace()
                                }
                            }
                            from "${buildDir}/outputs/mapping/${taskName}/mapping.txt"
                            into file(bakResguard.absolutePath)
                            rename { String fileName ->
                                fileName.replace("mapping.txt", "${targetApkPrefix}_mapping.txt")
                            }

                            from "${buildDir}/intermediates/symbols/${taskName}/R.txt"
                            into file(bakResguard.absolutePath)
                            rename { String fileName ->
                                fileName.replace("R.txt", "${targetApkPrefix}_R.txt")
                            }

                            from "${buildDir}/outputs/apk/${orgAndresPrefix}/resource_mapping_${orgApkPrefix}.txt"
                            into file(bakResguard.absolutePath)
                            rename { String fileName ->
                                try {
                                    fileName.replace("resource_mapping_${orgApkPrefix}.txt", "resource_mapping_${targetApkPrefix}.txt")
                                } catch (Exception e) {
                                    print "rename resource mapping error"
                                    e.printStackTrace()
                                }
                            }
                            print "one resguard backup tinker base apk ok! \n"
                        }
                        packageChannel("${buildDir}/outputs/apk/${orgAndresPrefix}/${orgApkPrefix}_signed_7zip_aligned.apk")
                    }
                }
            }
        }
    }
```

配置完成。发补丁包只需执行``./gradlew resguardRelease``便会打出多渠道包以及备份tinker需要的文件。

打tinker补丁不需要手动填依赖文件，包只需执行``./gradlew tinkerProcessRelease``。

正好现在并行三个项目，移植过程很快，copy两个gradle配置文件就搞定了。app的build.gradle代码也只有不到200行。
