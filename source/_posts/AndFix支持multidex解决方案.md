---
title: AndFix支持Multidex的解决方案
date: 2016-03-13 21:42:11
tags: [AndFix, 热更新, multidex]
---


# 背景
在上一篇文章[Android热补丁之AndFix原理解析](http://w4lle.github.io/2016/03/03/Android%E7%83%AD%E8%A1%A5%E4%B8%81%E4%B9%8BAndFix%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)中我们提到，AndFix的补丁文件是补丁生成工具apkPatch生成的，补丁文件``.apatch``基于``dex diff``的原理生成，简单来说就是对比两个apk包中的dex文件，代码在``DexFileFactory``类中

```java
public static DexBackedDexFile loadDexFile(File dexFile, int api, boolean experimental) throws IOException
  {
    return loadDexFile(dexFile, "classes.dex", new Opcodes(api, experimental));
  }
```

上面可以看到，只提取了apk包中的classes.dex文件，对于支持现阶段基本上所有的项目都是基于multidex方案的，那么如果apk包中有多个dex文件的话，AndFix的补丁工具就不会生效了。

# 解决方案

## 自己重新写一个工具
补丁生成工具apkPatch是一个jar文件，但是阿里团队并没有开源它的具体实现，我们也只能通过反编译来分析它，所以如果要重新写一个的话，要根据反编译全部重新敲一遍代码，只在关键的部分修改代码来达到支持multidex的目的。但是这种方式很低级，效率太低。

## 反射

大家都知道，Java代码编译完会生成.class文件，就是一堆字节码。JVM会解释执行这些字节码，由于字节码的解释执行是在运行时进行的，那我们能否手工编写字节码，再由JVM执行呢？答案是肯定的。

在``com.euler.patch.diff.DexDiffer``类中，抽取dex文件并进行对比

```java
public DiffInfo diff(File newFile, File oldFile)
    throws IOException
  {
    DexBackedDexFile newDexFile = DexFileFactory.loadDexFile(newFile, 19, 
      true);
    DexBackedDexFile oldDexFile = DexFileFactory.loadDexFile(oldFile, 19, 
      true);
      ....
   }
```
修改为

```java
public DiffInfo diff(File newFile, File oldFile) throws IOException {

    	HashSet<DexBackedClassDef> newset = getClassSet(newFile);
    	HashSet<DexBackedClassDef> oldset = getClassSet(oldFile);
        DiffInfo info = DiffInfo.getInstance();

        boolean contains = false;
        Iterator<DexBackedClassDef> iter = newset.iterator();
        while (iter.hasNext()) {
            DexBackedClassDef newClazz = iter.next();
            Iterator<DexBackedClassDef> iter2 = oldset.iterator();
            contains = false;
            while (iter2.hasNext()) {
                DexBackedClassDef oldClazz = iter2.next();
                if (newClazz.equals(oldClazz)) {
                    compareField(newClazz, oldClazz, info);
                    compareMethod(newClazz, oldClazz, info);
                    contains = true;
                    break;
                }
            }
            if (!contains) {
                info.addAddedClasses(newClazz);
            }
        }
        return info;
    }
```

抽取dex的``getClassSet``方法

```java
   private HashSet<DexBackedClassDef> getClassSet(File apkFile) throws IOException{
    	ZipFile localZipFile = new ZipFile(apkFile);
    	Enumeration localEnumeration = localZipFile.entries();
    	HashSet<DexBackedClassDef> newset = new HashSet<DexBackedClassDef>();
    	while (localEnumeration.hasMoreElements()) {
			ZipEntry localZipEntry = (ZipEntry) localEnumeration.nextElement();
			//所有以.dex结尾的文件都会加载。这样就支持的multidex
			if (localZipEntry.getName().endsWith(".dex")) {
				DexBackedDexFile newDexFile = DexFileFactory.loadDexFile(apkFile, localZipEntry.getName(), 19, true);
				FixedSizeSet<DexBackedClassDef> newclasses = (FixedSizeSet) newDexFile.getClasses();
				mergeHashSet(newset, newclasses);
			}
		}
    	return newset;
    }
```
然后将``DexDiffer``这个类单独达成一个jar包``dexdiffer.jar``，用于替换``apkPatch.jar``包中的``DexDiffer``类。加载的时候动态替换。

新增一个main.jar包，``Main.java``类

```java
public class Main {

    public static void main(String[] args) {
        try {
            OriginLoader oloader = getOriginClassLoader(Main.class.getClassLoader());
            FixLoader floader = getFixClassLoader(Main.class.getClassLoader());
            oloader.otherClassLoder = floader;
            oloader.otherLoadClassName = "com.euler.patch.diff.DexDiffer";
            floader.otherClassLoder = oloader;
            Class mainClass = oloader.loadClass("com.euler.patch.Main");
            //通过反射得到apkpatch.jar中的main()方法
            Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
            mainMethod.setAccessible(true);
            //执行apkpatch.jar中的main()方法
            mainMethod.invoke(mainClass, (Object)(args));
        }

    private static OriginLoader getOriginClassLoader(ClassLoader parent) throws MalformedURLException {
        URL[] urls = new URL[] {};
        OriginLoader loader = new OriginLoader(urls, parent);
        String path = Main.class.getProtectionDomain().getCodeSource().getLocation().getPath();
        int index = path.lastIndexOf(File.separator) + 1;
        path = path.substring(0, index);
        path = path + "apkpatch.jar";
        loader.addJar(new File(path).toURI().toURL());
        return loader;
    }
    
    private static FixLoader getFixClassLoader(ClassLoader parent) throws MalformedURLException {
        URL[] urls = new URL[] {};
        FixLoader loader = new FixLoader(urls, parent);
        String path = Main.class.getProtectionDomain().getCodeSource().getLocation().getPath();
        int index = path.lastIndexOf(File.separator) + 1;
        path = path.substring(0, index);
        path = path + "dexdiffer.jar";
        loader.addJar(new File(path).toURI().toURL());
        return loader;
    }
}
```
工具的入口由``apkpatch.jar``的``main()``方法改为了``main.jar``的``main()``方法。所以在这里我们就可以动态替换相关的关键类和方法。

``OriginLoader``的方法``findClass``方法

```java
@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = null;
        //上面设置的oloader.otherLoadClassName = 			//"com.euler.patch.diff.DexDiffer";
        if(name.equals(otherLoadClassName)){
            return otherClassLoder.loadClass(name);
        }
        clazz = super.findClass(name);
        return clazz;
    }
```
也就是说``OriginLoader``在加载的时候如果类是``com.euler.patch.diff.DexDiffer``，那么就会调用``FixLoader``去加载我们刚刚生成的``dexdiff.jar``中的``DexDiffer``达到替换的目的。
``apkpatch.sh``

```java
PRG="$0"
while [ -h "$PRG" ] ; do
  ls=`ls -ld "$PRG"`
  link=`expr "$ls" : '.*-> \(.*\)$'`
  if expr "$link" : '/.*' > /dev/null; then
    PRG="$link"
  else
    PRG=`dirname "$PRG"`/"$link"
  fi
done
PRGDIR=`dirname "$PRG"`
//入口改为main.jar
java -Xms512m -Xmx1024m -jar $PRGDIR/main.jar "$@"
```

最后的jar包有4个，main.jar, dexdiffer.jar, apkpatch.jar
方法代码参考[这里](https://github.com/w4lle/andfix_apkpatch_support_multidex/blob/master/apkpatch.sh)

## javassist修改class文件
[javassist](https://github.com/jboss-javassist/javassist)其实就是一个二方包，提供了运行时操作Java字节码的方法。Javassist就提供了一些方便的方法，让我们通过这些方法生成字节码。最后也是通过反射调用修改后的方法。

使用方法

```java
	//Classpool负责用Javassist来控制字节码的修改
	ClassPool cp = ClassPool.getDefault();
	//获得类文件名
	CtClass cc = cp.get("com.euler.patch.diff.DexDiffer");
	//获得要修改的方法名
	CtMethod m = cc.getDeclaredMethod("diff");
	CtMethod.setBody(“这里是修改后的代码”); 
	CtClass.addMethod(ctMethod);  
    Class<?> c=CtClass.toClass();  
    Object o=c.newInstance();  
    Method method=o.getClass().getMethod("diff", new Class[]{});  
    //调用字节码生成类的diff方法  
    method.invoke(o, new Object[]{});  

```

生成``.class``文件，然后替换``apkpatch.jar``包中的``.class``文件。
这种方法只用一个jar包就可以完成生成补丁的操作。

# 最后一步

先看代码，``AndFixaManager``

```java
 public synchronized void fix(File file, ClassLoader classLoader,
                                 List<String> classes) {
        final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
                    optfile.getAbsolutePath(), Context.MODE_PRIVATE);

            if (saveFingerprint) {
                mSecurityChecker.saveOptSig(optfile);
            }

            ClassLoader patchClassLoader = new ClassLoader(classLoader) {
                @Override
                protected Class<?> findClass(String className)
                        throws ClassNotFoundException {
                        // 代码1
                    Class<?> clazz = dexFile.loadClass(className, this);
                    if (clazz == null
                            && className.startsWith("com.alipay.euler.andfix")) {
                        return Class.forName(className);// annotation’s class
                        // not found
                    }
                    
                    //代码2
                    //add fix w4lle for multidex surpport
                    if (clazz == null) {
                        return Class.forName(className);
                    }
                    //add end
                    
                    if (clazz == null) {
                        throw new ClassNotFoundException(className);
                    }
                    return clazz;
                }
            };
  }
 ```

代码1处通过``classLoader``去加载目标类，但是有一点要明确，一个运行的Android应用至少有2个``ClassLoader``，一个是``BootClassLoader``（系统启动的时候创建的），另外一个或多个是``PathClassLoader``用于加载dex，每个dex文件由一个独立的``PathClassLoader``去加载。也就是说如果目标类在dex2中，代码1是加载不了目标类的，所以会抛出``ClassNotFoundException``。就像这样

```java
Caused by: java.lang.ClassNotFoundException: Didn't find class "com.boohee.status.MsgCategoryActivity$2_CF" on path: DexPathList[[zip file "/data/app/com.boohee.one-1.apk", zip file "/data/data/com.boohee.one/code_cache/secondary-dexes/com.boohee.one-1.apk.classes2.zip"],nativeLibraryDirectories=[/data/app-lib/com.boohee.one-1, /vendor/lib, /system/lib]]
	at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:56)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:511)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:469)
```

所以我们在代码2处添加了保护代码，保证目标类可以被加载。由于andfix官方并没有做这个支持，所以就不能通过gradle依赖，就需要我们把andfix的源码放到项目中然后修改。


# 参考

* [Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)

项目地址：[AndFix](https://github.com/alibaba/AndFix)，本文分析版本：[AndFix:0.3.1](https://github.com/alibaba/AndFix/tree/c68d9811bd756ee418fce761ca113376ec9c4e66)