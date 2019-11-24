---
title: Android热补丁之Robust（三）坑和解
date: 2018-06-20 16:18:16
tags: [Android, 热补丁]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/robust.png
---

在前两篇文章中，分析了 Android 热补丁框架 Robust 中，几个重要的流程包括：

- 补丁加载过程
- 基础包插桩过程
- 补丁包自动化生成过程

本篇文章主要分析下集成过程中遇到的坑以及分析问题的思路和最终的解决方案。包含：

- 打补丁包出错？
- Robust 定义的 API 不够用怎么办？
- 插件 Plugin Transform 的顺序问题？
- 与 Aspectj 冲突怎么办？
- static 方法中包含 super 方法怎么办？

<!-- more -->

系列文章:

- [Android热补丁之Robust原理解析(一)](http://w4lle.com/2017/03/31/robust-0/)
- [Android热补丁之Robust（二）自动化补丁原理解析](http://w4lle.com/2018/05/28/robust-1/)
- [Android热补丁之Robust（三）坑和解](http://w4lle.com/2018/05/29/robust-2/)

# 打补丁包出错？

在打补丁包过程中，碰到了一个错误 `execute command java -jar /Users/wanglinglong/Develop/u51/Credit51/CreditCardManager/robust/dx.jar --dex --output=classes.dex  meituan.jar error`，找了一大圈最后发现是jdk老版本在Mac上的一个bug，升级jdk就好了，参考 [Class JavaLaunchHelper is implemented in two places](https://stackoverflow.com/questions/43003012/class-javalaunchhelper-is-implemented-in-two-places?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)

# Robust 定义的 API 不够用怎么办？

Robust 提供了一些 API 可供开发者扩展使用，比如：
添加类库依赖 `compile 'com.meituan.robust:robust:0.4.82'`，其中 `PatchManipulateImp` 类的一些可扩展方法

```java
protected List<Patch> fetchPatchList(Context context);

protected boolean verifyPatch(Context context, Patch patch);
    
protected boolean ensurePatchExist(Patch patch);
```

但是在一些情况下，这些可扩展方法并不能满足我们的需求。

为了满足定制化需求，可以弃用 `com.meituan.robust:robust`，自己实现一套补丁加载逻辑，这个实现起来难度并不太大，主要补丁加载流程都可以参考 Robust 官方实现，具体加载逻辑可参考本系列第一篇文章，这里不再深入。

# 插件 Plugin Transform 的顺序问题？

首先，要找到 Gradle Plugin 编译过程中对于自定义 Transform 的处理，具体流程读者可以自行搜索查看，这里只给出关键代码

```java
TaskManager.java
/**
 * Creates the post-compilation tasks for the given Variant.
 *
 * These tasks create the dex file from the .class files, plus optional intermediary steps like
 * proguard and jacoco
 */
public void createPostCompilationTasks(
        @NonNull final VariantScope variantScope) {
    ...
    // ---- Code Coverage first -----
    ...
    // Merge Java Resources.
    createMergeJavaResTransform(variantScope);
    // ----- External Transforms -----
    // apply all the external transforms.
    List<Transform> customTransforms = extension.getTransforms();
    List<List<Object>> customTransformsDependencies = extension.getTransformsDependencies();
    for (int i = 0, count = customTransforms.size(); i < count; i++) {
        Transform transform = customTransforms.get(i);
        List<Object> deps = customTransformsDependencies.get(i);
        transformManager
                .addTransform(taskFactory, variantScope, transform)
                .ifPresent(
                        t -> {
                            if (!deps.isEmpty()) {
                                t.dependsOn(deps);
                            }
                            // if the task is a no-op then we make assemble task depend on it.
                            if (transform.getScopes().isEmpty()) {
                                variantScope.getAssembleTask().dependsOn(t);
                            }
                        });
    }
    ...
}
```

在生成 Dex 任务之前，会处理所有的自定义 Transform 任务，逻辑是按照顺序遍历然后处理任务依赖关系，那么从dzhe'l这里我们可以知道，Transform 的执行顺序是按照插件的声明顺序来执行的，也就是说，哪个 plugin 的声明在前，其对应的 Transform 就在前执行，举个例子：

```java
apply plugin: 'pluginA'
apply plugin: 'pluginB'
```

那么对应的 TransformA 任务就会先于 TransfromB 任务执行。

好了，知道了 Transform 的执行顺序问题，再来看下 Robust 插件的顺序问题。首先来看下基线包的处理插件 `apply plugin: 'robust'`，其逻辑是在每个方法中前置插入补丁加载逻辑代码，用于拦截基线包中的原有逻辑，达到修复方法的目的。
如果 robust Plugin 是先于其他插件执行的，那么会出现 Robust 插入代码后，再执行其他插件的代码逻辑，这样会有问题吗？其实要具体问题具体分析，可能会有问题，也可能没有问题。举个例子，我们项目中使用了听云，而且是较老的版本(2.5.9)，其插件内部插入代码没有用到 Transform，而是用了另外一种技术，其会导致不管在哪里声明插件，其处理顺序都是最后执行，那么对于 Robust 基线包插件来说，当基线包插件插入完代码后，又会去执行听云插件的插入代码逻辑，所以可能会看到以下这种代码：

![](http://7xs23g.com1.z0.glb.clouddn.com/robust_tingyun.png)

实际上，我们最终想要的结果是这样的：

![](http://7xs23g.com1.z0.glb.clouddn.com/robust_tingyun_right.png)

然而，如果仔细观察这种错误的代码插入逻辑，实际上并没有对最终的热修复逻辑产生影响。是因为在生成补丁包过程中，他们的执行顺序也是这样的，即听云插件最后执行，这样的结果就是Robust 自动化补丁插件在生成插件后就强制停止了整个编译流程，听云插件根本就没有机会执行。所以最后补丁包中仅仅会包含剔除了Robust 基线包插件插入的代码以及听云插件插入的代码。可以理解为第一张图中把红框下面的代码通过热修复的方式移入了红框里面，然后return。

对于听云来说，2.8.x版本之后，插入代码的逻辑也由 Tranform 来执行，也就是说，对于听云来说，不管是怎么样的执行顺序，都不会与 Robust 发生兼容性问题。

所以还是要具体问题具体分析，这里仅仅提供一些排查问题的思路。

# 与 Aspectj 冲突怎么办？

我们项目中大量使用了 AOP 技术，涉及到的框架有 Aspectj、javassist、ASM。其中由于  javassist 和 ASM 完全是有自己控制的，所以不会有问题，而对于 Aspectj 来说就没这么简单了。

刚开始介入就碰到了这样一个问题 `Caused by: java.lang.ClassCastException: com.meituan.robust.patch.MainFragmentActivityPatch cannot be cast to com.zhangdan.app.activities.MainFragmentActivity`，是一个类型强转错误。

问题分析，由于 Aspectj 框架为开发者省略了很多逻辑，开发者只需要编写切面相关代码即可，所以需要梳理清楚 Aspectj 的原理：

首先贴下未混淆的代码：
```java
MainFragmentActivity.class
    static final void onCreate_aroundBody2(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint) {
        onCreate_aroundBody1$advice(mainFragmentActivity, bundle, joinPoint, MainFragmentActivity$$Injector.aspectOf(), (ProceedingJoinPoint) joinPoint);
    }
    private static final void onCreate_aroundBody1$advice(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint, MainFragmentActivity$$Injector mainFragmentActivity$$Injector, ProceedingJoinPoint proceedingJoinPoint) {
        if (!PatchProxy.proxy(new Object[]{mainFragmentActivity, bundle, joinPoint, mainFragmentActivity$$Injector, proceedingJoinPoint}, null, changeQuickRedirect, true, 11888, new Class[]{MainFragmentActivity.class, Bundle.class, JoinPoint.class, MainFragmentActivity$$Injector.class, ProceedingJoinPoint.class}, Void.TYPE).isSupported) {
            MainFragmentActivity mainFragmentActivity2 = (MainFragmentActivity) proceedingJoinPoint.getTarget();
            onCreate_aroundBody0(mainFragmentActivity, bundle, proceedingJoinPoint);
        }
    }

    public void onCreate(Bundle bundle) {
        if (!PatchProxy.proxy(new Object[]{bundle}, this, changeQuickRedirect, false, 11849, new Class[]{Bundle.class}, Void.TYPE).isSupported) {
            JoinPoint makeJP = Factory.makeJP(ajc$tjp_0, this, this, bundle);
            MainFragmentActivity$$Injector.aspectOf().onCreate(new AjcClosure3(new Object[]{this, bundle, makeJP}).linkClosureAndJoinPoint(69648));
        }
    }

    private static final void onCreate_aroundBody0(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint) {
        if (!PatchProxy.proxy(new Object[]{mainFragmentActivity, bundle, joinPoint}, null, changeQuickRedirect, true, 11887, new Class[]{MainFragmentActivity.class, Bundle.class, JoinPoint.class}, Void.TYPE).isSupported) {
            super.onCreate(bundle);
            JudgeEmulatorUtil.uploadEmulatorInfoIfNeed(mainFragmentActivity);
            instance = mainFragmentActivity;
            mainFragmentActivity.setContentView(R.layout.main_activity);
            ButterKnife.bind((Activity) mainFragmentActivity);
            mainFragmentActivity.initUserCenterManager();
            mainFragmentActivity.mainPagerAdapter = new MainPagerAdapter(mainFragmentActivity, mainFragmentActivity.getSupportFragmentManager());
            mainFragmentActivity.userInfoPresenter = new UserInfoPresenter();
            mainFragmentActivity.refreshOldDataPresenter = new RefreshOldDataPresenter();
            mainFragmentActivity.tabRedPointPresenter = new TabRedPointPresenter(mainFragmentActivity);
            mainFragmentActivity.getMsgCenterRedPresenter = new GetMsgCenterRedPresenter(mainFragmentActivity);
            mainFragmentActivity.userInfoPresenter.setUserInfoView(mainFragmentActivity);
            mainFragmentActivity.userInfoPresenter.startGetCurUserInfoDBUseCase();
            INSTANCE_FLAG = 1;
            BaiduLocation.getInstance(ZhangdanApplication.getInstance()).start();
            mainFragmentActivity.initToolBar();
            mainFragmentActivity.showImportBillDialog();
            mainFragmentActivity.onLoginCreate(bundle);
            mainFragmentActivity.getLoggerABConfig();
        }
    }
```

对应的patch 文件：

```java
public class MainFragmentActivityPatch {
    MainFragmentActivity originClass;

    public MainFragmentActivityPatch(Object obj) {
        this.originClass = (MainFragmentActivity) obj;
    }

    protected void onCreate(Bundle savedInstanceState) {
        StaticPart staticPart = (StaticPart) EnhancedRobustUtils.getStaticFieldValue("ajc$tjp_0", MainFragmentActivity.class);
        Log.d("robust", "get static  value is ajc$tjp_0     No:  1");
        JoinPoint joinPoint = (JoinPoint) EnhancedRobustUtils.invokeReflectStaticMethod("makeJP", Factory.class, getRealParameter(new Object[]{staticPart, this, this, savedInstanceState}), new Class[]{StaticPart.class, Object.class, Object.class, Object.class});
        Object obj = (Injector) EnhancedRobustUtils.invokeReflectStaticMethod("aspectOf", Injector.class, getRealParameter(new Object[0]), null);
        Object[] objArr = new Object[]{this, savedInstanceState, joinPoint};
        Log.d("robust", "  inner Class new      No:  2");
        Object obj2 = (AjcClosure3) EnhancedRobustUtils.invokeReflectConstruct("com.zhangdan.app.activities.MainFragmentActivity$AjcClosure3", getRealParameter(new Object[]{objArr}), new Class[]{Object[].class});
        if (obj2 == this) {
            obj2 = ((MainFragmentActivityPatch) obj2).originClass;
        }
        ProceedingJoinPoint proceedingJoinPoint = (ProceedingJoinPoint) EnhancedRobustUtils.invokeReflectMethod("linkClosureAndJoinPoint", obj2, getRealParameter(new Object[]{new Integer(69648)}), new Class[]{Integer.TYPE}, AroundClosure.class);
        Log.d("robust", "invoke  method is       No:  3 linkClosureAndJoinPoint");
        if (obj == this) {
            obj = ((MainFragmentActivityPatch) obj).originClass;
        }
        EnhancedRobustUtils.invokeReflectMethod("onCreate", obj, getRealParameter(new Object[]{proceedingJoinPoint}), new Class[]{ProceedingJoinPoint.class}, Injector.class);
        Log.d("robust", "invoke  method is       No:  4 onCreate");
    }
}
```

我们往下跟下 Aspectj 的调用流程。

```java
AjcClosure 类
public abstract class AroundClosure {
   ...
    protected Object[] state;

    public AroundClosure(Object[] state) {
    	this.state = state;
    }

    public Object[] getState() {
      return state;
    }

	/**
	 * This takes in the same arguments as are passed to the proceed
	 * call in the around advice (with primitives coerced to Object types)
	 */
    public abstract Object run(Object[] args) throws Throwable;

    public ProceedingJoinPoint linkClosureAndJoinPoint(int flags) {
        //TODO is this cast safe ?
        ProceedingJoinPoint jp = (ProceedingJoinPoint)state[state.length-1];
        jp.set$AroundClosure(this);
        this.bitflags = flags;
        return jp;
    }
}
```

AjcClosure 接收一个 Object 的对象数组，在基础包中，它的实现是

```java
new Object[]{this, bundle, makeJP}
```

注意这个 ``this``，代表的是MainFragmentActivity 对象。
相对应的看下patch包中的实现

```java
Object[] objArr = new Object[]{this, savedInstanceState, joinPoint};
```

这里调用了下 ``getRealParameter(new Object[]{objArr})`` 进行了 this 转换，所以这里的this 也是MainFragmentActivity对象，这里是没问题的。
然后调用 ``linkClosureAndJoinPoint`` 方法得到 ProceedingJoinPoint 对象，当做参数传递给 MainFragmentActivity$$Injector.onCreate(ProceedingJoinPoint joinPoint)  方法，看下这个方法的实现：

```java
MainFragmentActivity$$Injector.class
@Aspect
public class MainFragmentActivity$$Injector {
  @Around("execution(* com.zhangdan.app.activities.MainFragmentActivity.onCreate(..))")
  public void onCreate(ProceedingJoinPoint joinPoint) throws Throwable {
    MainFragmentActivity target = (MainFragmentActivity)joinPoint.getTarget();
    ...
    joinPoint.proceed();
  }
```

调用了 ProceedingJoinPoint.proceed 抽象方法，实现在

```java
    JoinPointImpl.java
	public Object proceed() throws Throwable {
		// when called from a before advice, but be a no-op
			return arc.run(arc.getState());
	}
```
这里的 ``arc`` 是 AroundClosure，``arc.getState()`` 返回的是构造 AroundClosure 时传递过来的对象数组。
最后调用了抽象方法 ``run(Object[] args)``，实现在

```java

public class MainFragmentActivity$AjcClosure3 extends AroundClosure {
    public static ChangeQuickRedirect changeQuickRedirect;

    public MainFragmentActivity$AjcClosure3(Object[] objArr) {
        super(objArr);
    }

    public Object run(Object[] objArr) {
        PatchProxyResult proxy = PatchProxy.proxy(new Object[]{objArr}, this, changeQuickRedirect, false, 11911, new Class[]{Object[].class}, Object.class);
        if (proxy.isSupported) {
            return proxy.result;
        }
        Object[] objArr2 = this.state;
        MainFragmentActivity.onCreate_aroundBody2((MainFragmentActivity) objArr2[0], (Bundle) objArr2[1], (JoinPoint) objArr2[2]);
        return null;
    }
}
```

最后调用 ``MainFragmentActivity.onCreate_aroundBody2((MainFragmentActivity) objArr2[0], (Bundle) objArr2[1], (JoinPoint) objArr2[2]);``，整个 AOP 的流程就走通了。

最后总结下 Aspectj 的调用流程：

```java
MainFragmentActivity.onCreate -> 
MainFragmentActivity$$Injector.onCreate(ProceedingJoinPoint joinPoint) -> 
ProceedingJoinPoint.proceed() -> 
AroundClosure.run(Object[] args) ->  
MainFragmentActivity$AjcClosure3.run(Object[] objArr) -> 
MainFragmentActivity.onCreate_aroundBody2((MainFragmentActivity) objArr2[0], (Bundle) objArr2[1], (JoinPoint) objArr2[2]); -> 
MainFragmentActivity.onCreate_aroundBody1$advice -> 
MainFragmentActivity.onCreate_aroundBody0();
```

最后的 ``MainFragmentActivity.onCreate_aroundBody0();``方法实际上就是onCreate()的原始方法逻辑。

另外，对于修改后的代码，没有被打入补丁，也是可以解释的。
对于 auto-path-plugin，Transform 的顺序是 Aspectj -> auto-patch.
那么，对于标记修改的 onCreate 方法来说，Aspectj 处理完后，onCreate 方法被替换成了代理，真正的方法实现被新生成的方法隐藏起来了。
而我们仅仅标记了旧的 onCreate 方法，其结果就是，Aspectj 的代理 onCreate 方法被 patch 了，而实际的方法虽然方法体内有我们的修复，但是由于没有标记 `@modify` 而被忽略。

因为是强转 crash，所以在 $$Injector 代码中插入一些 Log

```java
MainFragmentActivity$$Injector.class
    @Around("execution(* com.zhangdan.app.activities.MainFragmentActivity.onCreate(..))")
    public void onCreate(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        if (!PatchProxy.proxy(new Object[]{proceedingJoinPoint}, this, changeQuickRedirect, false, 11892, new Class[]{ProceedingJoinPoint.class}, Void.TYPE).isSupported) {
            MainFragmentActivity target = (MainFragmentActivity) proceedingJoinPoint.getTarget();
            ...
            Log.d("robust-wll", "test for aspectj start ");
            ...
            Log.d("robust-wll", "getTarget : " + proceedingJoinPoint.getTarget().getClass().getSimpleName());
            Log.d("robust-wll", "getThis : " + proceedingJoinPoint.getThis().getClass().getSimpleName());
            Log.d("robust-wll", "getArgs[0] : " + (proceedingJoinPoint.getArgs()[0] != null ? proceedingJoinPoint.getArgs()[0].getClass().getSimpleName() : null));
            Field arcField = proceedingJoinPoint.getClass().getDeclaredField("arc");
            arcField.setAccessible(true);
            AroundClosure arc = (AroundClosure) arcField.get(proceedingJoinPoint);
            if (!(arc == null || arc.getState() == null || arc.getState().length < 3)) {
                Object[] states = arc.getState();
                Log.d("robust-wll", "states[0] : " + (states[0] != null ? states[0].getClass().getSimpleName() : null));
                Log.d("robust-wll", "states[1] : " + (states[1] != null ? states[1].getClass().getSimpleName() : null));
                Log.d("robust-wll", "states[2] : " + (states[2] != null ? states[2].getClass().getSimpleName() : null));
            }
            Log.d("robust-wll", "test for aspectj end ");
            proceedingJoinPoint.proceed();
        }
    }
```

在不加载补丁情况下的 log 如下：

```java
04-03 17:13:41.184 15469 15469 D robust-wll: test for aspectj start 
04-03 17:13:41.184 15469 15469 D robust-wll: getTarget : MainFragmentActivity
04-03 17:13:41.185 15469 15469 D robust-wll: getThis : MainFragmentActivity
04-03 17:13:41.185 15469 15469 D robust-wll: getArgs[0] : null
04-03 17:13:41.185 15469 15469 D robust-wll: states[0] : MainFragmentActivity
04-03 17:13:41.185 15469 15469 D robust-wll: states[1] : null
04-03 17:13:41.186 15469 15469 D robust-wll: states[2] : JoinPointImpl
04-03 17:13:41.186 15469 15469 D robust-wll: test for aspectj end 
```

加载补丁后，log 如下：

```java
04-03 17:27:41.885 18490 18490 D robust-wll: test for aspectj start 
04-03 17:27:41.885 18490 18490 D robust-wll: getTarget : MainFragmentActivity
04-03 17:27:41.886 18490 18490 D robust-wll: getThis : MainFragmentActivity
04-03 17:27:41.886 18490 18490 D robust-wll: getArgs[0] : null
04-03 17:27:41.886 18490 18490 D robust-wll: states[0] : MainFragmentActivityPatch
04-03 17:27:41.886 18490 18490 D robust-wll: states[1] : null
04-03 17:27:41.886 18490 18490 D robust-wll: states[2] : JoinPointImpl
04-03 17:27:41.886 18490 18490 D robust-wll: test for aspectj end 
```

**states[0] : MainFragmentActivityPatch** 这个明显是不对的，所以我们知道了原因，是因为在构造 ``AroundClosure`` 时候传进来的参数不对。
报错地方对应于上面的分析，也就是
```java
        Object[] objArr = new Object[]{this, savedInstanceState, joinPoint};
        Object obj2 = (AjcClosure3) EnhancedRobustUtils.invokeReflectConstruct("com.zhangdan.app.activities.MainFragmentActivity$AjcClosure3", getRealParameter(new Object[]{objArr}), new Class[]{Object[].class});
```
结果就是，把一个含有3个对象的一维数据，编程了含有一个对象的二维数组，然后去 getRealParameter。
```java
    public Object[] getRealParameter(Object[] objArr) {
        if (objArr == null || objArr.length < 1) {
            return objArr;
        }
        Object[] objArr2 = new Object[objArr.length];
        for (int i = 0; i < objArr.length; i++) {
            if (objArr[i] == this) {
                objArr2[i] = this.originClass;
            } else {
                objArr2[i] = objArr[i];
            }
        }
        return objArr2;
    }
```
而这个方法只判断了一维数组的情况，没有判断二维或多维数组的情况。终于找到原因了 😃
对应的修改方法：
```java
    public Object[] getRealParameter(Object[] objArr) {
        if (objArr == null || objArr.length < 1) {
            return objArr;
        }
        Object[] objArr2 = new Object[objArr.length];
        for (int i = 0; i < objArr.length; i++) {
            if (objArr[i] instanceof Object[]) {
                objArr2[i] = getRealParameter(((Object[]) objArr[i]));
            } else {
                if (objArr[i] == this) {
                    objArr2[i] = this.originClass;
                } else {
                    objArr2[i] = objArr[i];
                }
            } 
        }
        return objArr2;
    }
```
castException 终于是搞定了。具体解决方法见 [merge request #259](https://github.com/Meituan-Dianping/Robust/pull/259)。

# static 方法中包含 super 方法怎么办？

看到这个标题可能会一脸懵逼，static 方法中怎么可能包含 super 调用？别急慢慢往下看。

书接上节，至于被 Aspectj  处理过的方法无法被打入 patch 的问题，理论上来说跟泛型的桥方法是类似的，解决方案也是 ``@Modify -> RobustModify.modify();``，修改后经验证，会报错。log如下
```java
Caused by: javassist.CannotCompileException: [source error] not-available: this
        at javassist.expr.MethodCall.replace(MethodCall.java:241)
        at javassist.expr.MethodCall$replace$2.call(Unknown Source)
        at com.meituan.robust.autopatch.PatchesFactory$1.edit(PatchesFactory.groovy:144)
        at javassist.expr.ExprEditor.loopBody(ExprEditor.java:224)
        at javassist.expr.ExprEditor.doit(ExprEditor.java:91)
        at javassist.CtBehavior.instrument(CtBehavior.java:712)
        at javassist.CtBehavior$instrument$1.call(Unknown Source)
        at com.meituan.robust.autopatch.PatchesFactory.createPatchClass(PatchesFactory.groovy:76)
        at com.meituan.robust.autopatch.PatchesFactory.createPatch(PatchesFactory.groovy:310)
        at com.meituan.robust.autopatch.PatchesFactory$createPatch.call(Unknown Source)
        at robust.gradle.plugin.AutoPatchTransform.generatPatch(AutoPatchTransform.groovy:190)
        at robust.gradle.plugin.AutoPatchTransform$generatPatch$0.callCurrent(Unknown Source)
        at robust.gradle.plugin.AutoPatchTransform.autoPatch(AutoPatchTransform.groovy:138)
        at robust.gradle.plugin.AutoPatchTransform$autoPatch.callCurrent(Unknown Source)
        at robust.gradle.plugin.AutoPatchTransform.transform(AutoPatchTransform.groovy:97)
        at com.android.build.api.transform.Transform.transform(Transform.java:290)
        at com.android.build.gradle.internal.pipeline.TransformTask$2.call(TransformTask.java:185)
        at com.android.build.gradle.internal.pipeline.TransformTask$2.call(TransformTask.java:181)
        at com.android.builder.profile.ThreadRecorder.record(ThreadRecorder.java:102)
        ... 27 more
Caused by: compile error: not-available: this
        at javassist.compiler.CodeGen.atKeyword(CodeGen.java:1908)
        at javassist.compiler.ast.Keyword.accept(Keyword.java:35)
        at javassist.compiler.JvstCodeGen.atMethodArgs(JvstCodeGen.java:358)
        at javassist.compiler.MemberCodeGen.atMethodCallCore(MemberCodeGen.java:569)
        at javassist.compiler.MemberCodeGen.atCallExpr(MemberCodeGen.java:537)
        at javassist.compiler.JvstCodeGen.atCallExpr(JvstCodeGen.java:244)
        at javassist.compiler.ast.CallExpr.accept(CallExpr.java:46)
        at javassist.compiler.CodeGen.atStmnt(CodeGen.java:338)
        at javassist.compiler.ast.Stmnt.accept(Stmnt.java:50)
        at javassist.compiler.CodeGen.atStmnt(CodeGen.java:351)
        at javassist.compiler.ast.Stmnt.accept(Stmnt.java:50)
        at javassist.compiler.Javac.compileStmnt(Javac.java:569)
        at javassist.expr.MethodCall.replace(MethodCall.java:235)
        ... 45 more
```

问题分析，根据堆栈显示，这里是在做替换 super 方法的逻辑，跟了下 plugin 的 debug，生成需要 replace 的 javassist 代码为 ``{staticRobustonCreate(this,originClass,$$);}``，然后在 replace 后，javac 编译这条语句的时候跪了。
分析下需要替换的 super 的方法，这个方法实际上是 Aspectj 处理后的方法，根据上面分析的 Aspectj 的调用流程得知，该方法实际上是 onCreate 方法原始的逻辑，反编译出来如下：

```java
    private static final void onCreate_aroundBody0(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint) {
        if (!PatchProxy.proxy(new Object[]{mainFragmentActivity, bundle, joinPoint}, null, changeQuickRedirect, true, 11887, new Class[]{MainFragmentActivity.class, Bundle.class, JoinPoint.class}, Void.TYPE).isSupported) {
            super.onCreate(bundle);//这里是需要替换的地方
            ...
        }
    }
```

而 auto-patch 做的工作是将``super.onCreate``方法包装成 static 方法，正常生成的patch代码如下：

```java
public class SecondActivityPatch {
    SecondActivity originClass;

    public SecondActivityPatch(Object obj) {
        this.originClass = (SecondActivity) obj;
    }
    public void onCreate(Bundle bundle) {
        staticRobustonCreate(this,originClass,bundle);
       ...
    }

    public static void staticRobustonCreate(SecondActivityPatch secondActivityPatch, SecondActivity secondActivity, Bundle bundle) {
        SecondActivityPatchRobustAssist.staticRobustonCreate(secondActivityPatch, secondActivity, bundle);
    }

public class SecondActivityPatchRobustAssist extends Activity {
    public static void staticRobustonCreate(SecondActivityPatch secondActivityPatch, SecondActivity secondActivity, Bundle bundle) {
        super.onCreate(bundle);
    }
}
```

而对于已经被 Aspectj 处理过的方法，是这样的：

```java
    private static final void onCreate_aroundBody0(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint) {
        if (!PatchProxy.proxy(new Object[]{mainFragmentActivity, bundle, joinPoint}, null, changeQuickRedirect, true, 11887, new Class[]{MainFragmentActivity.class, Bundle.class, JoinPoint.class}, Void.TYPE).isSupported) {
            //super.onCreate(bundle);//这里是需要替换的地方
            staticRobustonCreate(this,originClass,bundle);
            ...
        }
    }
```

在static 方法中使用了 ``this``关键字，当然编译出错啦。同理这个 ``originClass`` 也不可以出现，因为它是非 static 变量。
由于 xxPatchRobustAssist.staticRobustonCreate() 方法并没有用到前两个变量(patch, activity)，直接传 null 行不行呢？经验证是不行的，原因如下。
看了下生成xxxPatchRobustAssist类的代码：

```java
class PatchesAssistFactory {
    def
    static createAssistClass(CtClass modifiedClass, String patchClassName, CtMethod removeMethod) {
       ....
        StringBuilder staticMethodBuidler = new StringBuilder();
        if (removeMethod.parameterTypes.length > 0) {
            staticMethodBuidler.append("public static  " + removeMethod.returnType.name + "   " + ReflectUtils.getStaticSuperMethodName(removeMethod.getName())
                    + "(" + patchClassName + " patchInstance," + modifiedClass.getName() + " modifiedInstance," + JavaUtils.getParameterSignure(removeMethod) + "){");

        } else {
            staticMethodBuidler.append("public static  " + removeMethod.returnType.name + "   " + ReflectUtils.getStaticSuperMethodName(removeMethod.getName())
                    + "(" + patchClassName + " patchInstance," + modifiedClass.getName() + " modifiedInstance){");

        }
        staticMethodBuidler.append(" return patchInstance." + removeMethod.getName() + "(" + JavaUtils.getParameterValue(removeMethod.getParameterTypes().length) + ");");
        staticMethodBuidler.append("}");
        ...
        return assistClass;
    }
}
```

实际上，最终生成的调用是 ``xxPatch.superMethod($$);`` ，$$代表全部参数。对于与上面的 onCreate 方法就是 ``xxPatch.onCreate(bundle);``。
所以，patch 应该不能传 null 了，否则运行时会报空指针，那第二个参数 activity  能不能传 null 呢？继续往下看。
首先，根据常识，static 方法中肯定是不能调用 super方法的。从最终生成的代码也能看出，这并不是最终反编译出的的 ``super.onCreate(bundle)``方法调用。所以处理的地方肯定在javassist修改编译之后，对应处理的地方在smali 层，代码：

```java
SmaliTool.java
    private String invokeSuperMethodInSmali(final String line, String fullClassName) {
                    ...
                    result = line.replace(Constants.SMALI_INVOKE_VIRTUAL_COMMAND, Constants.SMALI_INVOKE_SUPER_COMMAND);
                    try {
                        if (!ctMethod.getReturnType().isPrimitive()) {
                            returnType = "L" + ctMethod.getReturnType().getName().replaceAll("\\.", "/");
                        } else {
                            returnType = String.valueOf(((CtPrimitiveType) ctMethod.getReturnType()).getDescriptor());
                        }
                        if (NameManger.getInstance().getPatchNameMap().get(fullClassName).equals(fullClassName)) {
                            result = result.replace("p0", "p1");
                        }
                       ...
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                ...
    }
```

实际上就是把方法调用从 ``invoke-invoke-virtual {p0, p2}, Lcom/meituan/robust/patch/SecondActivityPatch;->onCreate(Landroid/os/Bundle;)``V 转换成 ``invoke-super {p1, p2}, Landroid/support/v7/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V``，这步处理完才会真正的调用父类的super方法。
也就是说，在 smali 处理完后，参数从 p0 -> p1，也就是参数从 xxpatch 换成了 Activity，第二个参数会在运行时用到，所以也不能传null。
分析完了总结下，第二个参数 originClass 肯定不能传 null，否则会空指针；第一个参数  xxPatch，由于在 smali 被替换成了第二个参数，所以有可能是可以传 null 的。

解决方案：
1. 修改 originClass 为static，并新增一个 static patch 变量
2. 由于目前已经是在 static 方法中存在 super 方法，对应的 smali 代码：
3. 
```java
.method private static final onCreate_aroundBody0(Lcom/zhangdan/app/activities/MainFragmentActivity;Landroid/os/Bundle;Lorg/aspectj/lang/JoinPoint;)V
 ...
invoke-super {p0, p1}, Lcom/zhangdan/app/activities/WrapperAppCompatFragmentActivity;->onCreate(Landroid/os/Bundle;)V
```

所以只要不处理就好了，需要做的就是在 auto-plugin 中增加条件判断，符合 static 方法中带有 super 的不处理，一共有三处，一处是生成 xxPatchRobustAssist 辅助类，第二处在 javassit 替换 super 方法，第三处在 smali 处理补丁中的 super 方法。

对于方案1，问题：
如果 patch 和 originClass 都是 static，那么就会有内存泄露的风险。
并且如果被 patch 方法是 static 方法，那么在初始化 patch 时，originClass 会传 null。

```java
    public Object accessDispatch(String methodName, Object[] paramArrayOfObject) {
        try {
            Log.d("robust", new StringBuffer().append("arrivied in AccessDispatch ").append(methodName).append(" paramArrayOfObject  ").append(paramArrayOfObject).toString());
            MainFragmentActivityPatch mainFragmentActivityPatch;
            if (!methodName.split(":")[2].equals("false")) {
                Log.d("robust", "static method forward ");
                mainFragmentActivityPatch = new MainFragmentActivityPatch(null);
```

同样应该也是为了避免内存泄露，每修复一个方法就会生成一个 patch 对象并持有 static 的 originClass 引用。
对于方案2 ，问题：
首先，Aspectj 在 static 方法中插了个 super 方法（猜测也是在 smali 层做的修改），直接写的话 javac 编译时会报错，smali 处理吧还没到这一步。所以被修复后，这个 static 方法是在 xxPatch 类中的，auto-patch 即使不处理，运行时也不能正常运行，因为 xxPatch 不是 originClass 父类的子类，不能直接其调用 super 方法。

观察 Aspectj 生成的方法，所有 的static 方法，第一个参数都是当前类的引用，比如 ``private static final void onCreate_aroundBody0(MainFragmentActivity mainFragmentActivity, Bundle bundle, JoinPoint joinPoint) {``。
所以比如根据上面的分析，得出一个可行的方案：

```java
    def static String invokeSuperString(MethodCall m, String originClass) {
           ...
           stringBuilder.append(getStaticSuperMethodName(m.methodName) + "(" + null + "," + originClass + ",\$\$);");
           ...
    }
```

如果是 static 方法中含有 super 方法，就如下处理。
第一个 xxPatch 对象传空，最后在 smali 处理的时候会被替换掉。
第二个参数是从类似``onCreate_aroundBody0()``中传过来的，后面的是其他参数。
最终结果：

```java
    public static void staticRobustonCreate(MainFragmentActivityPatch mainFragmentActivityPatch, MainFragmentActivity mainFragmentActivity, Bundle bundle) {
        MainFragmentActivityPatchRobustAssist.staticRobustonCreate(mainFragmentActivityPatch, mainFragmentActivity, bundle);
    }

    private static final void onCreate_aroundBody0(MainFragmentActivity ajc$this, Bundle savedInstanceState, JoinPoint joinPoint) {
        ...
        staticRobustonCreate(null, ajc$this, savedInstanceState);
    }
```

到这里这个问题就分析完了，详细解决方发见 [merge request #265](https://github.com/Meituan-Dianping/Robust/pull/265)

# 总结

这个系列到这里基本就结束了。这篇文章主要介绍了在接入 Robust 过程中碰到的一些坑以及解决思路，其实根本还是熟读源码，碰到问题学习从源码中找答案。要坚信，坑踩的多了，也就不怕坑了。最后福利一张。

![](http://wx3.sinaimg.cn/large/0078WU5Lgy1fsgrhhx37tj30o31040zf.jpg)