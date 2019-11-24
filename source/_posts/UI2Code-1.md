---
title: UI2Code（二）pixeltoapp
date: 2019-03-22 17:36:57
tags: [机器学习]
thumbnail: https://ws2.sinaimg.cn/large/006tKfTcly1g1bmgzbb4rj31o90u00z4.jpg
---

[pixeltoapp](http://pixeltoapp.com/) 是一个通过传统图像处理把屏幕截图转换为 Android 代码的项目，使用python实现，提供在线服务，具体实现在[项目源码地址](https://github.com/soumikmohianuta/pixtoapp)

<!--more-->

系列文章：

- [UI2Code（一）pix2code](http://w4lle.com/2019/03/13/UI2Code-0/)
- [UI2Code（二）pixeltoapp](http://w4lle.com/2019/03/22/UI2Code-1/)
- [UI2Code（三）imgcook](http://w4lle.com/2019/04/08/UI2Code-2/)

# 实现效果

![原图](https://ws2.sinaimg.cn/large/006tKfTcly1g1bjfidsnoj30ku1940wr.jpg) 

![代码渲染图](https://ws3.sinaimg.cn/large/006tKfTcly1g1bjglum9yj30ly118wgz.jpg) 

生成的布局代码：

```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
             android:layout_width="250.0dip"
             android:layout_height="541.3333333333334dip"
             android:layout_gravity="center_vertical|center_horizontal"
             android:background="@color/color_0">

    <ImageView
        android:id="@+id/ImageView_0"
        android:layout_width="10.333333333333334dip"
        android:layout_height="10.333333333333334dip"
        android:layout_marginLeft="74.33333333333333dip"
        android:layout_marginTop="499.3333333333333dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_0"/>

    <ImageView
        android:id="@+id/ImageView_1"
        android:layout_width="10.333333333333334dip"
        android:layout_height="10.0dip"
        android:layout_marginLeft="41.333333333333336dip"
        android:layout_marginTop="499.3333333333333dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_2"/>

    <ImageView
        android:id="@+id/ImageView_2"
        android:layout_width="31.666666666666668dip"
        android:layout_height="18.0dip"
        android:layout_marginLeft="198.0dip"
        android:layout_marginTop="491.3333333333333dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_5"/>

    <ImageView
        android:id="@+id/ImageView_3"
        android:layout_width="10.333333333333334dip"
        android:layout_height="10.333333333333334dip"
        android:layout_marginLeft="52.666666666666664dip"
        android:layout_marginTop="499.0dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_6"/>

    <ImageView
        android:id="@+id/ImageView_4"
        android:layout_width="9.333333333333334dip"
        android:layout_height="11.0dip"
        android:layout_marginLeft="188.0dip"
        android:layout_marginTop="498.6666666666667dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_7"/>

    <ImageView
        android:id="@+id/ImageView_5"
        android:layout_width="10.666666666666666dip"
        android:layout_height="11.0dip"
        android:layout_marginLeft="176.33333333333334dip"
        android:layout_marginTop="498.6666666666667dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_8"/>

    <ImageView
        android:id="@+id/ImageView_6"
        android:layout_width="10.666666666666666dip"
        android:layout_height="10.666666666666666dip"
        android:layout_marginLeft="165.33333333333334dip"
        android:layout_marginTop="498.6666666666667dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_9"/>

    <ImageView
        android:id="@+id/ImageView_7"
        android:layout_width="1.6666666666666667dip"
        android:layout_height="14.666666666666666dip"
        android:layout_marginLeft="124.33333333333333dip"
        android:layout_marginTop="496.6666666666667dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_10"/>

    <ImageView
        android:id="@+id/ImageView_8"
        android:layout_width="250.0dip"
        android:layout_height="1.6666666666666667dip"
        android:layout_marginLeft="0.0dip"
        android:layout_marginTop="489.0dip"
        android:scaleType="fitXY"
        android:src="@drawable/img_11"/>

    <FrameLayout
        android:id="@+id/FrameLayout_9"
        android:layout_width="229.66666666666666dip"
        android:layout_height="101.0dip"
        android:layout_marginLeft="10.0dip"
        android:layout_marginTop="383.3333333333333dip"
        android:background="@color/color_0">

        <ImageView
            android:id="@+id/ImageView_9"
            android:layout_width="22.0dip"
            android:layout_height="7.666666666666667dip"
            android:layout_marginLeft="70.0dip"
            android:layout_marginTop="78.33333333333333dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_12"/>

        <ImageView
            android:id="@+id/ImageView_10"
            android:layout_width="22.0dip"
            android:layout_height="7.666666666666667dip"
            android:layout_marginLeft="48.0dip"
            android:layout_marginTop="78.33333333333333dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_13"/>

        <ImageView
            android:id="@+id/ImageView_11"
            android:layout_width="7.0dip"
            android:layout_height="7.333333333333333dip"
            android:layout_marginLeft="40.666666666666664dip"
            android:layout_marginTop="78.33333333333333dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_14"/>

        <ImageView
            android:id="@+id/ImageView_12"
            android:layout_width="14.666666666666666dip"
            android:layout_height="7.333333333333333dip"
            android:layout_marginLeft="26.0dip"
            android:layout_marginTop="78.33333333333333dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_15"/>

        <ImageView
            android:id="@+id/ImageView_13"
            android:layout_width="14.666666666666666dip"
            android:layout_height="7.666666666666667dip"
            android:layout_marginLeft="11.333333333333334dip"
            android:layout_marginTop="78.33333333333333dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_16"/>

        <ImageView
            android:id="@+id/ImageView_14"
            android:layout_width="44.666666666666664dip"
            android:layout_height="20.0dip"
            android:layout_marginLeft="174.33333333333334dip"
            android:layout_marginTop="72.0dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_18"/>

        <ImageView
            android:id="@+id/ImageView_15"
            android:layout_width="49.0dip"
            android:layout_height="8.333333333333334dip"
            android:layout_marginLeft="24.333333333333332dip"
            android:layout_marginTop="40.333333333333336dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_21"/>

        <ImageView
            android:id="@+id/ImageView_16"
            android:layout_width="12.333333333333334dip"
            android:layout_height="7.0dip"
            android:layout_marginLeft="11.666666666666666dip"
            android:layout_marginTop="41.0dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_22"/>

        <ImageView
            android:id="@+id/ImageView_17"
            android:layout_width="8.666666666666666dip"
            android:layout_height="8.0dip"
            android:layout_marginLeft="210.33333333333334dip"
            android:layout_marginTop="40.666666666666664dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_23"/>

        <ImageView
            android:id="@+id/ImageView_18"
            android:layout_width="24.0dip"
            android:layout_height="8.333333333333334dip"
            android:layout_marginLeft="184.0dip"
            android:layout_marginTop="40.333333333333336dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_24"/>

        <ImageView
            android:id="@+id/ImageView_19"
            android:layout_width="8.0dip"
            android:layout_height="5.666666666666667dip"
            android:layout_marginLeft="210.66666666666666dip"
            android:layout_marginTop="29.0dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_25"/>

        <ImageView
            android:id="@+id/ImageView_20"
            android:layout_width="37.333333333333336dip"
            android:layout_height="9.333333333333334dip"
            android:layout_marginLeft="49.0dip"
            android:layout_marginTop="25.333333333333332dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_26"/>

        <ImageView
            android:id="@+id/ImageView_21"
            android:layout_width="37.333333333333336dip"
            android:layout_height="9.333333333333334dip"
            android:layout_marginLeft="11.333333333333334dip"
            android:layout_marginTop="25.333333333333332dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_27"/>

        <ImageView
            android:id="@+id/ImageView_22"
            android:layout_width="7.666666666666667dip"
            android:layout_height="12.666666666666666dip"
            android:layout_marginLeft="200.0dip"
            android:layout_marginTop="22.0dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_28"/>

        <ImageView
            android:id="@+id/ImageView_23"
            android:layout_width="4.333333333333333dip"
            android:layout_height="12.333333333333334dip"
            android:layout_marginLeft="193.33333333333334dip"
            android:layout_marginTop="22.0dip"
            android:scaleType="fitXY"
            android:src="@drawable/img_29"/>
    </FrameLayout>
    ...
```

# 实现原理

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bmgzbb4rj31o90u00z4.jpg) 

整体处理过程： 

原图 -> 灰度处理 - 降噪 - 边缘探测，提取边缘轮廓 - 膨胀处理 - 框选边缘轮廓 - 遍历轮廓构建ViewTree - 删除重复元素，合并Layout布局，梳理ViewTree - 判断控件类型，目前支持Text和Image - 切割图像、文本识别，绑定控件属性 - 构建XML布局 

整体分为三个大的过程：背景分析、前景分析和布局构建 

\- 背景分析：通过机器视觉算法，得到图像轮廓 

\- 前景分析：对轮廓碎片进行整理，合并，识别 

\- 布局构建：基于以上信息构建布局 

下面依次看下 

## 背景分析 

包含 灰度处理、降噪、边缘探测、膨胀处理、框选边缘轮廓等几个步骤 

### 灰度处理

二值化处理，作用是得到相对干净的背景底色 

```
img_gray = cv2.cvtColor(img_color, cv2.COLOR_BGR2GRAY)
```

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bkowh846j30ku194q53.jpg) 

### 降噪

```
cv2.fastNlMeansDenoising(img_gray)
```

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bkraa5qfj30ku194jtm.jpg) 

### 边缘探测

使用Canny进行边缘探测，Canny算子是一种经典的边缘检测算子，它能得到精确的边缘位置。

Canny检测的一般步骤为：

1. 用高斯滤波进行降噪
2. 用一阶偏导的有限差分计算梯度的幅值和方向 
3. 对梯度幅值进行非极大值抑制
4. 用双阈值检测和连接边缘。实验过程中，需要多次尝试选择较好的双阈值参数。 

```
cv2.Canny(imgData,self.lowThreshold,self.highThreshold)
```

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1bksbdqhgj30ku194dgj.jpg) #20-40 

### 形态学膨胀

检测出来的边缘在某些局部地方会断开，可以采用特定形状和尺寸的结构元素对二值化图像进行形态学膨胀处理来连接断开的边缘。 

使用大小为(3，3)的十字线进行膨胀处理 

```python
ratio =2; 

kernel = np.ones((2 * dilationSize + 1, 2 * dilationSize + 1), np.uint8) 

img_dilation = cv2.dilate(imgData, kernel, iterations=1) 
```

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1bkvi7ng4j30ku194t9b.jpg) 

### 框选轮廓

为了直观的看到轮廓形状，我们用红色把轮廓框起来，其中可以看到很多相同控件内的文字并没有联通，所以上一步的膨胀处理的参数仍然有调整空间 

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1bkz5o1mbj30ku194jyb.jpg) 

