---
title: 51信用卡 Android 架构演进实践
date: 2018-11-16 11:24:10
tags: [Android]
thumbnail: https://ws3.sinaimg.cn/large/006tNbRwly1fwrpnjc8xdj31hc0u0gt5.jpg
---

随着业务的快速扩张，原本小作坊式的单个工程的开发模式越来与不能满足实际需求。早在两年多以前，51信用卡管家就向下沉淀出了单独的公用基础库，一些通用的功能组件和个别独立的业务被拆分成 SDK，形成了一套中型项目、多人并行的开发模式，也为未来组件化拆分做准备。

<!-- more -->

![image-20181023162224348](https://ws1.sinaimg.cn/large/006tNbRwly1fwi9479hnhj31g00uk49x.jpg)

这套框架运行了一段时间之后，伴随着单应用内业务需求的增加、开发人员数量的增多、基础库数量的膨胀，导致了一些问题：

- 主工程代码耦合严重，牵一发而动全身
- 需求测试影响面大，不能聚焦单一业务模块
- 主工程代码越来越多，编译耗时
- 依赖倒置，业务代码依赖App工程
- SDK 界限模糊，基础库和业务库界限不明确
- 业务模块间可以任意依赖调用，依赖规则不明确
- 类库越来越多，不好管理

除了以上问题，动态化需求也越来越强烈，依赖 Hybrid + H5 打开页面慢的问题也凸显出来。

这些问题推动我们更进一步的升级开发构架。

# 组件化 or 插件化

## 动态化

最近两年，插件化框架层出不穷，各大厂都放出了自家开源的插件化框架。作为 Native 动态化与性能兼顾的插件化方案，很多公司选择插件化作为动态化技术方案。动态性通常有两部分的作用：一是动态热修复；二是动态下发业务插件。对于第一点，我们有热修复框架可以完成这部分工作；对于第二点，我们使用了 Hybrid 加载H5的方式实现，虽然性能上有所欠缺，但完全切到 Native 来做有点推倒重来的意思，并且跟业界同学交流后，对于动态下发业务插件用到的情况也不多，业务更新主要还是依靠 App 升级来实现。技术方案没有最优解，选择适合自己的才是最好的。

由于插件化也存在一些弊端，比如不可避免的 hook framework、修改 aapt、包装 Gradle Plugin、代理组件等等非常规操作，日常维护也是一笔不小的开销，稳定性、兼容性、新版本适配等等问题都需要考虑进去。对于 Android 端是否使用插件化，公司内部做过一些讨论，结论是不急着上，边走边看，先把业务组件拆分出来再说。

如今回过头看，自从 Android P发布以来，限制 hook framework 后，插件化逐渐开始式微，后面走向大概率是维护成本越来越高，成本收益比逐渐降低，最终弃坑不用。

除了插件化外，动态化方案近两年比较火的就是以 ReactNative、Weex 为代表的大前端方向，结合51信用卡的实际情况，最终选择拥抱大前端， Weex 作为动态化方案，以 Native 为主， Hybrid 离线化方案为辅，Weex 逐步迭代的架构开发模式。

Weex 的基础建设和前端同学合作，历经大半年时间，目前已经稳定应用在51信用卡各个 App 上，Weex 作为动态化页面的首选方案，已经完成了线上数百个页面的开发需求。配合离线化方案，各项性能指标也都达到要求。

## 组件分离

代码解耦与代码隔离，最有效的方案是工程隔离。审视我们最初的方案，每个 SDK 对应单独的仓库，通过 maven 依赖，通过工程分离隔离代码，这种方案没有问题，只不过需要往前更近一步，各个业务模块也需要独立主工程，拆分成独立的业务组件。

同时，划分清楚代码边界，控制依赖关系，梳理清楚层次结构，最终形成如下图所示的架构。

![组件化层次.001](https://ws3.sinaimg.cn/large/006tNbRwly1fwrpnjc8xdj31hc0u0gt5.jpg)

整体架构上提供三种容器：

- Native 容器，采用组件化架构，用于原生业务开发
- Hybrid 容器，webview 加载 H5，配合离线化方案
- Weex 容器，用于编写常规的页面，js 动态转化成 Native 控件，天然具有动态化特性，配合离线化方案，达到页面秒开的效果，同时共用 Hybrid 沉淀出的比较完善的 PG 方法

同时，Hybrid 和 Weex 依赖于原生提供的方法，通过 JsBridge 进行通信，目前共有 200 多个 PG 方法供 js 调用。长远来看，这三种容器并不会互相取代，相反地，它们应该是相互依存、取长补短、长期共存的状态。

# 组件化实践

Native 容器对应上图中各个层级的定义：

- 工程 App，各个应用工程，目前已有十多个应用并行开发，51信用卡管家作为平台应用，其余应用为独立的业务工程应用
- 业务组件，独立的业务组件，一般为复合业务组件，api 与实现分离，相互之间依赖隔离
- 基础业务 SDK，独立的小的单功能模块，提供基础功能，目前这一层级中还包含遗留未改造的部分业务组件
- 基础 Lib，业务无关的基础组件

组件化拆分的核心诉求是解耦合，提高组件内聚，所以应该从诉求出发，在沿用当下开发模式，并且不强依赖组件化框架的情况下，逐渐的进行组件化拆分。

通过工程隔离进而进行组件化拆分后，基本可以解决上面提到的问题：

- 高内聚，低耦合，代码边界清晰，代码变动影响面可以准确评估
- 提高开发效率，每个组件可以独立打包，单独调试，最多几十秒就可以完成打包过程
- 每个组件负责组件内的事情，理论上只要保证组件内部稳定，接入工程 App 后也不会产生新的问题
- 降低 App 工程编译时间，最理想的情况是，App 工程仅仅是一个空壳，用于加载各个组件

解耦，一般需要避免直接依赖，转为间接依赖，简单来说就是依赖隔离。对于组件化而言，每个组件都是单独的实现，单个组件对外提供的服务尽可能单一，依赖尽可能少；同时，依赖其它组件功能或页面的情况下，尽可能避免直接依赖，最好依赖中间层进行集中式管理，然后再进行逻辑分发。所以我们一般采用分总分的结构：组件内部分别注册，编译时生成汇总代码、运行时集中式管理，调用时处理逻辑分发。

![image-20181026181156315](https://ws2.sinaimg.cn/large/006tNbRwly1fwrpngbj7qj30yy0i4di0.jpg)

组件化需要解耦处理的几个基础模块：

- 页面路由
- 模块间调用
- 消息总线
- 数据总线

下面依次介绍。

## 页面路由

路由分发本质上是把直接依赖引用转化为中心化管理分发的一个过程，由于组件化拆分后，各个业务组件间不存在直接的依赖关系，所以必然要有一个统一收集页面跳转规则进而再分发的过程。

51信用卡在 2017 年就在进行路由化实践，以应对后面进行的组件化拆分需求，并沉淀出一套自研的路由框架 U51OkDeepLink，它也采用分总分结构，主要原理是组件内注册路由，编译时在组件内生成独立的路由表，并用 AOP 在编译时做好所有组件内路由表汇总的工作，调用初始化方法时进行路由表汇总，页面跳转时再进行管理分发，其用法很简单：

```java
//组件内注册路由
public interface SampleService {
    @Path("/main")
    @Activity(MainActivity.class)
    void startMainActivity(@Query("key") String key);
}

//其余组件唤起页面
new DeepLinkClient(context).buildRequest("old://app/main?key=value").addQuery("key2", "2").start();
```

并且支持强大的异步特性，支持跳转过程中的中间逻辑处理。

其原理图如下

![router](https://ws2.sinaimg.cn/large/006tNbRwly1fwrpnhjehsj31ju0uewka.jpg)

感兴趣的读者可以阅读 [Android 组件化 —— 路由设计最佳实践](https://www.jianshu.com/p/8a3eeeaf01e8) 获取更多技术细节。

## 模块间调用

组件间层次和边界模糊问题的产生，根本原因是各个业务组件间的相互依赖关系混乱，为了进行业务组件间的隔离，首先要做好组件之间的服务调用解耦。

这里采用的是 ServiceLoader 的模式，组件工程目录一般如下所示

![image-20181024204942179](https://ws1.sinaimg.cn/large/006tNbRwly1fwrpnfmr75j30bs052glm.jpg)

每个组件内一般声明三个 module：

- api module，声明对外暴露的服务接口和对外暴露的实体类及 Event 事件
- imp module，依赖 api module，是 api module 的具体实现，不对外暴露细节，不允许其他组件对 imp module 进行直接依赖
- app module，是工程的壳，可以直接运行调试，通过 SDKTemplate 创建生成，包含各种运行时所需环境

业务组件之间依赖 api 库的服务接口，imp 库作为实现动态查找。版本发布时，同时发布 api 和 imp 两个库，并且保证 api 和 imp 具有相同版本号，这个在组件发版时统一管理。

```java
 //组件内 api module 接口声明
@Service
public interface TestService {
    void sayHello();
}

//组件内 imp module 接口实现
@ServiceImpl
public class TestServiceImpl implements TestService {
    @Override
    public void sayHello() {
    }
}

//跨组件调用
compile 'com.u51.android:test-lib-api:$version'

CommentService service = ServicesLoader.getInstance().getService(TestService.class);
service.sayHello();
```

它的实现原理与路由类似，也是采用分总分结构，在编译时通过 APT 生成汇总代码，调用时动态查找注入 Service 及其实现类的绑定关系。

与路由初始化汇总路由表不同的是，ServiceLoader 在调用时查找，省去了初始化的逻辑，Service 不会像路由这么多，查找起来不会存在遍历太慢的问题。

## 消息总线

消息总线是基于 [EventBus](https://github.com/greenrobot/EventBus) 实现的跨三端（Native、Hybrid、Weex）事件管理分发组件 U51EventBus。跨三端是指在任意一端注册监听后，在事件触发时都可以得到响应。

对于原生开发来说，EventBus 本身可以满足需求，虽然有点事件满天飞的缺点，但是还在可接受范围之内。对于业务组件来说，其 Event 类需要放在 api module 中进行暴露。

对于 Hybrid 和 Weex 来说，一般的 bridge 都是 callback 形式得到异步响应，对于全局事件通知支持不太友好。通过 bridge 通道连接 U51EventBus 消息总线，打通跨三端全局的事件监听及分发，得以实现任意事件可以在 Native、H5、Weex 之间相互发送和监听。比如，类似登陆、登出操作在 Native 发出后，全局已打开的 H5 或 Weex 页面可以立即得到感知。

其实现原理也是采用分总分结构，在编译时对 EventBus 进行了定制封装，事件分发还是使用的原有的 EventBus 分发逻辑。

## 数据总线

数据存储采用基于 [Room](https://developer.android.com/topic/libraries/architecture/room) 实现的统一 KV 存储框架，底层数据库依然是 sqlite，性能这块没有做特别强调，强制其在子线程中进行操作，用于支持日常开发中配置和业务数据的存取操作。

另外，数据总线支持按模块进行存取，每个业务组件都可以定义自有 tag，避免字段冲突问题。

# 跨平台混合开发实践

无论从早期的 PhoneGap、Cordova，还是近年来比较火的 ReactNative、Weex，到最近两年崛起的 Flutter，跨平台混合开发一直深受众多开发青睐。究其原因，还是其跨平台和动态化是原生开发所不具备的特性。

## Hybrid 容器实践

Native 和 H5 混合开发一般是比较常见的混合开发模式，H5 开发效率高、迭代快速、不依赖 App 发版，51信用卡众多 App 产品中，有很多页面都是用 H5 来开发，嵌入原生 App 中使用 webview 进行加载显示。

早期 H5 容器在各个 App 中分别独立实现，没有统一的架构和规范，导致对 H5 的支持效率较低，PG 方法（来源于 PhoneGap）的开发、测试和维护都相当的混乱，重复性工作太多。

Native 层提供一套通用性强、功能丰富、稳定性高的 H5 容器对业务的高速发展至关重要。

![image-20181105173903973](https://ws2.sinaimg.cn/large/006tNbRwly1fwxgmjrohtj31kw0zkwth.jpg)

### 插件管理

由于 H5 不具备直接调用原生方法，所以原生壳要提供一套通用的通信方式，一般为 JsBridge，在 Android 端，实现 JsBridge 通信的通道一般有以下几种：

- shouldOverrideUrlLoading
- addJavascriptInterface
- onJsPrompt/onJsAlert

而通道不是关键，怎样管理和维护 PG 方法调用才是核心。为此，我们把每个方法定义为一个 Plugin，用插件的形式管理 PG 方法，这样可以做到每个插件独立运行，互不干扰。插件管理也是采用分总分结构，在各个业务组件中分别注册，编译是通过 APT 生成汇总代码，运行时进行插件汇总，最后调用通过 PluginManager 查找分发逻辑。

插件注册代码如下，其中 `onExecute()` 方法在 js 调用该方法时触发，执行结果通过 `evaluateJavaScript()` 方法异步返回。

```java
@JsPlugin(name = TestPlugin.PLUGIN_NAME, loadOnInit = false, version = 1)
public class TestPlugin extends EnNiuJsPlugin {
    public static final String PLUGIN_NAME = "TestPlugin";
    
    @Override
    public String getPluginName() {
        return PLUGIN_NAME;
    }
    ...
    @Override
    public boolean onExecute(String args) {
        doSomething();                    
        callbackContext.callback(...);
        return true;
    }
}
```

其中，H5 容器和插件都具有 Activity 生命周期感知能力，插件的生命周期：

![image-20181105191433330](https://ws4.sinaimg.cn/large/006tNbRwly1fwxgmhf6r9j31kw16eas1.jpg)

### 配套设施

插件统一通过插件管理平台进行维护管理，目前已有200+插件。PG 插件作为基础通用功能，采取集中式管理机制，任何人在新增、修改插件都需要进行相关负责人审核，以避免出现 Android、iOS 两端实现不统一，版本间实现不统一等问题。

![image-20181105191949792](https://ws4.sinaimg.cn/large/006tNbRwly1fwxgmi1nm7j31kw0ebdl6.jpg)

插件调试通过调试平台进行操作，浏览器中打开调试地址，App 端通过调试工具扫码建立连接，即可进行插件调试。

![image-20181105192429943](https://ws3.sinaimg.cn/large/006tNbRwly1fwxgmiq9l1j31kw110thw.jpg)

### 离线加载

Hybrid 混合开发的一大劣势就是性能比较差，打开页面较慢，特别是在弱网情况下。由于51信用卡业务大部分都是静态资源请求，参考业界做法，我们实现了动态下发离线包的方式来提升H5页面打开速度。

![lixianbao](https://ws1.sinaimg.cn/large/006tNbRwly1fwxgmkmdnlj31ca12wn45.jpg)

这里细节问题不具体展开。

除了以上提到的实践外，我们还做了很多工作，比如 UI 统一、Back 键拦截、公共参数处理、PG 白名单机制、H5监控、PG 方法监控等等，限于文章篇幅，这里不再一一列出，敬请关注后续相关文章。

## Weex 容器实践

在 Hybrid 已有配套基础上，51信用卡选择了 Weex 作为跨平台方案，经过一年的踩坑填坑过程，目前已经有 20+ 个项目、数百个 Weex 页面在线上稳定运行，并且，目前 Weex 方案趋于成熟，已经作为51信用卡端内首选业务方案。

### 共享插件

由于 Hybrid 良好的面向接口编程特性，在进行 Weex 基础建设过程中，很方便的就把已有的插件集成进来，并且共享已沉淀的配套设施。

```java
public class ENBridgeModule extends WXModule {
    @JSMethod
    public synchronized void send(String method, String args, JSCallback jsCallback) {
        ...
        weexWebView = weexEngine.getWeexVirtualWebView();
        EnNiuJsBridge enNiuJsBridge = weexWebView.getEnNiuJsBridge();
        enNiuJsBridge.notify(pg);
    }
}
```

注册 Weex 的 Module，并且每个 Weex Engine 中会新建出一个虚拟 webview，用于桥接 JsBridge 进而调用 PluginManager 进行插件逻辑分发。

Weex 容器实践在之前的文章中已经提到过一部分，具体请看 [Weex避坑指南-理论篇 ](https://mp.weixin.qq.com/s/PSquf5ILDykC9jYFu911qg)，后续还将有 Weex 实践相关的文章放出，这里不做过多篇幅的介绍，敬请关注后续相关文章。

# 工程化实践

工程化本质上是为了提高研发效率。51信用卡客户端团队自研的大风车管理平台，用于 App 管理、持续集成、类库管理、发版管理等，围绕客户端研发上下游流程，建立统一的管理入口。

目前，51信用卡 iOS 和 Android 共 30 多个应用 App、 200 多个类库依托大风车平台进行管理。下面主要介绍下类库管理相关内容。

## 类库管理

51信用卡目前有 100 多个 Android 类库，每个类库对应一个独立的 Gitlab 仓库。过多的独立组件及独立仓库，管理起来有些麻烦。

![image-20181030193630574](https://ws1.sinaimg.cn/large/006tNbRwly1fwrpnk3uc9j31kw0scaj6.jpg)

依托于大风车平台，所有类库的名称、最新版本及标签类型都会展示在列表页，标签类型对应组件化架构的层次结构，包括：基础组件、单业务功能组件、多业务功能组件。

在类库详情页，会有库的功能描述、groupId:artifactId 依赖信息、版本历史记录、分支信息、README、CHANGELOG、负责人等详情信息。

所有的类库管理工作都可以在大风车完成，包括新建类库、类库发版、查阅相关信息等等，这大大提高了基础组的研发效率，降低了团队间的沟通成本。

并且 App 工程中，该 App 所依赖的所有类库信息一目了然，在多人维护、多类库并行开发、类库频繁发版的情况下，依赖类库信息 check 更加便捷。

![image-20181031160854953](https://ws2.sinaimg.cn/large/006tNbRwly1fwrpng2bwcj31kw1540z0.jpg)

## 版本管理

由于类库之间是仓库隔离，所以它们的依赖关系是 maven 依赖，所有类库的 aar 包都需要发布到内部 maven 服务器上，上传工作由 PublishMavenPlugin 完成。

### SNAPSHOT 预览版

对于开发调试阶段，每个类库自带 DemoApp 工程，所以采用源码依赖；开发完成后，类库使用`SNAPSHOT`版本（比如 1.0.0-SNAPSHOT）发布到 maven 服务器，接入 App 工程后 push 代码触发大风车打包，进行集成测试。需要修改类库时，可以再重复发布相同版本的`SNAPSHOT`版本。

`SNAPSHOT`版本可以在开发同学自己的机器上进行打包发布。

### 正式版

对于发布阶段，类库必须使用正式版本发布，由于正式版本不可重复发布，这也就要求开发同学保证每个正式版本的版本质量，在正式发布前都应达到发布标准。

由于类库内部也存在相互依赖的情况，所以在类库正式发布时，不允许依赖包含`SNAPSHOT`版本的类库，`DependencyCheck`工作也会在  PublishMavenPlugin 完成。

同时，正式版本不允许开发同学在本机打包发布，PublishMavenPlugin 会检测是否在云端打包环境。功能分支经 CodeReview 后合并 master 分支，然后创建对应版本的 tag，触发大风车进行打包发布工作，发布成功后，会邮件通知 Android 组同学，并附带 CHANGELOG。

![image-20181105204154472](https://ws4.sinaimg.cn/large/006tNbRwly1fwxgz042voj312q0qa44m.jpg)

## 依赖管理

### 依赖传递

App 工程下采用 compile 依赖，compile 会解析类库 maven 包中的 pom 文件，进而间接依赖 pom 文件中声明的其他类库，也就是依赖传递。正常情况下，依赖传递会减少不必要的类库声明，当出现版本冲突时会自动处理 merge 操作。

但是，在多人协同工作、多类库并行开发情况下，事情变得有些复杂。考虑一种情况，应用 A 依赖类库 B，类库 B 依赖类库 C，正常情况下，A 中只需要声明依赖 B 即可，C 会被依赖传递过去。如果 C 中改变了方法签名，并且在应用 A 中显示声明依赖 C，编译时和运行时会分别出现什么情况？在编译时没有问题，正常编译通过；在运行时，当运行到类库 B 中使用的类库 C 中被改变签名的方法时，App crash。这是因为，maven 在处理类库版本 merge 时，会将 C 升级到最高版本，而此时 B 中已经编译好的 class 中使用的还是老版本 C 中的方法。

为了处理这个问题，我们使用 APICheckGradlePlugin 在编译时进行 check 操作，当发现被调用的方法找不到时，主动报错，将错误提前暴露在编译期，而非在运行时。同时内部强调 API 接口的向下兼容性，不用的方法标记为废弃，而非直接修改其方法签名或删除方法。

APICheckGradlePlugin 核心代码如下：

```java
try {
    c.getClassPool().get(callClassName)
    isClassNotFound = false
    m.getMethod()
} catch (NotFoundException e) {
    if (isClassNotFound) {
        dealException(String.format("在%s类中的第%d行是用到的%s类不存在", className, line, callClassName))
    } else {
        dealException(String.format("在%s类中的第%d行是用到的%s类的%s方法不存在", className, line, callClassName, methodName))
    }
}
```

### 多module发布

上文中提到，在多业务组件库工程中会有多个 module，一个 api module，一个 imp module，在使用 DemoApp 编译调试时采用源码依赖， imp module 依赖 api module，App 依赖 imp module，这样在打包上传 maven 时，会出现无法一起上传的问题；并且我们也要确保 api 和 imp 的版本号一致。为了解决这个问题，需要在上传时动态修改他们的 pom 文件，代码如下：

```java
modifyPom { pom ->
    pom.dependencies.findAll { dep -> dep.groupId == rootProject.name }.collect { dep ->
        dep.groupId = pom.groupId = rootProject.groupId
        dep.version = pom.version = rootProject.sdkVersion
    }
}
```

## 一键创建项目

### 模板工程

由于每个新建组件类库的 App 工程需要运行时环境基本相同，包括网络环境、调试环境、gradle 配置、通用依赖配置等等，这些重复性的工作最好放在一起统一处理。为此，我们创建了组件库的模板工程，只需要 clone 下来模板仓库，然后修改一些代码即可开发需求代码。

### 一键创建类库

但是，这种方式依然有很多共性的工作，比如 clone 代码、修改类库名、修改 groupId:artifactId、创建新的类库仓库、push 代码、在大风车中新建类库关联仓库地址等等操作。这些共性操作仍然可以用机器来操作，所以我们在大风车新建类库这一步中，把前面所有要做的事情全部做完，只需要在新建类库时填入必要的参数，一键就可以创建出可用的类库项目。

![image-20181031201425732](https://ws1.sinaimg.cn/large/006tNbRwly1fwrpngxyd4j31g40mqdht.jpg)

### 一键创建应用

随着我司 App 越来越多，新建 App 的配置同样面临类库刚开始时的困扰，新建 App 与新建类库本质上是一样的，只不过所需参数更多一些，并且这些参数可能不固定，有些 App 需要有些 App 不需要。参考类库，我们提取共性操作，创建了 App 的模板工程，并且对接大风车，一键即可创建出 App 工程，那些可变的参数留在模板工程中按需手动配置。

## 模块负责人

在组件化初步开始时，我们的每个模块都有固定的负责人，每个人手上都有固定的若干个模块，责任人对自己负责的模块负责。

但是随着组内的人员变动和业务变动，导致一些模块频繁易主，一些模块的文档长期处于不被维护状态，README 和 CHANGELOG 常年失修。

依赖大风车的类库管理，重新为每个模块指定负责人，并且梳理现存类库哪些缺失文档，进行补全。自从大风车自动抄送类库发版 CHANGELOG 后，CHANGELOG 不全的情况也大幅改善，基本每个新的版本都会附上该版本所做修改。

同时，我们也强调 CodeReview 机制，每个模块在提测前进行 CodeReview，强制merge request 必须有人点赞后才能合并 master 分支等等代码审查机制。未来，我们可能会进一步实践负责人 backup 方案，主副负责人相互 review，扩大大家技术视野的同时，可以进一步提高大家的主人翁意识。

# 总结

好的架构不是设计出来的，而是演进出来的。本文简单阐述了51信用卡 Android 架构演进的一些实践经验，同时我们坚信技术方案没有最优解，重要的是要选择选择适合自己的。脱离所处环境和问题本身谈技术方案，都将不能得到适合自身的开发架构。同时，我们也应当吸取和借鉴业界优秀的架构和设计理念，并将其根据自身适用场景加以改造，在理论和实践中逐渐交替探索演进。

当然，我们目前所使用的架构依然存在一些问题，比如组件拆分不完全、主工程业务仍然很多、CodeReview 机制不健全、代码扫描不够严格、一些组件库没有严格按照 api 工程来改造、一些老的组件依然没有 api module等等问题。我们也应该看到，正是因为这些实际的问题在推动我们进行技术改造，架构升级。同时，我们也要审视行业内大的方向，紧跟技术趋势，主动拥抱变化，毕竟技术世界唯一不变的，便是变化。