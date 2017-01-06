---
title: ClassyShark——apk分析利器
date: 2016-02-15 14:32:15
tags: [工具, Android]

---

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/ClassyShark.png)

## 背景
对于一个感兴趣的android非开源项目，通常使用逆向工程查看apk中的内容，俗称反编译。工具大概包括[dex2jar](https://github.com/pxb1988/dex2jar)、[JD-GUI](http://jd.benow.ca/)、[apktool](http://ibotpeaches.github.io/Apktool/install/)、[procyon](https://bitbucket.org/mstrobel/procyon),这些工具使用起来相对比较麻烦，如果我们只想知道该项目的基本框架、使用到哪些开源项目的话，那么就有些浪费时间。
对于一些大厂的项目，我们还比较关心的是用到了哪些新的框架和技术，对于新技术的流行程度和使用普遍程度有个比较好的把握，指导是否需要进行深度的使用学习。比如最近的比较流行的rxjava，热更新技术等等。

## ClassyShark


[ClassyShark](https://github.com/google/android-classyshark)是一款可以查看Android可执行文件的浏览工具，支持.dex, .aar, .so, .apk, .jar, .class, .xml 等文件格式，分析里面的内容包括classes.dex文件，包、方法数量、类、字符串、使用的NativeLibrary等。

### 使用方法

1. 打开apk文件``java -jar ClassyShark.jar -open <YOUR_APK.apk>`` 
2. 将生成的所有数据导出到文本文件里``java -jar ClassyShark.jar -dump <BINARY_FILE>``
3. 将指定类生成的文件导出到文本文件里``java -jar ClassyShark.jar -dump <BINARY_FILE> <FULLY_QUALIFIED_CLASS_NAME>``
4. 打开ClassyShark，在GUI界面展示某特定的类
5. ``java -jar ClassyShark.jar -open <BINARY_FILE> <FULLY_QUALIFIED_CLASS_NAME>``
5. 检测APK``java -jar ClassyShark.jar -inspect <YOUR_APK.apk>
``
6. 导出所有的字符串 ``java -jar ClassyShark.jar -stringdump <YOUR_APK.apk>``

### 具体使用
以美团项目为例，让我们看看能得到什么有用的信息


``java -jar ClassyShark.jar -open ~/Downloads/group-351_3-meituan_.apk``

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/Classy_group.png)

美团项目中使用了MultiDex,并且classes.dex文件有3个，说明方法数肯定非常多。
美团的编译版本非常新, 紧跟时代, 23版本(Android 6.0)。
并且TargetSdkVersion也是23版本，紧跟技术方向。
最低版本是16(Android 4.1), 4.1以下的手机无法运行。
而且有好多的so库，有美团自己的，也有好多是第三方的库。

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/Classy_group_2.png)

可以看到9万多个方法，怪不得会有3个classes.dex文件。
项目中应用了大量的第三方库，并且一般都是主流的比较稳定的开源库。
我们来看下都用到了哪些库

* [ZXing](https://github.com/zxing/zxing)二维码识别库;
amap: 高德地图;
* [PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh)使用最广的下拉刷新组件；
* [jackson](https://github.com/FasterXML/jackson-dataformat-smile), json解析库;
* [NineOldAndroids](https://github.com/JakeWharton/NineOldAndroids) Jake大神的android兼容库
* [fresco](https://github.com/facebook/fresco),facebook出品的图片处理库，图片加载节省很多内存，避免OOM。
* [RxJava](https://github.com/ReactiveX/RxJava)java响应式编程库，再加上``Square``的``Retrofit``库的支持，可以说未来就是``rxjava``的天下，目前市面上已经有很多基于rxjava的项目；我们团队也将基于rxjava来开发项目；
圈内最牛逼的开源公司[Square](https://github.com/square)，Jake大神所在的公司，可以毫不夸张的说，[Square](https://github.com/square)的开源项目使得Android开发提速了好几年
* [okhttp](https://github.com/square/okhttp)网络请求库，已被官方采用;
* [retrofit](https://github.com/square/retrofit)非常牛逼的网络请求库，配合``rxjava``和lambda使用，代码量减少90%;
* [otto](https://github.com/square/otto)事件总线;
* [picasso](https://github.com/square/picasso)图片加载库；
* [dagger](https://github.com/square/dagger)依赖注入框架；
* [ExpandableTextView](https://github.com/Manabu-GT/ExpandableTextView)可折叠的TextView
* iflytek, 科大讯飞的语音集成;
* [ViewPagerIndicator](https://github.com/JakeWharton/ViewPagerIndicator)还是Jake大神的项目，viewpager的滚动控件；
* [actionbarsherlock](http://actionbarsherlock.com/)依然是Jake大神的项目，Actionbar的适配库，不过已经过时了；
* [华为推送](http://developer.huawei.com/push)
* [SystemBarTint](https://github.com/jgilfelt/SystemBarTint)状态栏沉浸效果库
* 百度地图
* 新浪微博
* 腾讯的QQ和微信
* 大众点评,已经合并一家,东西也得用;
* [umpay](http://www.umpay.com/umpay_cms/), 联动优势支付;
* 支付宝；
* [andfix](https://github.com/alibaba/AndFix)阿里出品的android热更新框架；
* [flurry](http://www.flurry.com/)统计库；
* [小米推送](http://dev.xiaomi.com/doc/?page_id=1670)
* [http-request](https://github.com/kevinsawicki/http-request)网络请求库；
* [EventBus](https://github.com/greenrobot/EventBus)事件总线库；
* [PhotoView](https://github.com/chrisbanes/PhotoView)放大缩小的图片处理库；
* [roboguice](https://github.com/roboguice/roboguice)依赖注入框架，类似``Dagger``；
* [zip4j](http://www.lingala.net/zip4j/)处理zip压缩的库;
[link](https://github.com/BoltsFramework/Bolts-Android)异步task关联库,很像``rxjava``；

## 总结
从上面分析我们可见看出，美团是一个技术很开放的公司，对于框架的使用比较多，使用的基本都是主流的开发框架，减少开发成本，增强app的稳定性和体验，对于我们来说，有很大的借鉴意义。比如，目前都在试水的热更新框架，美团选择了阿里的``andfix``,那么该技术方案肯定是得到了美团团队的验证；另外，美团团队也是比较潮流的，``Retrofit``+``Rxjava``的潮流趋势已经不可阻挡，美团已经开始使用；但是，从项目引用库中我们也可以看到一些不足之处；比如，同一种框架引用了多种第三方库，如网络库(``okhttp``,``http-request``),图片加载库(``fresco``,``picasso``),事件总线(``EventBus``, ``Otto``),依赖注入(``Dagger``,``roboguice``)，推送相关的库等很多重复的库，如果去掉重复的库那么可以节省很多的编译时间和apk包的大小；还有就是，我们基本可以断定，美团团队的内部并不能很好的统一，没有有效的沟通，代码开发很混乱，导致项目结构上的臃肿，重复库的使用等等问题。

通过分析App的项目结构和引用库的信息，我们大致掌握了该项目的架构，一些开发中的经验和不足，拓宽下开发视野，发现一些好用的开源库，增强我们的武器，这些都是我们在开发中可以借鉴的东西。

参考

* [分析应用使用的技术框架和开源库](http://www.jianshu.com/p/8e8b88ea2197)