背景分析基本就这些，下面看前景分析。

## 前景分析

前景分析包括对轮廓碎片进行整理并构建ViewTree、ViewTree优化合并、控件识别。

### 构建ViewTree

得到轮廓参数后遍历操作，contours自带层级关系，按照该层级关系依次递归遍历，得到粗糙的ViewTree 

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1bl2vf5naj31lm0u0gv0.jpg) 

### ViewTree优化

包括如下几个方面：

- 删除重叠区域view 
- 合理划分父子关系 
- 合并无意义的Layout控件 

### 控件识别

到这一步就得到了每一个View的坐标属性及轮廓属性了，根据这些数据进行抠图并识别控件类型，**这里只支持两种控件**：TextView 和 ImageView，但是生成的代码只包含ImageView，猜测是因为OCR识别中文有问题，所以控件类型识别错误导致。 

- 控件识别 

- 属性提取，图像、文字、颜色 

这里作者还提供了ListView的实现，但是应该不是完整的，所以被注释掉了。

到这一步，根据上面的工作，就可以得到整个ViewTree的描述信息了，相当于得到了布局的DSL，但是这里并没有输出DSL，仅仅是在内存中数据结构的表现。

# 布局构建

背景分析和前景分析都做完之后，就可以根据以上信息进行布局生成工作 ：

