---
title: Android热补丁之Robust（二）自动化补丁原理解析 
date: 2018-05-28 19:47:06
tags: [Android, 热补丁]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/robust.png
---

在 Android 热补丁框架 Robust 中，几个重要的流程包括：

- 补丁加载过程
- 基础包插桩过程
- 补丁包生成过程

在上一篇文章[Android热补丁之Robust原理解析(一)](http://w4lle.com/2017/03/31/robust-0/)中，我们分析了前两个，补丁加载过程和基础包插桩过程，分析的版本为 ``0.3.2``。
该篇文章为该系列的第二篇文章，主要分析补丁自动化生成的过程，分析的版本为`0.4.82`。
<!-- more -->
时间跨度有点大...

系列文章:

- [Android热补丁之Robust原理解析(一)](http://w4lle.com/2017/03/31/robust-0/)
- [Android热补丁之Robust（二）自动化补丁原理解析](http://w4lle.com/2018/05/28/robust-1/)
- [Android热补丁之Robust（三）坑和解](http://w4lle.com/2018/05/29/robust-2/)

# 整体流程

首先在 Gradle 插件中注册了一个名为 AutoPatchTranform 的 Tranform 

```java
class AutoPatchTransform extends Transform implements Plugin<Project> {
@Override
void apply(Project target) {
    initConfig();
    project.android.registerTransform(this)
}

def initConfig() {
    NameManger.init();
    InlineClassFactory.init();
    ReadMapping.init();
    Config.init();
    ...
    ReadXML.readXMl(project.projectDir.path);
    Config.methodMap = JavaUtils.getMapFromZippedFile(project.projectDir.path + Constants.METHOD_MAP_PATH)
}
```

该类为插件的入口，实现了 GradlePlugin 并继承自 Transform，在入口处初始化配置并注册 Transform。配置主要是读取 Robust xml 配置、混淆优化后的 mapping 文件、插庄过程中生成的 methodsMap.robust 文件、初始化内联工厂类等等。
然后最主要的是`transform`方法

```java
@Override
void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
    project.android.bootClasspath.each {
        Config.classPool.appendClassPath((String) it.absolutePath)
    }
    def box = ReflectUtils.toCtClasses(inputs, Config.classPool)
    ...
    autoPatch(box)
    ...
}

def autoPatch(List<CtClass> box) {
    ...
    ReadAnnotation.readAnnotation(box, logger);
    if(Config.supportProGuard) {
        ReadMapping.getInstance().initMappingInfo();
    }

    generatPatch(box,patchPath);

    zipPatchClassesFile()
    executeCommand(jar2DexCommand)
    executeCommand(dex2SmaliCommand)
    SmaliTool.getInstance().dealObscureInSmali();
    executeCommand(smali2DexCommand)
    //package patch.dex to patch.jar
    packagePatchDex2Jar()
    deleteTmpFiles()
}
```

在 `transform` 方法中，使用 javassist API 把所有需要处理的类加载到待扫描队列中，然后调用`autoPatch`方法自动生成补丁。
在 `autoPatch`方法中，主要做了这么几件事情：

1. 读取被 @Add、@Modify、RobustModify.modify() 标注的方法或类并记录
2. 解析 mapping 文件并记录每个类和类中方法混淆前后对应的信息，其中方法存储的信息有：返回值，方法名，参数列表，混淆后的名字；字段存储的信息有：字段名，混淆后的名字
3. 根据得到的信息，`generatPatch` 方法实际生成补丁
4. 将生成的补丁class打包成jar包
5. jar -> dex
6. dex -> smali
7. 处理 smali，主要是处理 super 方法和处理混淆关系
8. smali -> dex
9. dex -> jar

1、2 比较好懂就不逐步分析了，主要看3；后面的5、6、7、8、9 都是为了 7 中处理 smali，所以只要搞懂smali处理就好了。下面我们分步来看。

# 生成补丁

主要逻辑在 `generatPatch` 方法

```java
def  generatPatch(List<CtClass> box,String patchPath){
...
    InlineClassFactory.dealInLineClass(patchPath, Config.newlyAddedClassNameList)
    initSuperMethodInClass(Config.modifiedClassNameList);
    //auto generate all class
    for (String fullClassName : Config.modifiedClassNameList) {
        CtClass ctClass = Config.classPool.get(fullClassName)
        CtClass patchClass = PatchesFactory.createPatch(patchPath, ctClass, false, NameManger.getInstance().getPatchName(ctClass.name), Config.patchMethodSignatureSet)
        patchClass.writeFile(patchPath)
        patchClass.defrost();
        createControlClass(patchPath, ctClass)
    }
    createPatchesInfoClass(patchPath);
    }
    ...
}
```

分为两个部分

## 逐步翻译

首先调用`InlineClassFactory.dealInLineClass(patchPath, Config.newlyAddedClassNameList)`识别被优化过的方法，这里的优化是泛指，包括被优化、内联、新增过的类和方法，具体的逻辑为扫描修改后的所有类和类中的方法，如果这些类和方法不在 mapping 文件中存在，那么可以定义为被优化过，其中包括`@Add`新增的类或方法。
然后调用`initSuperMethodInClass`方法识别修改后的所有类和类中的方法中，分析是否如包含 `super` 方法，如果有那么缓存下来。
然后调用 `PatchesFactory.createPatch` 反射翻译修改的类和方法，具体的实现在

```java
private CtClass createPatchClass(CtClass modifiedClass, boolean isInline, String patchName, Set patchMethodSignureSet, String patchPath) throws CannotCompileException, IOException, NotFoundException {
    //清洗需要处理的方法，略..

    CtClass temPatchClass = cloneClass(modifiedClass, patchName, methodNoNeedPatchList);
    
    JavaUtils.addPatchConstruct(temPatchClass, modifiedClass);
    CtMethod reaLParameterMethod = CtMethod.make(JavaUtils.getRealParamtersBody(), temPatchClass);
    temPatchClass.addMethod(reaLParameterMethod);

    dealWithSuperMethod(temPatchClass, modifiedClass, patchPath);

    for (CtMethod method : temPatchClass.getDeclaredMethods()) {
        //  shit !!too many situations need take into  consideration
        //   methods has methodid   and in  patchMethodSignatureSet
        if (!Config.addedSuperMethodList.contains(method) && reaLParameterMethod != method && !method.getName().startsWith(Constants.ROBUST_PUBLIC_SUFFIX)) {
            method.instrument(
                    new ExprEditor() {
                        public void edit(FieldAccess f) throws CannotCompileException {
                            ...
                                if (f.isReader()) { f.replace(ReflectUtils.getFieldString(f.getField(), memberMappingInfo, temPatchClass.getName(), modifiedClass.getName()));
                                } else if (f.isWriter()) {
                                    f.replace(ReflectUtils.setFieldString(f.getField(), memberMappingInfo, temPatchClass.getName(), modifiedClass.getName()));
                                }
                            ...
                        }


                        @Override
                        void edit(NewExpr e) throws CannotCompileException {
                            //inner class in the patched class ,not all inner class
                            ...
                                if (!ReflectUtils.isStatic(Config.classPool.get(e.getClassName()).getModifiers()) && JavaUtils.isInnerClassInModifiedClass(e.getClassName(), modifiedClass)) {
                                    e.replace(ReflectUtils.getNewInnerClassString(e.getSignature(), temPatchClass.getName(), ReflectUtils.isStatic(Config.classPool.get(e.getClassName()).getModifiers()), getClassValue(e.getClassName())));
                                    return;
                                }
                            ...

                        @Override
                        void edit(Cast c) throws CannotCompileException {
                            MethodInfo thisMethod = ReflectUtils.readField(c, "thisMethod");
                            CtClass thisClass = ReflectUtils.readField(c, "thisClass");

                            def isStatic = ReflectUtils.isStatic(thisMethod.getAccessFlags());
                            if (!isStatic) {
                                //inner class in the patched class ,not all inner class
                                if (Config.newlyAddedClassNameList.contains(thisClass.getName()) || Config.noNeedReflectClassSet.contains(thisClass.getName())) {
                                    return;
                                }
                                // static函数是没有this指令的，直接会报错。
                                c.replace(ReflectUtils.getCastString(c, temPatchClass))
                            }
                        }

                        @Override
                        void edit(MethodCall m) throws CannotCompileException {
                            ...
                                if (!repalceInlineMethod(m, method, false)) {
                                    Map memberMappingInfo = getClassMappingInfo(m.getMethod().getDeclaringClass().getName());
                                    if (invokeSuperMethodList.contains(m.getMethod())) {
                                        int index = invokeSuperMethodList.indexOf(m.getMethod());
                                        CtMethod superMethod = invokeSuperMethodList.get(index);
                                        if (superMethod.getLongName() != null && superMethod.getLongName() == m.getMethod().getLongName()) {
                                            String firstVariable = "";
                                            if (ReflectUtils.isStatic(method.getModifiers())) {
                                                //修复static 方法中含有super的问题，比如Aspectj处理后的方法
                                                MethodInfo methodInfo = method.getMethodInfo();
                                                LocalVariableAttribute table = methodInfo.getCodeAttribute().getAttribute(LocalVariableAttribute.tag);
                                                int numberOfLocalVariables = table.tableLength();
                                                if (numberOfLocalVariables > 0) {
                                                    int frameWithNameAtConstantPool = table.nameIndex(0);
                                                    firstVariable = methodInfo.getConstPool().getUtf8Info(frameWithNameAtConstantPool)
                                                }
                                            }
                                            m.replace(ReflectUtils.invokeSuperString(m, firstVariable));
                                            return;
                                        }
                                    }
                                    m.replace(ReflectUtils.getMethodCallString(m, memberMappingInfo, temPatchClass, ReflectUtils.isStatic(method.getModifiers()), isInline));
                                }
                            ...
                        }
                    });
        }
    }
    //remove static code block,pay attention to the  class created by cloneClassWithoutFields which construct's
    CtClass patchClass = cloneClassWithoutFields(temPatchClass, patchName, null);
    patchClass = JavaUtils.addPatchConstruct(patchClass, modifiedClass);
    return patchClass;
}
```

这段代码其实是这个插件的核心部分，总体来说就是将修改后的代码全部翻译成反射调用生成 xxxPatch 类。
我们先只关注`method.instrument()`这个方法，这个Javassist的API，作用是遍历方法中的代码逻辑，包括：

- FieldAccess，字段访问操作。分为字段读和写两种，分别调用`ReflectUtils.getFieldString`方法，将代码逻辑使用Javassist翻译成反射调用，然后替换。
- NewExpr，new 对象操作。也分为两种
    - 非静态内部类，调用`ReflectUtils.getNewInnerClassString`翻译成反射，然后替换
    - 外部类，调用`ReflectUtils.getCreateClassString`翻译成反射，然后替换
- Cast，强转操作。调用`ReflectUtils.getCastString`翻译成反射，然后替换
- MethodCall，方法调用操作。情况比较复杂，以下几种情形
    - lamda表达式，调用`ReflectUtils.getNewInnerClassString`生成内部类的方法并翻译成反射，然后替换
    - 修改的方法是内联方法，调用`ReflectUtils.getInLineMemberString`方法生成占位内联类`xxInLinePatch`，并在改类中把修改的方法翻译成反射，然后替换调用，这方法中又有一些其他情况判断，感兴趣的读者可以自行阅读
    - 如果是super方法，这个情况后面单独拎出来说
    - 正常方法，调用`ReflectUtils.getMethodCallString`方法翻译成反射，然后替换
- 生成补丁类并增加构造方法

请注意，以上所有方法和需要处理的方法都需要特别注意方法签名！

## 控制补丁行为

最后调用 `createControlClass(patchPath, ctClass)`、`createPatchesInfoClass(patchPath);`生成 PatchesInfoImpl、xxxPatchControl 写入补丁信息和控制补丁行为。

其中，`PatchesInfoImpl`中包含所有补丁类的一一对应关系，比如 `MainActivity -> MainActivityPatch`，不清楚的可以参考该系列的上一篇文章。生成的`xxxPatchControl`类用于生成`xxPatch`类，并判断补丁中的方法是否和methods.robust中的方法id匹配，如果匹配才会去调用补丁中方法。

至此整体流程基本梳理完成。后面会针对具体的复杂情况加以解析。

# this 如何处理

首先，在补丁类中 xxPatch 中，`this`指代的是xxPatch类的对象，而我们是想要的对象是被补丁的类的对象。

在`PatchesFactory.createPatchClass()`方法中

```java
CtMethod reaLParameterMethod = CtMethod.make(JavaUtils.getRealParamtersBody(), temPatchClass);
temPatchClass.addMethod(reaLParameterMethod);
    
public static String getRealParamtersBody() {
    StringBuilder realParameterBuilder = new StringBuilder();
    realParameterBuilder.append("public  Object[] " + Constants.GET_REAL_PARAMETER + " (Object[] args){");
    realParameterBuilder.append("if (args == null || args.length < 1) {");
    realParameterBuilder.append(" return args;");
    realParameterBuilder.append("}");
    realParameterBuilder.append(" Object[] realParameter = new Object[args.length];");
    realParameterBuilder.append("for (int i = 0; i < args.length; i++) {");
    realParameterBuilder.append("if (args[i] instanceof Object[]) {");
    realParameterBuilder.append("realParameter[i] =" + Constants.GET_REAL_PARAMETER + "((Object[]) args[i]);");
    realParameterBuilder.append("} else {");
    realParameterBuilder.append("if (args[i] ==this) {");
    realParameterBuilder.append(" realParameter[i] =this." + ORIGINCLASS + ";");
    realParameterBuilder.append("} else {");
    realParameterBuilder.append(" realParameter[i] = args[i];");
    realParameterBuilder.append(" }");
    realParameterBuilder.append(" }");
    realParameterBuilder.append(" }");
    realParameterBuilder.append("  return realParameter;");
    realParameterBuilder.append(" }");
    return realParameterBuilder.toString();
}
```
    
这段的作用是，在每个xxPatch补丁类中都插入一个`getRealParameter()`方法，反编译出来最终的结果：

```java
public Object[] getRealParameter(Object[] objArr) {
    if (objArr == null || objArr.length < 1) {
        return objArr;
    }
    Object[] objArr2 = new Object[objArr.length];
    for (int i = 0; i < objArr.length; i++) {
        if (objArr[i] instanceof Object[]) {
            objArr2[i] = getRealParameter((Object[]) objArr[i]);
        } else if (objArr[i] == this) {
            objArr2[i] = this.originClass;
        } else {
            objArr2[i] = objArr[i];
        }
    }
    return objArr2;
}
```

它的作用是，在每个xxPatch中使用的`this`，都转换成xx被补丁类的对象。其中的`originClass`就是补丁类的对象。


# super 如何处理

同`this`类似，xxPatch中调用 `super` 方法同样需要转为调用被补丁类中相关方法的`super`调用。

还是在`PatchesFactory.createPatchClass()`方法中有`dealWithSuperMethod(temPatchClass, modifiedClass, patchPath);`调用

```java
private void dealWithSuperMethod(CtClass patchClass, CtClass modifiedClass, String patchPath) throws NotFoundException, CannotCompileException, IOException {
    ...
    methodBuilder.append("public  static " + invokeSuperMethodList.get(index).getReturnType().getName() + "  " + ReflectUtils.getStaticSuperMethodName(invokeSuperMethodList.get(index).getName()) + "(" + patchClass.getName() + " patchInstance," + modifiedClass.getName() + " modifiedInstance," + JavaUtils.getParameterSignure(invokeSuperMethodList.get(index)) + "){");
    ...
    CtClass assistClass = PatchesAssistFactory.createAssistClass(modifiedClass, patchClass.getName(), invokeSuperMethodList.get(index));
    ...
    methodBuilder.append(NameManger.getInstance().getAssistClassName(patchClass.getName()) + "." + ReflectUtils.getStaticSuperMethodName(invokeSuperMethodList.get(index).getName())  + "(patchInstance,modifiedInstance");
    ...
    }
```

保留主要代码，根据方法签名生成了一个新的方法，以`staticRobust+methodName`命名，方法中调用以`RobustAssist`结尾的类中的同名方法，并调用 `PatchesAssistFactory.createAssistClass` 方法生成该类，这个类的父类是被补丁类的父类。

```java
PatchesAssistFactory.createAssistClass:
static createAssistClass(CtClass modifiedClass, String patchClassName, CtMethod removeMethod) {
    ...
    staticMethodBuidler.append("public static  " + removeMethod.returnType.name + "   " + ReflectUtils.getStaticSuperMethodName(removeMethod.getName())
            + "(" + patchClassName + " patchInstance," + modifiedClass.getName() + " modifiedInstance){");
                
    staticMethodBuidler.append(" return patchInstance." + removeMethod.getName() + "(" + JavaUtils.getParameterValue(removeMethod.getParameterTypes().length) + ");");
    staticMethodBuidler.append("}");
    ...
}
```

然后在遍历`MethodCall`过程中，处理方法调用

```java
m.replace(ReflectUtils.invokeSuperString(m, firstVariable));
...
def static String invokeSuperString(MethodCall m, String originClass) {
    ...
    stringBuilder.append(getStaticSuperMethodName(m.methodName) + "(this," + Constants.ORIGINCLASS + ",\$\$);");
    ...
}
```

传递的参数，patch、originClass（被补丁类对象）、方法实际参数列表。

反编译出的结果实际是这样的：

```java
MainFragmentActivity:
public void onCreate(Bundle bundle) {
    super.onCreate(bundle);
    ...
}

MainFragmentActivityPatch:
public static void staticRobustonCreate(MainFragmentActivityPatch mainFragmentActivityPatch, MainFragmentActivity mainFragmentActivity, Bundle bundle) {
    MainFragmentActivityPatchRobustAssist.staticRobustonCreate(mainFragmentActivityPatch, mainFragmentActivity, bundle);
}

MainFragmentActivityPatchRobustAssist:
public class MainFragmentActivityPatchRobustAssist extends WrapperAppCompatFragmentActivity {
    public static void staticRobustonCreate(MainFragmentActivityPatch mainFragmentActivityPatch, MainFragmentActivity mainFragmentActivity, Bundle bundle) {
        mainFragmentActivityPatch.onCreate(bundler);
    }
}
```

参数是按照实际的方法参数传进去的，最后调用了xxPatch.superMethod方法。但是这样也并没有实现`super`方法的转义啊，再往下看。

在处理smali过程中，有这么一段：

```java
private String invokeSuperMethodInSmali(final String line, String fullClassName) {
    ...
    result = line.replace(Constants.SMALI_INVOKE_VIRTUAL_COMMAND, Constants.SMALI_INVOKE_SUPER_COMMAND);
    ...
    result = result.replace("p0", "p1");
    ...
}
```


在处理之前，smali是长这样的：`invoke-invoke-virtual {p0, p2}, Lcom/meituan/robust/patch/SecondActivityPatch;->onCreate(Landroid/os/Bundle;)V`,
处理之后是这样的：`invoke-super {p1, p2}, Landroid/support/v7/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V`，意思就是本来是调用正常方法，现在转为调用super方法，并且把参数换了一下，把p0(补丁类对象)换成了p1（被补丁类对象），这样就完成了`super`的处理。反编译后最终结果：

```java
public class MainFragmentActivityPatchRobustAssist extends WrapperAppCompatFragmentActivity {
    public static void staticRobustonCreate(MainFragmentActivityPatch mainFragmentActivityPatch, MainFragmentActivity mainFragmentActivity, Bundle bundle) {
        super.onCreate(bundle);
    }
}
```

# 内联怎么处理

内联是个广义的概念，包括了混淆过程中的优化(修改方法签名、删除方法等)、内联。在上面的分析中处理zi方法基本也提到了，缺啥补啥：就是把内联掉的方法再补回来。
对于内联的方法，不能用`@Modify`注解标注，只能使用`RobustModify.modify()`标注，因为在基础包中方法都没了，打了l补丁方法也没用。

主要逻辑在遍历`MethodCall` -> `repalceInlineMethod()` -> `ReflectUtils.getInLineMemberString()`

```java
...
stringBuilder.append(" instance=new " + NameManger.getInstance().getInlinePatchName(method.declaringClass.name) + "(\$0);")
stringBuilder.append("\$_=(\$r)instance." + getInLineMethodName(method) + "(" + parameterBuilder.toString() + ");")
...
```

作用就是把内联掉的方法调用替换为InLine类中的新增方法。

结果就是这样的：
```java
public class Parent {
    private String first=null;
    //privateMethod被内联了
    // private void privateMethod(String fir){
    //    System.out.println(fir);
    //}
    public void setFirst(String fir){
        first=fir;
        Parent children=new Children();
        //children.privateMethod("Robust");
        //内联替换的逻辑
        ParentInline inline= new ParentInline(children);
        inline.privateMethod("Robust");
    }
}
public class ParentInline{
    private Parent children ;
    public ParentInline(Parent p){
       children=p;
    }
    //混淆为c
    public void privateMethod(String fir){
        System.out.println(fir);
    }
}
```

# 总结

Robust 的核心其实就是自动化生成补丁这块，至于插庄、补丁加载这些都是很好实现的，因为没有很多的特殊情况需要处理。
这篇文章主要分析了自动化补丁插件的主要工作流程，和一些特殊情况的处理，文章有限，当然还有很多特殊情况没有分析，这里只是提供一些分析源码的思路，碰到特殊情况可以按照这个思路排查解决问题。

就像代码中有一行注释，我觉得特别能概括自动化生成补丁的团队的心里路程，在此也再次感谢美团团队对开源做出的贡献。
```java
//  shit !!too many situations need take into  consideration
```

总体来说，Robust 坑是有的，但是它也是高可用性、高兼容性的热修复框架，尤其是在Android 系统开放性越来越收紧的趋势下，Robust  作为不 hook 系统 API 的热修复框架优势更加突出。虽然其中可能有一些坑，只要我们对原理熟悉掌握，才有信心能搞定这些问题。

下一篇文章，主要就讲下Robust与其他框架搭配使用出现的一些坑。


# 参考

- [Android热更新方案Robust开源，新增自动化补丁工具](https://tech.meituan.com/android_autopatch.html)
- [Javassist 使用指南（二）](https://www.jianshu.com/p/b9b3ff0e1bf8)
- [[转] 深入理解 Dalvik 字节码指令及 Smali 文件](https://juejin.im/entry/579ef6e37db2a2005a6350d8)