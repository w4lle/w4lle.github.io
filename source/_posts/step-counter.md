---
title: Android计步
date: 2016-06-26 10:35:39
tags: [Android]
---

# 前言

计步越来越成为人们的强需求，一般有手环计步，手表计步，手机计步等实现，这篇文章讨论下手机上实现计步的两种方案。

# 加速度传感器（gsensor）

人在走路时，加速度传感器会形成一个类似正弦波形图，因此可以根据检测波峰波谷记步。见下图：

![](http://www.analog.com/library/analogDialogue/archives/44-06/AD44_06_FIG_03.jpg)

可以分为几步进行：

 1. 特征选取，由于手机在不同放置条件下三轴传感器会有不同的数据表现，所以 可以取三轴的平方和。
 2. 滤波，由于得到的数据存在一定的噪音，我们需要过滤掉这些噪音得到比 较平滑的数据，一般有中值滤波和低通滤波。
 3. 计步监测，我们一般采用判断峰谷值来决定是否是一次计步。
 4. 动态阈值，采用判断峰谷值来记步需要动态的计算阈值，符合阈值标准才能算一次计步。
 5. 步数矫正，人的步伐速度在200-2000ms之间，通过记录记步的时间戳，矫正步数。步伐间隔<200ms和>2000ms，认为是无效步数。

使用gsensor来计步可能会存在以下问题：

 - 续航问题，由于要不停的监测gsensor变化和计算，就会导致手机不会进入休眠状态，一直保持CPU的唤醒，这对于手机的续航可能会带来一定的影响。
 - 精度问题，计步算法需要不断的产品迭代，并且要有较好的步数矫正机制，短时间内不能做到优秀的计步功能。
 - 后台service保活问题，要保证后台可以实时统计步数需要后台服务一直运行，包括低内存防杀机制、自启机制等。
 - 对于一些锁屏自动关闭sensor的定制系统来说不可用。
 
# 计步传感器（step-sensor）

相对于使用加速度传感器获取数据和计算实现计步的方式，通过计步传感器sensor监测或者读取计步数对于终端的续航能力有了很大的提高。

在Android4.4（KITKAT）系统API提供了两种硬件计步传感器的支持，因为是硬件所以需要厂商支持。可以通过以下方式查看你的手机是否支持

 - adb shell pm list features
 如果有以下两项说明支持
```java
feature:android.hardware.sensor.stepcounter
feature:android.hardware.sensor.stepdetector
```
 - 代码检测Android4.4后更高并且有sensor支持
```java
    private boolean isKitkatWithStepSensor() {
        // BEGIN_INCLUDE(iskitkatsensor)
        // Require at least Android KitKat
        int currentApiVersion = android.os.Build.VERSION.SDK_INT;
        // Check that the device supports the step counter and detector sensors
        PackageManager packageManager = getActivity().getPackageManager();
        return currentApiVersion >= android.os.Build.VERSION_CODES.KITKAT
                && packageManager.hasSystemFeature(PackageManager.FEATURE_SENSOR_STEP_COUNTER)
                && packageManager.hasSystemFeature(PackageManager.FEATURE_SENSOR_STEP_DETECTOR);
        // END_INCLUDE(iskitkatsensor)
    }
```

在使用前需要声明权限

```java
<uses-feature android:name="android.hardware.sensor.stepcounter" />
<uses-feature android:name="android.hardware.sensor.stepdetector" />
```

 1. TYPE_STEP_COUNTER
 API的解释说返回从开机被激活后统计的步数，当重启手机后该数据归零，该传感器是一个硬件传感器所以它是低功耗的。为了能持续的计步，请不要反注册事件，就算手机处于休眠状态它依然会计步。当激活的时候依然会上报步数。该sensor适合在长时间的计步需求。

 2. TYPE_STEP_DETECTOR
 翻译过来就是走路检测，API文档也确实是这样说的，该sensor只用来监监测走步，每次返回数字1.0。如果需要长事件的计步请使用TYPE_STEP_COUNTER。

用法比较简单，实现比较方便，由于我们需要长时间的计步，所以一般我们采用``TYPE_STEP_COUNTER``。
优点：

 - 精度相对来说精确。
 - 低功耗，对手机续航基本没有影响。
 - 不需要后台服务持续唤醒，所以不需要后台保活。
 - 实现简单。

缺点：

 - 手机必须要Android4.4及以上版本，并且内置计步sensor硬件
 - 并不像iOS的M7协处理器记录每天各个时段的计步数据，step counter sensor记录从开机以来所有的步数，每日计步数需要自己维护。
 
# 总结

总结来说，如果用g-sensor来实现计步会保证良好的兼容性，排除锁屏自动关闭sensor的定制系统除外，基本所有的Android手机都可以使用，但是需要一定时间的算法调试和后台保活机制的健全保证，开发周期较长；如果项目需求开发周期较短，并且没有强制性的全机型兼容，那么可以考虑使用step-sensor，优点比较明显，而且使用协处理器代替纯软件计算实现计步监测已经是大势所趋。另外以上两种方案都需要开机自启权限。

# 参考

[如何在手机上实现高精度及自适应多种场景的计步器算法][1]
[Android-Developers-Samples-BatchStepSensor][2]
[利用3轴数字加速度计实现功能全面的计步器设计][3]


  [1]: http://www.wujiame.com/blog/2013/12/27/pedometer/
  [2]: https://github.com/johnjohndoe/Android-Developers-Samples/tree/master/BatchStepSensor
  [3]: http://www.analog.com/library/analogDialogue/china/archives/44-06/pedometer.html