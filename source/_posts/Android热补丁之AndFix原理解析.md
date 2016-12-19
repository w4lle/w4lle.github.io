---
title: Android热补丁之AndFix原理解析
date: 2016-03-03 14:43:44
tags: [Android, 热补丁]
---

# 背景

2015年下半年开源了很多Android热更新的项目，其中大部分是以QQ空间技术团队写的那篇[文章](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=1&srcid=1106Imu9ZgwybID13e7y2nEi#wechat_redirect)为依据写出的基于``multidex``的热更新框架，包括[Nuwa](https://github.com/jasonross/Nuwa)、[HotFix](https://github.com/dodola/HotFix)、[DroidFix](https://github.com/bunnyblue/DroidFix)等；还有这篇文章的主角，阿里开源的[AndFix](https://github.com/alibaba/AndFix)。

在这之前，热补丁框架并没有那么火，原因无非就是要么用起来太重，要么不支持ART。比如携程出品的[DynamicAPK](https://github.com/CtripMobile/DynamicAPK),这种框架是为了解决平台级的产品相关业务开发之间的解耦，热补丁只是其附属功能，对于量级没有那么大的项目，没有必要采用这种很重的框架。另外就是基于阿里出品的基于[Xposed](https://github.com/rovo89/Xposed)的AOP框架[dexposed](https://github.com/alibaba/dexposed)，剥离掉``Xposed``的root部分功能，主要应该与AOP编程、插桩 (如测试、性能监控等)、在线热补丁、SDK hooking等，用起来比较重并且不支持``ART``。
众多的热补丁框架为开发者带来了福利，不用发版本就可以紧急修复线上版本的bug。

这篇文章主要是分析AndFix的实现原理。

# AndFix

## 使用方法

引用

```gradle
dependencies {
    compile 'com.alipay.euler:andfix:0.3.1@aar'
}
```
初始化

```java
patchManager = new PatchManager(context);
patchManager.init(appversion);//current version
```
加载补丁，尽量在``Application``的``onCreate``方法中使用

```java
patchManager.loadPatch();
```
应用补丁

```java
patchManager.addPatch(path);//path of the patch file that was downloaded
```
项目中提供了一个生成补丁(后缀为``.apatch``)的工具``apkpatch``
用法：

```shell
usage: apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -f,--from <loc>        new Apk file path.
 -k,--keystore <loc>    keystore path.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
 -t,--to <loc>          old Apk file path.
 ```
 如下生成补丁文件
 
 ```shell
./apkpatch.sh -f new.apk -t old.apk -o ./ -k ../one.keystore -p *** -a one -e ***
```
## apkPatch工具解析

``apkpatch``是一个jar包，并没有开源出来，我们可以用``JD-GUI``或者``procyon``来看下它的源码,版本1.0.3。
首先找到``Main.class``，位于``com.euler.patch``包下，找到``Main()``方法

```java
public static void main(final String[] args) {
        .....
        //根据上面命令输入拿到参数        
       final ApkPatch apkPatch = new ApkPatch(from, to, name, out, keystore, password, alias, entry);
       apkPatch.doPatch();
  }
```
``ApkPatch``的``doPatch``方法

```java
public void doPatch() {
        try {
        //生成smali文件夹
            final File smaliDir = new File(this.out, "smali");
            if (!smaliDir.exists()) {
                smaliDir.mkdir();
            }
            //新建diff.dex文件
            final File dexFile = new File(this.out, "diff.dex");
            //新建diff.apatch文件
            final File outFile = new File(this.out, "diff.apatch");
            //第一步，拿到两个apk文件对比，对比信息写入DiffInfo
            final DiffInfo info = new DexDiffer().diff(this.from, this.to);
            //第二步，将对比结果info写入.smali文件中，然后打包成dex文件
            this.classes = buildCode(smaliDir, dexFile, info);
            //第三步，将生成的dex文件写入jar包，并根据输入的签名信息进行签名,生成diff.apatch文件
            this.build(outFile, dexFile);
            //第四步，将diff.apatch文件重命名，结束
            this.release(this.out, dexFile, outFile);
        }
        catch (Exception e2) {
            e2.printStackTrace();
        }
    }
```

以上可以简单描述为两步

1. 对比apk文件，得到需要的信息
2. 将结果打包为apatch文件

### 对比apk文件

``DexDiffer().diff()``方法

```java
public DiffInfo diff(final File newFile, final File oldFile) throws IOException {
		//提取新apk的dex文件
        final DexBackedDexFile newDexFile = DexFileFactory.loadDexFile(newFile, 19, true);
        //提取旧apk的dex文件
        final DexBackedDexFile oldDexFile = DexFileFactory.loadDexFile(oldFile, 19, true);
        final DiffInfo info = DiffInfo.getInstance();
        boolean contains = false;
        for (final DexBackedClassDef newClazz : newDexFile.getClasses()) {
            final Set<? extends DexBackedClassDef> oldclasses = oldDexFile.getClasses();
            for (final DexBackedClassDef oldClazz : oldclasses) {
            	 //对比相同的方法,存储为修改的方法
                if (newClazz.equals(oldClazz)) {
                	 //对比class文件的变量
                    this.compareField(newClazz, oldClazz, info);
                    //对比class文件的方法，如果同一个类中没有相同的方法
                    //则判定为新增方法
                    this.compareMethod(newClazz, oldClazz, info);
                    contains = true;
                    break;
                }
            }
            if (!contains) {
            	 //否则是新增的类
                info.addAddedClasses(newClazz);
            }
        }
        //返回包含diff信息的DiffInfo对象
        return info;
    }
```
其原理就是根据 ``dex diff``得到两个apk文件的差别信息。对比方法过程中对比两个``dex``文件中同时存在的方法，如果方法实现不同则**存储为修改过的方法**；如果方法名不同，**存储为新增的方法**，也就是说**AndFix支持增加新的方法**，这一点已经测试证明。另外，在比较``Field``的时候有如下代码

```java
  public void addAddedFields(DexBackedField field) {
    addedFields.add(field);
    throw new RuntimeException("can,t add new Field:" + 
      field.getName() + "(" + field.getType() + "), " + "in class :" + 
      field.getDefiningClass());
  }
  
  public void addModifiedFields(DexBackedField field) {
    modifiedFields.add(field);
    throw new RuntimeException("can,t modified Field:" + 
      field.getName() + "(" + field.getType() + "), " + "in class :" + 
      field.getDefiningClass());
  }
```
也就是说**``AndFix``不支持增加成员变量，但是支持在新增方法中增加的局部变量**。**也不支持修改成员变量**。已经测试证明这一点。
还有一个地方要注意，就是提取``dex``文件的地方，在``DexFileFactory``类中

```java
public static DexBackedDexFile loadDexFile(File dexFile, int api, boolean experimental) throws IOException
  {
    return loadDexFile(dexFile, "classes.dex", new Opcodes(api, experimental));
  }
```
可以看到，只提取出了``classes.dex``这个文件，所以源生工具并**不支持multidex**，如果使用了``multidex``方案，并且修复的类不在同一个``dex``文件中，那么补丁就不会生效。所以这里并不像作者在issue中提到的支持``multidex``那样，不过我们可以通过``JavaAssist``工具**修改``apkpatch``这个jar包，来达到支持multidex的目的**，后续我们会讲到。

### 将对比结果打包

这一步我们重点关注拿到``DiffInfo``后将其存入``smali``文件的过程
``ApkPatch.buildCode()``方法

```java
private static Set<String> buildCode(final File smaliDir, final File dexFile, final DiffInfo info) throws IOException, RecognitionException, FileNotFoundException {
        final ClassFileNameHandler outFileNameHandler = new ClassFileNameHandler(smaliDir, ".smali");
        final ClassFileNameHandler inFileNameHandler = new ClassFileNameHandler(smaliDir, ".smali");
        final DexBuilder dexBuilder = DexBuilder.makeDexBuilder();
        for (final DexBackedClassDef classDef : list) {
            final String className = classDef.getType();
            baksmali.disassembleClass(classDef, outFileNameHandler, options);
            final File smaliFile = inFileNameHandler.getUniqueFilenameForClass(TypeGenUtil.newType(className));
            classes.add(TypeGenUtil.newType(className).substring(1, TypeGenUtil.newType(className).length() - 1).replace('/', '.'));
            SmaliMod.assembleSmaliFile(smaliFile, dexBuilder, true, true);
        }
        dexBuilder.writeTo(new FileDataStore(dexFile));
        return classes;
    }
```
将上一步得到的``diff``信息写入``smali``文件，并且生成``diff.dex``文件。``smali``文件的命名以``_CF.smali``结尾，并且在修改的地方用自定义的**Annotation**(``MethodReplace``)标注，用于在替换之前查找修复的变量或方法，如下。

```smali
.method private getUserProfile()V
    .locals 2
    .annotation runtime Lcom/alipay/euler/andfix/annotation/MethodReplace;
        clazz = "com.boohee.account.UserProfileActivity"
        method = "getUserProfile"
    .end annotation

```
在打包生成的``diff.dex``文件中，反编译出来可以看到这段代码

```java
//生成的注解
@MethodReplace(clazz="com.boohee.account.UserProfileActivity", method="onCreate")
  public void onCreate(Bundle paramBundle)
  {
    super.onCreate(paramBundle);
    getUserProfile();
    addPatch();
  }
```

然后就是签名，打包，加密的流程，就不具体分析了。注意，``apkPatch``在生成``.apatch``补丁文件的时候会加入签名信息，并且会进行加密操作，在应用补丁的时候会验证签名信息是否正确。


## 打补丁原理

### Java层

``PatchManager.init()``方法

```java
public void init(String appVersion) {
		SharedPreferences sp = mContext.getSharedPreferences(SP_NAME,
				Context.MODE_PRIVATE);
		String ver = sp.getString(SP_VERSION, null);
		//根据版本号加载补丁文件，版本号不同清空缓存目录
		if (ver == null || !ver.equalsIgnoreCase(appVersion)) {
			cleanPatch();
			sp.edit().putString(SP_VERSION, appVersion).commit();
		} else {
			initPatchs();
		}
	}

	private void initPatchs() {
		// 缓存目录data/data/package/file/apatch/会缓存补丁文件
		// 即使原目录被删除也可以打补丁
		File[] files = mPatchDir.listFiles();
		for (File file : files) {
			addPatch(file);
		}
	}
```

``addPatch``和``loadPatch()``方法

```java
	public void addPatch(String path) throws IOException {
		...
		FileUtil.copyFile(src, dest);// copy to patch's directory
		Patch patch = addPatch(dest);
		if (patch != null) {
			loadPatch(patch);
		}
	}
	
	private void loadPatch(Patch patch) {
		Set<String> patchNames = patch.getPatchNames();
		ClassLoader cl;
		List<String> classes;
		for (String patchName : patchNames) {
			if (mLoaders.containsKey("*")) {
				cl = mContext.getClassLoader();
			} else {
				cl = mLoaders.get(patchName);
			}
			if (cl != null) {
				classes = patch.getClasses(patchName);
				mAndFixManager.fix(patch.getFile(), cl, classes);
			}
		}
	}
```

再看下``AndFixManager``的``fix()``方法
	
```java
...
//省略掉验证签名信息、安全检查的代码，安全方面做得很好
...

private void fixClass(Class<?> clazz, ClassLoader classLoader) {
		...
		for (Method method : methods) {
			//还记得对比过程中生成的Annotation注解吗
			//这里通过注解找到需要替换掉的方法
			methodReplace = method.getAnnotation(MethodReplace.class);
			if (methodReplace == null)
				continue;
			//标记的类
			clz = methodReplace.clazz();
			//需要替换的方法
			meth = methodReplace.method();
			if (!isEmpty(clz) && !isEmpty(meth)) {
				//所有找到的方法，循环替换
				replaceMethod(classLoader, clz, meth, method);
			}
		}
	}
	
	private static native void replaceMethod(Method dest, Method src);
	private static native void setFieldFlag(Field field);

	public static void addReplaceMethod(Method src, Method dest) {
		try {
			replaceMethod(src, dest);
			initFields(dest.getDeclaringClass());
		} catch (Throwable e) {
			Log.e(TAG, "addReplaceMethod", e);
		}
	}
```
后面就是调用``native``层的方法，写在``jni``中，打包为``.so``文件供``java``层调用。

总结一下，``java``层的功能就是找到补丁文件，根据补丁中的注解找到将要替换的方法然后交给jni层去处理替换方法的操作。好了，继续往下看。

### Native层

在``jni``的代码中支持``Dalvik``与``ART``，那么这是怎么区分的呢?在``AndFixManager``的构造方法中有这么一句

```java
mSupport = Compat.isSupport();
```

```java
public static synchronized boolean isSupport() {
	if (isChecked)
		return isSupport;

		isChecked = true;
		// not support alibaba's YunOs
		//SDK android 2.3 to android 6.0
		if (!isYunOS() && AndFix.setup() && isSupportSDKVersion()) {
			isSupport = true;
		}
	return isSupport;
}
```

``AndFix``的`setUp()``方法

```java
public static boolean setup() {
	try {
		final String vmVersion = System.getProperty("java.vm.version");
		//判断是否是ART
		boolean isArt = vmVersion != null && vmVersion.startsWith("2");
		int apilevel = Build.VERSION.SDK_INT;
		//这里也是native方法
		return setup(isArt, apilevel);
	} catch (Exception e) {
		Log.e(TAG, "setup", e);
		return false;
	}
}
```
最后调用``setup(isArt, apilevel);``的``native``方法，在``andfix.cpp``中注册``jni``方法

```c++
static JNINativeMethod gMethods[] = {
/* name, signature, funcPtr */
{ "setup", "(ZI)Z", (void*) setup }, 
{ "replaceMethod",
		"(Ljava/lang/reflect/Method;Ljava/lang/reflect/Method;)V",(void*) replaceMethod },
{ "setFieldFlag",
		"(Ljava/lang/reflect/Field;)V", (void*) setFieldFlag }, };
```

``native``实现

```c++
static jboolean setup(JNIEnv* env, jclass clazz, jboolean isart,
		jint apilevel) {
	isArt = isart;
	LOGD("vm is: %s , apilevel is: %i", (isArt ? "art" : "dalvik"),
			(int )apilevel);
	if (isArt) {
		return art_setup(env, (int) apilevel);
	} else {
		return dalvik_setup(env, (int) apilevel);
	}
}

static void replaceMethod(JNIEnv* env, jclass clazz, jobject src,
		jobject dest) {
	if (isArt) {
		art_replaceMethod(env, src, dest);
	} else {
		dalvik_replaceMethod(env, src, dest);
	}
}
```
根据上层传过来的``isArt``判断调用``Dalvik``还是``Art``的方法。
以``Dalvik``为例,继续往下分析,代码在``dalvik_method_replace.cpp``中
``dalvik_setup``方法

```c++
extern jboolean __attribute__ ((visibility ("hidden"))) dalvik_setup(
		JNIEnv* env, int apilevel) {
	jni_env = env;
	void* dvm_hand = dlopen("libdvm.so", RTLD_NOW);
	if (dvm_hand) {
		...
		//使用dlsym方法将dvmCallMethod_fnPtr函数指针指向libdvm.so中的		//dvmCallMethod方法，也就是说可以通过调用该函数指针执行其指向的方法
		//下面会用到dvmCallMethod_fnPtr
		dvmCallMethod_fnPtr = dvm_dlsym(dvm_hand,
			apilevel > 10 ?
			"_Z13dvmCallMethodP6ThreadPK6MethodP6ObjectP6JValuez" :
			"dvmCallMethod");
		...
		}
}

```
替换方法的关键在于``native``层怎么影响内存里的java代码，我们知道``java``代码里将一个方法声明为``native``方法时,对此函数的调用就会到``native``世界里找，AndFix原理就是将一个不是native的方法修改成native方法，然后在``native``层进行替换，通过``dvmCallMethod_fnPtr``函数指针来调用``libdvm.so``中的``dvmCallMethod()``来加载替换后的新方法，达到替换方法的目的。``Jni``反射调用``java``方法时要用到一个``jmethodID``指针,这个指针在``Dalvik``里其实就是``Method``类,通过修改这个类的一些属性就可以实现在运行时将一个方法修改成``native``方法。

看下``dalvik_replaceMethod(env, src, dest);``

```c++
extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
	jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(
			dvmThreadSelf_fnPtr(), clazz);
	//设置为初始化完毕
	clz->status = CLASS_INITIALIZED;
	//meth是将要被替换的方法
	Method* meth = (Method*) env->FromReflectedMethod(src);
	//target是新的方法
	Method* target = (Method*) env->FromReflectedMethod(dest);
	LOGD("dalvikMethod: %s", meth->name);

	meth->jniArgInfo = 0x80000000;
	//修改method的属性，将meth设置为native方法
	meth->accessFlags |= ACC_NATIVE;

	int argsSize = dvmComputeMethodArgsSize_fnPtr(meth);
	if (!dvmIsStaticMethod(meth))
		argsSize++;
	meth->registersSize = meth->insSize = argsSize;
	//将新的方法信息保存到insns
	meth->insns = (void*) target;
	//绑定桥接函数，java方法的跳转函数
	meth->nativeFunc = dalvik_dispatcher;
}

static void dalvik_dispatcher(const u4* args, jvalue* pResult,
		const Method* method, void* self) {
		
	Method* meth = (Method*) method->insns;
	meth->accessFlags = meth->accessFlags | ACC_PUBLIC;
	if (!dvmIsStaticMethod(meth)) {
		Object* thisObj = (Object*) args[0];
		ClassObject* tmp = thisObj->clazz;
		thisObj->clazz = meth->clazz;
		argArray = boxMethodArgs(meth, args + 1);
		if (dvmCheckException_fnPtr(self))
			goto bail;

		dvmCallMethod_fnPtr(self, (Method*) jInvokeMethod,
				dvmCreateReflectMethodObject_fnPtr(meth), &result, thisObj,
				argArray);

		thisObj->clazz = tmp;
	} else {
		argArray = boxMethodArgs(meth, args);
		if (dvmCheckException_fnPtr(self))
			goto bail;

		dvmCallMethod_fnPtr(self, (Method*) jInvokeMethod,
				dvmCreateReflectMethodObject_fnPtr(meth), &result, NULL,
				argArray);
	}
	bail: dvmReleaseTrackedAlloc_fnPtr((Object*) argArray, self);
}
```
通过``dalvik_dispatcher``这个跳转函数完成最后的替换工作，到这里就完成了两个方法的替换，有问题的方法就可以被修复后的方法取代。ART的替换方法就不讲了，原理上差别不大。

## 总结
AndFix热补丁原理就是在``native``动态替换方法``java``层的代码，通过``native``层hook ``java``层的代码。
### 优点
* 因为是动态的，所以不需要重启应用就可以生效
* 支持ART与Dalvik
* 与multidex方案相比，性能会有所提升(Multi Dex需要修改所有class的class_ispreverified标志位，导致运行时性能有所损失)
* 支持新增加方法
* 支持在新增方法中新增局部变量
* 足够轻量，生成补丁文件简单
* 安全性够高，验证签名

### 缺点
* 因为是动态的，跳过了类的初始化，设置为初始化完毕，所以对于静态方法、静态成员变量、构造方法或者class.forname()的处理可能会有问题
* 不支持新增成员变量和修改成员变量
* 官方apkPatch工具不支持multidex,但是可以通过修改工具来达到支持multidex的目的
* 由于是在native层替换方法，某些缺心眼厂商可能会修改源生关键部分的native层实现，导致可能在某些特定ROM支持不够好

## 参考
* [注入安卓进程,并hook java世界的方法](http://bbs.pediy.com/showthread.php?t=186054)
* [Hook Java的的一个改进版本](http://blog.csdn.net/l173864930/article/details/39667355)
* [关于Android APP在线热修复bug方案的调研(一)(AndFix)](http://blog.csdn.net/xxooyc/article/details/50317455)
* [各大热补丁方案分析和比较](http://blog.zhaiyifan.cn/2015/11/20/HotPatchCompare/)


项目地址：[AndFix](https://github.com/alibaba/AndFix)，本文分析版本：[AndFix:0.3.1](https://github.com/alibaba/AndFix/tree/c68d9811bd756ee418fce761ca113376ec9c4e66)
