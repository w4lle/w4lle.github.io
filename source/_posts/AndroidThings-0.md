---
title: Android Things初探
date: 2017-01-06 17:26:47
tags: [Android Things]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/androidthings.png
---

# 背景

Android Things是Google推出的全新物联网操作系统
<!-- more -->
前身是去年发布的物联网平台 Brillo。Brillo 使用 C/C++ 基于 NDK 进行开发，而Android Things使用JAVA、Android Studio、Android SDK、NDK等进行开发，另外还新增了名为``Things Support Library``的库这个库有两个主要功能：通过多种协议和接口（GPIO、PWM、I2C、SPI、UART等）访问传感器和执行器的外围I/O API；以及一个用户驱动API（User Driver API），可以给应用程序添加新的设备驱动，用于将硬件事件注入系统，使它们可以为应用程序所用。还有，Android Things 天生支持物联网通讯协议 Weave，可让所有类型的设备能够连上云端并与其他服务如 Google Assistant 交互。Android Things目前支持三种硬件平台：Intel Edison、NXP Pico、Raspberry Pi 3。

![](http://7xs23g.com1.z0.glb.clouddn.com/things0.png)

简单来说，通过Android Things，开发者可以像开发Android应用一样简单地控制硬件设备。

# 搭建开发环境

硬件平台选择了树莓派3，需要准备的硬件设备：
 - 树莓派3开发板一个
 - SD卡一张，我买的32G
 - USB插口线一条
 - HDMI视频输出线一条
 - 显示器一台
 - 有线鼠标一个
 - LED灯2个
 - 杜邦线若干条
 - 网线一条

然后准备把从官网下载解压好的img文件烧入SD卡，步骤参考[官网](https://developer.android.com/things/hardware/developer-kits.html)和[树莓派官网](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)，我按照官网操作

```
    sudo dd bs=1m if=iot_rpi3.img of=/dev/disk2
```

提示``Resource busy``，[解决方案](http://raspberrypi.stackexchange.com/questions/9217/resource-busy-error-when-using-dd-to-copy-disk-img-to-sd-card)如下:

```java
    df -h
    sudo diskutil unmount /dev/disk2s1
    sudo dd bs=1m if=iot_rpi3.img of=/dev/rdisk2
```

烧好后把SD卡插入开发板的卡槽里，USB线连电脑，HDMI连显示器。然后通电开机。

![](http://7xs23g.com1.z0.glb.clouddn.com/things1.jpg)
![](http://7xs23g.com1.z0.glb.clouddn.com/things2.jpg)

看到这个界面就启动好了，实际上这个界面就是Android应用Launcher的一个Activity IoTLauncher。IoT(Internet of Things)物联网的简称。

```java
➜ code >adb shell dumpsys activity | grep "mFocusedActivity"
  mFocusedActivity: ActivityRecord{a7148f7 u0 com.android.iotlauncher/.IoTLauncher t1}
```

然后插网线，内网IP会显示在Launcher界面，然后用adb连接开发板，连wifi

```
    adb connect 192.168.1.143
    adb shell am startservice \\n    
    -n com.google.wifisetup/.WifiSetupService \\n    
    -a WifiSetupService.Connect \\n
    -e boohee \\n
    -e passphrase boohee0690apple
    adb shell ping 8.8.8.8
```

# 连接led灯

根据demo给的图连接led灯

![](http://7xs23g.com1.z0.glb.clouddn.com/rpi3_schematics.png)

这张图给出了两根线一个插地线，一个插绿色的GPIO。
这里放上一张图展示了树莓派上对应的针脚类型和name，接下来的操作led需要name才能控制。

![](http://7xs23g.com1.z0.glb.clouddn.com/gpio.png)

# 运行程序

官网提供了几个Sample，我们下载下来然后运行第一个Sample-Blink。

```java
    PeripheralManagerService service = new PeripheralManagerService();
    try {
        mLedGpio = service.openGpio("BCM5");
        mLedGpio.setDirection(Gpio.DIRECTION_OUT_INITIALLY_LOW);
    } catch {
    }
```

openGpio方法中的参数就是上面提到的针脚对应的name。程序跑起来之后就可以看到每隔一秒led闪一下，挺有意思。

我们稍微改造下，写一个xml文件中有一个button，每次点击button控制led灯的状态。

```java
    findViewById(R.id.btn_blink).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            try {
                mLedGpio.setValue(!mLedGpio.getValue());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    });
```

外接鼠标，每点一下button改变一下led的亮灭。
然后再改下，插两个led灯，第二个在BCM21 ``mLedGpio2 = service.openGpio("BCM21");``
然后每隔1s控制两个灯的状态取反。

```java
    mLedGpio2.setValue(mLedGpio.getValue());
    mLedGpio.setValue(!mLedGpio.getValue());
```

效果如图，gif压缩太严重，凑合看吧。
![](http://7xs23g.com1.z0.glb.clouddn.com/things3.gif)

对于开发者来说，通过Android Things控制硬件真的感觉跟开发一款Android App没有太大什么区别，同样的开发工具，同样的开发环境，甚至SDK大于24的Andriod App同样可以安装运行在Android Things系统上；而对于NDK开发，跟Android上也是一模一样。

# Android Everywhere

还记得RoyLi在做的名为Ruff的物联网平台，RoyLi是这样介绍它的
 >  Ruff 让不会做硬件的软件工程师根据产品经理的创意编写应用，(软件工程师)不用关心底层实现和驱动移植问题，通过开发者建立的应用生态最终解决物联网软件落后的问题。
Ruff的目的就是要做物联网领域的Android平台，采用比较流行的JavaScript作为编程语言直接操作硬件。

而现在Android Things已经移植到了物联网领域，Google的野心也可见一斑，其对于像Ruff这样的创业项目的冲击也可想而知。放一张2015年Google I/O大会上的旧图

![](http://7xs23g.com1.z0.glb.clouddn.com/google_to.png)

Android有庞大的开发者支持，完善的开发工具配套，也许Android真的可以做到连接万物的可能，Android在不远的未来真的可能无处不在，乘此东风，Android开发者也可大有可为。

# 安全

说到物联网就不得不说安全，在软件层面盗个号，个人信息被窃取可能感觉问题不大，但是对于联网的硬件来说就很可怕了，比如说门锁被破解，你家的煤气被远程控制，半夜你家电视突然响了，就问你怕不怕。

Google既然想统一IoT标准，那么IoT的安全性必须重视，因为在标准未统一之前破解某一款硬件设备性意义不大，而标准统一之后，只要破解这个平台就搞定了一切。真心希望Google可以做到让人满意的安全性。

# 参考

[Android Things](https://developer.android.com/things/hardware/index.html)
[Android Things Sample](https://github.com/androidthings)
