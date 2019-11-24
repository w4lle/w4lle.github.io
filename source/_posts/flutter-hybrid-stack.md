---
title: Flutter 混合栈管理
date: 2019-11-22 18:42:54
tags: [Flutter]
thumbnail: https://raw.githubusercontent.com/w4lle/developnote/images/images20191123185156.png
---

# Flutter 混合栈管理

本文主要聊一下 Flutter 混合栈，由于 Flutter 版本跨度较大，所以 Flutter API 也有很大变化，下文中前几个方案的实现看看就好，不用深究。重点关注兼容目前 Flutter 版本(v1.9.1) 的实现。本文以 Android 平台为例进行讲解。

<!-- more -->

系列文章：

- [Flutter 混合栈管理](http://w4lle.com/2019/11/22/flutter-hybrid-stack/)
- [Flutter 工程架构](http://w4lle.com/2019/11/23/flutter-project-manage/)

# 为什么需要混合栈？

在讨论 Flutter 之前，先看下 Weex 及 H5 Hybrid 如何处理多实例问题的。



对于Weex，其引入了 JavaScript 通过JS Runtime 完成动态运算，再把运算结果和Native客户端进行通信，完成真实ViewTree的构建、渲染等操作指令。

而当客户端面对多个 Weex 页面时，为了性能方面的考虑，并没有为每个 Weex 页面提供各自独立的 JS Runtime，相反只有一个 JS Runtime，这意味着所有的 Weex 页面共享同一份 JS Runtime，共用全局环境、变量、内存、和外界通信的接口等等。

所以我们可以看到，Weex初始化过程中，只需要初始化一次 JS Framework(weex-jsfm.js)，后面每次打开新的Weex页面都不必重新初始化。



为了隔离Weex独立页面的运行环境，在Native层面，每打开一个Weex页面，都会有一个唯一的WXSDKInstance，其中持有唯一InstanceId，在与JS双向通信过程中，每端都要携带InstanceId，例如声明周期调用：

- `createInstance(id, code, ...)`：：创建一个新的 Weex 页面，通过一整段 Weex JS Bundle 的代码，在 JS Runtime 开辟一块新的空间用来存储、记录和运算
- `refreshInstance(id, data)`：更新这个 Weex 页面的“顶级”根组件的数据
- `destroyInstance(id)`：销毁

指令调用：

- `sendTasks(id, tasks)`：发送指令
- `receiveTasks(id, tasks)`：接受指令

而在 JS Runtime中，每个InstanceId都有一份独立的运算和数据状态等与客户端相对应，JS通过闭包将其隔离在不同的闭包里，达到隔离的目的。



对于需要共享的数据，则不用InstanceId做对应，如WeexSDK初始化过程中的各种 registe：

- `registerComponents(xxcomponent)` ： 注册视图组件
- `registerModules(xxmodule)`：注册功能模块

对应Native端代码 ``mWXBridge.execJS("", null, METHOD_REGISTER_XX, args);`` ，第一个字段即InstanceId为空。这样即可在全局范围内查找并使用已测试的组件和模块，而不需要每个实例分别注册。



再看下 H5 Hybrid 混合开发，H5的容器是webview，可以在一个webview中管理所有H5页面，有点类似目前Flutter的方式；性能方面，Android平台来讲，从 Android 7.0开始，webview可选作为独立进程，Android8.0开始默认开启多进程模式，所以，即使App内打开多个webview也不会导致性能下降的很厉害。



而Flutter一开始的设计就是为了纯净的Flutter应用设计的，纯Flutter应用整个运行生命周期都只有一个FlutterView和Root Isolate，依靠Flutter Framework自身Route支持在FlutterView内部完成界面跳转，类似webview。借用一张图，在新版上有些变化，但大体相通。

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191123170804.png)

而 Isolate 之间在内存上是隔离的，这个隔离跟上面讲的Weex Instance隔离是两回事情。



Weex Instance隔离可以认为是两个内部类之间的隔离关系（实际是闭包），它们可以通过共同的外部类（一个比喻）来进行通信，可以共用同一份全局变量，也就是说它们之间是可以做到共用内存进行通信的。



而Isolate则是彻底的内存隔离，两个Isolate之间不存在内存上通信的可能性，只能通过第三方介入才可以通信。

试想一种情况，我们在多个页面Widget之间不能直接通信，并且像InheritedWidget也不能做到多Widget数据共享，而我们知道Flutter中的状态管理很大一部分方案都是依赖InheritedWidget来做数据共享的，这就相当于直接废弃了Flutter原生状态管理，得从Native绕道过来通信啦，这对开发来说体验太糟糕。



更重要的是对资源的占用，FlutterEngine运行环境初始就会占用很大内存，通信通道也会创建多个，缓存空间也会有多份，而且每个Engine会存在四个线程（实际是三个）：

- Platform Task Runner 相当于主线程，跟Flutter Engine的所有交互（接口调用）必须发生在这里，所有Engine实例共享同一个Platform Runner
- UI Task Runner 用于执行 Root Isolate，对创建的对象和Widgets进行Layout并生成一个Layer Tree，处理来自Native Plugins的消息响应，Timers，Microtasks
- GPU Task Runner 被用于执行GPU指令调用
- IO Runner用于 IO 读写

# Flutter 混合栈方案

总体来讲有几种种方案：

1. 多 Activity 多 FlutterView 方案，即多引擎方案
2. 共享 FlutterView，代表为闲鱼 [hybrid_stack_manager](https://github.com/FlutterRepo/hybrid_stack_manager)
3. 共享 FlutterNativieView，代表为 [微店](https://juejin.im/post/5c419c07f265da616f703aa1)
4. 共享 FlutterView升级版，代表为闲鱼 [Flutter_Boost](https://github.com/alibaba/flutter_boost)
5. 共享 Isolate，代表为 [头条](https://www.msup.com.cn/share/details?id=226)

其实核心思想都是公用同一个 FlutterEngine，避免不必要的资源浪费，优化性能及页面跳转体验，并实现多端逻辑统一。

下面会深入理解每个框架的实现细节。



## 多引擎方案

多引擎方案即一系列连续Flutter页面对应一个Activity(VC)，类似于webview中打开h5，但是存在着本质上的区别。

例如，我们进行下面一组导航操作：

``Flutter Page1 -> Flutter Page2 -> Native Page1 -> Flutter Page3``

我们只需要在Flutter Page1和Flutter Page3创建不同的Flutter实例即可。

这个方案的好处就是简单易懂，逻辑清晰，但是该方案也存在显著的问题：

- 性能问题，每个FlutterView对应一个FlutterEngine，FlutterEngine随着FlutterView的增多而线性增多，而FlutterEngine本身是一个较重的对象。包括线程数量、图片缓存、内存缓存、消息通道等都是存在多份的
- 通信问题，每个FlutterView对应的Isolate在内存上隔离，也就是说跨FlutterView的Widget间通信需要原生介入支持
- 转场动画问题，Native之间的跳转动画和Flutter Widget间的跳转动画不同，使用体验不太好

总结起来就是多引擎方案不适合在生产环境中使用。

## hybrid_stack_manager

这个框架实现思路很简单，即用XFlutterView包装FlutterView，进而代理 FlutterNativeView。

并且替重写了 FlutterWrapperActivity 用于替换了FlutterActivity，里面的逻辑是相似的，只不过把其中的FlutterView和FlutterActivityDelegate都换成了代理类。

其中的FlutterView是唯一的，全局共用一个FlutterView。

当发生跳转时，有几种情况：

**当原生跳转Flutter时**

- 原生跳转Flutter其实是跳转到FlutterWrapperActivity
- 在FlutterWrapperActivity.onCreate方法中动态绑定FlutterView，并将参数通过MethodChannel传递给Flutter
- Flutter通过Navigator管理页面widget，并且在Flutter层唯一确定一个FlutterWrapperActivity
- 在FlutterWrapperActivity.onResume方法中更新curFlutterActivity，代表当前的FlutterWrapperActivity

**当Flutter跳转Flutter时**

- Flutter跳转Flutter通过MethodChannel传值给原生，调用curFlutterActivity.openUrl方法
- 原生这边接收到参数后，再开启一个FlutterWrapperActivity2
- 截屏保存bitmap，绑定到对应的FlutterWrapperActivity1，并将截图显示出来
- FlutterActivity1.onStop方法，移除FlutterView
- 其余逻辑同上

**当Flutter跳转原生时**

- Flutter跳原生通过MethodChannel传值给原生，调用curFlutterActivity.openUrl方法
- 原生这边接收到参数后会返回一个Class，通过startActivity实现页面跳转
- 截屏保存bitmap，绑定到对应FlutterWrapperActivity，这种情况截屏不需要显示

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191122170546.png)

该方案基于一个事实：**任何时候我们最多只能看到一个页面。（特殊情况不在考虑范围内）**

如图所示，当从FlutterActivity跳转到另一个FlutterActivity时，FlutterView从FlutterActivity1移除，并动态绑定到FlutterActivity2。

此时，为了保证切换在显示上的统一，避免FlutterView从FlutterActivity1移除时页面出现白屏的情况，需要对FlutterActivity1进行截屏操作，并且将截屏显示出来。

上面只描述了打开页面的情况，对于返回操作是一样的，在onResume和onStop中分别做处理。



该方案可取之处：

- 每一个页面都有一个VC(Activity)，保证所有基于VC(Activity)生命周期的逻辑(如埋点等)照常工作
- 不同的Flutter页面之间可以正常通信，共享数据
- Native可以调起任意的Flutter页面，无论是首次打开还是之后

这种方案的缺点：

- 需要反射FlutterSDK，**侵入性强**
- 单例的HybridStackManager持有context上下文，容易造成内存泄露
- **强依赖Flutter版本**，实现基于Flutter v0.x
- 依赖FlutterSDK NavigatorState history属性，新版该属性已经私有化，所以 [新版不可用]( https://github.com/FlutterRepo/hybrid_stack_manager/issues/5)
- 同级的Flutter页面无法实现，例如tab中的同级Flutter页面
- 每个FlutterActivity都持有一张截屏的bitmap，占用内存空间

## 共享FlutterNativeView方案

跟方案2类似，只不过从全局共用FlutterView变为全局共用一个FlutterNativeView，保持一个Flutter页面对应一个原生Activity(VC)。[原理解析](https://juejin.im/post/5c419c07f265da616f703aa1)及部分[源码](https://github.com/voiddog/hybrid_stack_manager)

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesflutter_weidian.jpg)

在实现上跟方案二类似，但是要更简洁，废弃了FlutterActivity，重写了FlutterWrapperActivity，仍然复用Delegate用于管理生命周期，在onCreate方法中判断FlutterNativeView是否已经attach过，如果已经attach，那么就先detach操作，detach操作是重点。

同样的，在FlutterWrapperActivity的onDestroy方法中，也需要detach操作。

FlutterNativeView的声明是static，所以是全局唯一的，可以与任何FlutterWrapperActivity对应的FlutterView绑定。

因为在初始化时，getFlutterView和getFlutterNativeView都被ViewFactory的实现类FlutterWrapperActivity所重写，在构造FlutterView时，将唯一的FlutterNativeView当做参数，传进去就完成了FlutterView和FlutterNativeView的绑定。

并且FlutterNativeView绑定的context是ApplicationContext，所以不存在context内容泄露的风险。

```java
FlutterWrapperActivity
@Override
public FlutterView createFlutterView(Context context) {
    FlutterNativeView flutterNativeView = createFlutterNativeView();
    return new FlutterView(this, null, flutterNativeView);
}
 
@Override
public FlutterNativeView createFlutterNativeView() {
    if (sFlutterNativeView == null) {
        isCreatePage = true;
        sFlutterNativeView = new FlutterNativeView(getApplicationContext());
    }
    return sFlutterNativeView;
}
```

上面提到的detach方法是重点，是因为这是该方案唯一hook FlutterSDK的地方，实际上这里不能说是严格意义上的detach，最终调用的是 FlutterView.nativeSurfaceDestroyed()。

```java
public static void onSurfaceDestroyed(FlutterView flutterView, FlutterNativeView flutterNativeView) {
    try {
        //Flutter 较旧版本，新版本已不兼容
        Method nativeSurfaceDestroyed = FlutterView.class.getDeclaredMethod("nativeSurfaceDestroyed", long.class);
        nativeSurfaceDestroyed.setAccessible(true);
        nativeSurfaceDestroyed.invoke(flutterView, flutterNativeView.get());
    }
}
 
 
//对应的老版本engine代码 FlutterView.java
        @Override
        public void surfaceDestroyed(SurfaceHolder holder) {
            assertAttached();
            nativeSurfaceDestroyed(mNativeView.get());
        }
 
 
        private static native void nativeSurfaceDestroyed(long nativePlatformViewAndroid);
```

为了兼容新版本，只需要替换反射那里的实现即可

```java
//Flutter v1.7.8
Field mFlutterJNI = flutterNativeView.getClass().getDeclaredField("mFlutterJNI");
mFlutterJNI.setAccessible(true);
Object o = mFlutterJNI.get(flutterNativeView);
 
Method onSurfaceDestroyed = o.getClass().getDeclaredMethod("onSurfaceDestroyed");
onSurfaceDestroyed.invoke(o);
 
 
//对应的 FlutterSDK 代码 FlutterView.java
public void surfaceDestroyed(SurfaceHolder holder) {
    FlutterView.this.assertAttached();
    FlutterView.this.mNativeView.getFlutterJNI().onSurfaceDestroyed();
}
```

修改后在Flutter v1.7.8版本上可以顺利运行。

其中，[Flutter Engine代码地址](https://github.com/flutter/engine)，平台代码目录在 shell/platform/android

该方案最大的特点是不需要截屏，是因为FlutterView是和FlutterActivity绑定的，当切换FlutterActivity时，FlutterNativeView 从 FlutterView1 detach，此时FlutterActivity1中的FlutterView1显示的内容不再更新，所以显示内容不变，不用担心白屏的问题。

iOS 如果支持滑动返回的话可能还是需要截屏，因为在侧滑的时候，页面不一定结束。

总结一下，该方案的优点：

- hook 少，侵入性较少，就一处
- 不需要截屏，内存占用会稍微好一点
- 单例的FlutterNativeView不持有Activity的context，没有内存泄露的风险
- 支持页面间数据传递，切是await 的方式，非通知形式

缺点：

- 首次进入白屏时间较长
- 不支持平级的FlutterView展示，比如tab中的Flutter界面

总体来说，这个方案还是有很大的参考价值的。

## FlutterBoost

[项目地址](https://github.com/alibaba/flutter_boost)

该方案是多Navigator方案，要研究这个方案的实现，首先要先读下Flutter中路由管理和Widget层级关系的相关代码，可以看[这篇文章](https://juejin.im/post/5c8db8e8f265da2d864b889f)。

具体原理即Flutter层通过封装过的Widget，即ContainerManagerWidget，管理多个Navigator，每个Navigator对应一个（或多个）具体的业务Widget，并且支持当前Navigator中正常的push Widget的操作。

原生层和Flutter层的容器通过唯一id对应起来，并通过消息通道进行生命周期同步和数据交互。



这里原生层是驱动方，所有的页面级别的操作都是统一发送到原生层处理，然后再次分发同步给Flutter层依次处理。



### 多Navigator实现

这个方案的精髓在于，从FlutterView中的单Navigator栈级别的导航，进化到了多Navigator平级导航，即可以随时随地找到任意一个Flutter页面，它们之间的关系是同级的。这在之前的方案中是做不到的。

下面主要分析下多Navigator的实现。

首先，在Flutter中，万事皆Widget。

Navigator也不例外。

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191122173237.png)

对于Navigator的页面管理，比如 Navigator.of(context).push(route); 默认从当前控件的context依次向上寻找距离自己最近的NavigatorState，然后调用它的push方法入栈。

```dart
static NavigatorState of(
    BuildContext context, {
      bool rootNavigator = false,
      bool nullOk = false,
    }) {
    final NavigatorState navigator = rootNavigator
        ? context.rootAncestorStateOfType(const TypeMatcher<NavigatorState>())
        : context.ancestorStateOfType(const TypeMatcher<NavigatorState>());
    return navigator;
}

@override
State ancestorStateOfType(TypeMatcher matcher) {
  ///向父节点寻找类型匹配的对象
  assert(_debugCheckStateIsActiveForAncestorLookup());
  Element ancestor = _parent;
  while (ancestor != null) {
    if (ancestor is StatefulElement && matcher.check(ancestor.state))
      break;
    ancestor = ancestor._parent;
  }
  final StatefulElement statefulAncestor = ancestor;
  return statefulAncestor?.state;
}
```

所以只要在Navigator中再插入一个ContainerManagerWidget，进行拦截页面跳转的操作，用来管理多个Navigator，这样就实现了Flutter页面的扁平化操作，规避掉了原有Navigator的栈结构。

Flutter层的整体架构图如下

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191122173608.png)

其中，ContainerManager自己维护了一个Overlay，用于管理多Navigator的上下文切换。

由于OverlayState在遍历entry过程中是倒叙的，所以只要保证列表结构的 _leastEntries 在添加_ContainerOverlayEntry时，始终保持onstage需要前台展示的最后添加即可。





上面提到原生层和Flutter层的容器通过唯一id对应起来，并通过消息通道进行生命周期同步和数据交互。

而在Flutter层，每个Widget之间是共享内存的，它们之间可以共用同一套运行环境、全局变量、内存、通信接口等。所以他们之间可以正常通信，

这样看是不是和Weex JS 层有点像了。

实际上当深入了解后，会发现在DOM处理、View的映射关系上Flutter和Weex有很多相似支持。

比如Flutter中的三棵树 —— Widget、Element、RenderObject 和 Weex Native 中的三棵树—— WxDomObject、WXComponent、NativeView 之间的相似性等等。



### 原生层实现

对于原生层有两种实现，我们分别来看：

1. 共享FlutterView
2. 共享FlutterEngine



**共享 FlutterView ：**

分析的代码基于master分支，当前版本0.420，适用于**Flutter v1.5**之前的版本。

整体上来说，该方案使用了较为复杂的FlutterView共享管理方案，FlutterView仍然是单例的，但是相对于方案二hybrid_stack_manager有了长足的进步，逻辑复杂度也提高了不少。

下图是官方提供的原理图

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191122173908.png)

对于原生层和Flutter层来说，分别有一个ContainerManager用于管理调度Flutter容器，这个容器的概念，在原生层就是包装过的FlutterActivity，在Flutter层是Navigator。

简单来说，原生层通过ContainerManager管理包装过的FlutterActivity，从而共享单例的FlutterView。

这里为了避免切换过程中出现白屏的问题，依然需要截屏。

由于截屏，这里可能依然会出现闪动、黑屏的出现，见 [issue](https://github.com/alibaba/flutter_boost/issues/221)



**共享 FlutterEngine：**

分支：flutter_1.5_upgrade_opt 适配了**Flutter v1.5**版本。

该方案是多FlutterView，单FlutterEngine的方案，有点类似于共享FlutterNativeView方案。



实际上这里的**FlutterEngine就是Flutter v1.5版本之后用于替代FlutterNativeView的。**

**高版本的FlutterSDK，提供了embedding包，该包下提供了新的容器实现及TextureView的支持。**

该方案仍然是废弃了FlutterActivity，而自己组装了一个BoostFlutterActivity，并且废弃了delegate相关声明周期管理，所有的声明周期管理都是自己来管理。



不同点在于共享FlutterNativeView方案 detach过程中需要反射拿到FlutterJNI，进而调用onSurfaceDestroyed方法，而这个方案不需要，最终在FlutterView的detach过程中，调用路径 FlutterView.detach()->FlutterRender.

```
detachFromRenderSurface()->
FlutterRender.surfaceDestroyed()->
flutterJNI.onSurfaceDestroyed()->
FlutterJNI.nativeSurfaceDestroyed()
```

可以看到最终需要调用的方法是一致的，都需要调用

```
flutterJNI.nativeSurfaceDestroyed().
```



以Flutter v1.5为界限简单对比如下：

**Flutter v1.5 之前的 Android SDK：**

io.flutter.view.FlutterView: 与FlutterNativeView关联，FlutterNativeView通过DartExecutor对FlutterJNI下jni方法进行消息通道传递；

视图渲染实际实现为SurfaceView；

视图销毁与创建通过embedding包下的FlutterJNI通知native

核心成员：

```
private final DartExecutor dartExecutor;
private final FlutterRenderer flutterRenderer;
private final NavigationChannel navigationChannel;
private final KeyEventChannel keyEventChannel;
private final LifecycleChannel lifecycleChannel;
private final LocalizationChannel localizationChannel;
private final PlatformChannel platformChannel;
private final SettingsChannel settingsChannel;
private final SystemChannel systemChannel;
private final InputMethodManager mImm;
private final TextInputPlugin mTextInputPlugin;
private final AndroidKeyProcessor androidKeyProcessor;
private final AndroidTouchProcessor androidTouchProcessor;
```

**Flutter 1.5之后的 Android SDK提供了embedding包，废弃了io包：**

io.flutter.embedding.android.FlutterView 与FlutterEngine关联，废弃了FlutterNativeView；

视图渲染实际为FlutterSurefaceView或FlutterTextureView；

视图销毁与创建通过embedding包下的FlutterJNI通知native；

核心成员：

```
private FlutterView.RenderMode renderMode;
private FlutterView.TransparencyMode transparencyMode;
private RenderSurface renderSurface;
private FlutterEngine flutterEngine;
private TextInputPlugin textInputPlugin;
private AndroidKeyProcessor androidKeyProcessor;
private AndroidTouchProcessor androidTouchProcessor;
private AccessibilityBridge accessibilityBridge;
```

Flutter Engine核心成员：

```
private final AccessibilityChannel accessibilityChannel;
private final KeyEventChannel keyEventChannel;
private final LifecycleChannel lifecycleChannel;
private final LocalizationChannel localizationChannel;
private final NavigationChannel navigationChannel;
private final PlatformChannel platformChannel;
private final SettingsChannel settingsChannel;
private final SystemChannel systemChannel;
private final TextInputChannel textInputChannel;
```



另外，在最新版本 Flutter v1.9.1 已经提供了FlutterEngineProvider相关接口，即官方有意提供混合栈的管理方案，但现在只是个半成品，如果直接用的话，会发现返回键点不动，跟了下发现是把PlatformChannel的Handler给置空了，除此之外还有一些其他的问题。

具体实现参见最新版的Flutter_Boost，已经做好了Flutter v1.9.1的适配，总体上实现已经跟embedding差别不大，有差别的点在于FlutterEngine的attach和detach的时机不同、FlutterPlugin的生命周期做了下同步，感兴趣的自己去阅读，这里不详细说了。



总结一下：

优点：

- Flutter层多Navigator方案，支持同级Flutter Widget随意切换
- 提供了一种新的思路，理论上Flutter跳转可以仍然使用官方api，可以在中间拦截
- 多FlutterView，不需要截屏

缺点：

- 略有侵入性，各个Flutter版本需要适配



## 共享Isolate

头条的方案，多FlutterView，多FlutterEngine，单Isolate方案

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191122175942.png)

该方案需要修改Flutter engine源码，暂不考虑。

多FlutterEngine在同一运行环境下可以做到内存共享，但同时也需要注意内存同步的问题，毕竟每个FlutterEngine都各自持有UITaskRunner，可以同时操作同一份内存的，头条的解决方案是把这些线程全部做成共享的了。



# 总结

分析了上面几种混合栈管理方案，整体上来说闲鱼的共享FlutterEngine方案比较主流，其中的多Navigator有很大的参考价值。



对于51信用卡来说，可以以此为基础，建设符合公司内部使用的混合栈管理方案。主要有几个事情：

- Plugin 管理问题，Plugin 是指每条指令对应在Native（Android、iOS）上的实现，得益于Hybrid的基础设施建设，公司内部目前有200+ Plugin可以直接使用，这也就意味着大部分需要Native参与的功能，都已经实现好了，Flutter 端直接调用即可，这块可以参考之前的文章 [51信用卡 Android 架构演进实践](http://w4lle.com/2018/11/16/51credit-android-architecture/) 
- 路由管理，接入已有路由框架
- Flutter v1.9.1兼容



经过几个版本的迭代，目前已经完成了上面的几件事情，并在业务中使用。



**注：此篇文章成文时间较久，近期做了一些修改，主要增加了Flutter v1.9.1 及相关的内容。**



参考

- [Weex 在 JS Runtime 内的多实例管理](https://jiongks.name/blog/weex-multi-instance-runtime/)

- [Flutter混合开发——FlutterBoost](https://www.jiqizhixin.com/articles/2019-03-07-15)

- [Flutter新锐专家之路：混合开发篇](https://juejin.im/post/5b764acb51882532dc1812b1)

- [微店的Flutter混合栈管理技术实践](https://juejin.im/post/5c419c07f265da616f703aa1)

- [让Flutter真正支持View级别的混合开发](https://www.msup.com.cn/share/details?id=226)
- [为追求高性能，我必须告诉你Flutter引擎线程的事实](https://zhuanlan.zhihu.com/p/38026271)