- 根布局是FrameLayout 

- 子布局有两种FrameLayout和RelativeLayout 

- 控件都是相对父布局的绝对布局，layout_marginLeft，layout_marginTop 

- 代码中看 RelativeLayout 中有相对布局，不过看生成的代码没有看到 

这一步处理之后就可以得到布局代码了，生成布局代码后，这里也有一个compile的过程，根据提供的template工程，将布局代码带入，就可以得到可以运行的Android工程了。

# 总结

由于设计稿是iOS，宽度750px的设计稿，所以出来的布局代码在Android上是没有适配的，不过这个关系都不大。 

这个项目提供了通过传统的机器视觉图像算法生成布局的思路。但是也存在一些问题： 

- 对于复杂界面处理能力有限
- 传统图像处理的方式，对于不同场景阈值可能需要频繁的调整，典型的如膨胀参数
- 泛化能力较弱，支持控件类型太少，并且识别准确率确实不行
- 布局能力有限，基本都是绝对坐标，布局比较死板 

现在回过头来翻看闲鱼关于[版面分析的文章](https://www.jiqizhixin.com/articles/2019-02-27-14)，其中的主要流程跟pixtoapp基本一致，并且一些参数都是一样的，比如膨胀参数，猜测闲鱼团队应该也是参考过这个项目，并在这基础上做了一些优化。 

结合pix2code项目，pixtoapp进行版面分析，切割控件、得到布局，pix2code使用ML进行控件识别，最后进行组装，得到一个完整的布局，这种思路是可行的，猜测闲鱼也是基于这样的思路来做的。 

这种方案其中的一个最重要的点，也是最难的点，就是布局能力太弱，闲鱼文中提到的布局方式也是规则实现，那么就是类似pixtoapp中的实现方式，但是有一点不同的是，切割方式不同，所以闲鱼有row和col，而pixtoapp没有，RNN前置反馈这个还没有了解到，后面再看下。 

> 前期我们采用4层LSTM网络进行训练学习，由于样本量比较小，我们改为规则实现。规则实现也比较简单，我们在第一步切图时5刀切割的顺序就是row和col。缺点是布局比较死板，需要结合RNN进行前置反馈。 

通过两篇文章的分析，UI2Code的大体思路是通了的，但是核心点构建布局还没有很好的解决方案，还需要再详细思考下实现方案。 

参考 ：

[基于AI的移动端自动化测试框架的设计](https://www.jiqizhixin.com/articles/2018-12-29-5) 

[UI2Code智能生成Flutter代码——版面分析篇](https://www.jiqizhixin.com/articles/2019-02-27-14)