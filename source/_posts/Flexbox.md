---
title: 前端基础(二)--快速布局神器Flexbox布局
date: 2016-05-08 20:38:35
tags: [前端, Flexbox]
---

# 什么是Flexbox

在[上一篇文章](http://w4lle.github.io/2016/04/17/%E5%89%8D%E7%AB%AF%E5%9F%BA%E7%A1%80-%E4%B8%80-CSS%E5%B8%83%E5%B1%80%E5%9F%BA%E7%A1%80/)中，我们知道CSS布局传统的布局方式是基于盒模型，主要依赖 display属性 + position属性 + float属性。这种盒模型对于一些复杂的布局解决起来比较麻烦，所以一种新的布局方式应运而生。
2009年，W3C提出了一种新的方案--Flexbox布局(弹性布局)，可以简便、完整、响应式地实现各种页面布局。Flex布局模型不同于块和内联模型布局，块和内联模型的布局计算依赖于块和内联的流方向。
并且React Native也是使用的Flex布局，对于客户端开发来说学习Flex大有裨益。

# 基本概念和属性

``Flexbox``布局依赖于``flex directions``，简单的说：``Flexbox``是布局模块，而不是一个简单的属性。采用Flex布局的元素，称为``Flex容器``（flex container），它的所有``子元素``（flex item）自动成为容器成员。
``Flexbox``布局很像``Android``开发中的``LinearLayout``布局，但是要比``LinearLayout``要强大。跟``LinearLayout``类似，``Flexbox``也存在两个方向的布局--主轴（main axis）和副轴（cross axis），可以简单的理解为``LinearLayout``的水平布局和垂直布局。主轴的开始位置（与边框的交叉点）叫做``main start``，结束位置叫做``main end``；交叉轴的开始位置叫做``cross start``，结束位置叫做``cross end``。项目默认沿主轴排列。单个项目占据的主轴空间叫做``main size``，占据的交叉轴空间叫做``cross size``。
![image](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071004.png)

# 属性

属性分为``flex container``属性和``flex item``属性。对应父容器和子元素。

## flex container属性

以下几个属于``flex container``属性

 * flex container 属性
 * order
 * flex-wrap
 * flex-flow
 * justify-content
 * align-content
 * align-items

### flex-direction属性

flex-direction属性决定主轴的方向（即项目的排列方向）。类似``LinearLayout``的``vertical``or``Horizontal``。

```css
flex-direction: row | row-reverse | column | column-reverse
```

有四个值可以选择：

* row（默认值）：主轴为水平方向，起点在左端。
* row-reverse：主轴为水平方向，起点在右端。
* column：主轴为垂直方向，起点在上沿。
* column-reverse：主轴为垂直方向，起点在下沿。

### order属性

默认情况下子元素的排列方式按照文档流的顺序依次排序，而``order``属性可以控制排列的顺序，负值在前，正值灾后，按照从小到大的顺序依次排列。我们说之所以``flex box``相对``LinearLayout``强就是因为一些属性比较给力，``order``就是其中之一。

![image](https://sfault-image.b0.upaiyun.com/318/287/3182877863-56fde345ba339_articlex)

### flex-wrap属性

默认情况下``Flex``跟``LinearLayout``一样，都是不带换行排列的，但是``flex-wrap``属性可以支持换行排列。这个也比``LinearLayout``吊啊有三个值：

```
flex-wrap: nowrap | wrap | wrap-reverse;
```

![image](https://sfault-image.b0.upaiyun.com/159/596/15959628-56fde3489ff74_articlex)

* nowrap ：不换行
* wrap：按正常方向换行
* wrap-reverse：按反方向换行

### flex-flow属性

``flex-flow``属性是``flex-direction``属性和``flex-wrap``属性的简写形式，默认值为``row nowrap``。

```
flex-flow: <flex-direction> || <flex-wrap>;
```

### justify-content属性

``justify-content``属性定义了项目在主轴上的对齐方式。

```
justify-content: flex-start | flex-end | center | space-between | space-around;
```

* flex-start（默认值）：左对齐
* flex-end：右对齐
* center： 居中
* space-between：两端对齐，项目之间的间隔都相等。
* space-around：每个项目两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍。

![image](https://sfault-image.b0.upaiyun.com/421/275/4212753174-56fde6083157c_articlex)

### align-items属性
``align-items``属性定义项目在副轴轴上如何对齐。

```
align-items: flex-start | flex-end | center | baseline | stretch;
```

* flex-start：交叉轴的起点对齐。
* flex-end：交叉轴的终点对齐。
* center：交叉轴的中点对齐。
* baseline: 项目的第一行文字的基线对齐。
* stretch（默认值）：如果项目未设置高度或设为auto，将占满整个容器的高度。

![image](http://img.blog.csdn.net/20150616152600533)

### align-content属性

``align-content``属性定义了多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用。

```
align-content: flex-start | flex-end | center | space-between | space-around | stretch;
```

* flex-start：与交叉轴的起点对齐。
* flex-end：与交叉轴的终点对齐。
* center：与交叉轴的中点对齐。
* space-between：与交叉轴两端对齐，轴线之间的间隔平均分布。
* space-around：每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍。
* stretch（默认值）：轴线占满整个交叉轴。

## 子元素属性

以下几个属性设置在子元素上。

* flex-grow
* flex-shrink
* flex-basis
* flex
* align-self

### flex-grow属性

``flex-grow``属性定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大。一张图看懂。跟``LinearLayout``中的``weight``属性一样。

```
.item {
  flex-grow: <number>; /* default 0 */
}
```

![image](https://sfault-image.b0.upaiyun.com/271/846/2718468848-56fde60740335_articlex)

如果所有项目的``flex-grow``属性都为1，则它们将等分剩余空间（如果有的话）。如果一个项目的``flex-grow``属性为2，其他项目都为1，则前者占据的剩余空间将比其他项多一倍。

### flex-shrink属性

``flex-shrink``属性定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小。

```
.item {
  flex-shrink: <number>; /* default 1 */
}
```

如果所有项目的``flex-shrink``属性都为1，当空间不足时，都将等比例缩小。如果一个项目的``flex-shrink``属性为0，其他项目都为1，则空间不足时，前者不缩小。
负值对该属性无效。

### flex-basis属性

``flex-basis``属性定义了在分配多余空间之前，子元素占据的``main size``主轴空间，浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为``auto``，即子元素的本来大小。

```
.item {
  flex-basis: <length> | auto; /* default auto */
}
```

### flex属性

``flex``属性是``flex-grow``, ``flex-shrink`` 和 ``flex-basis``的简写，默认值为0 1 auto。后两个属性可选。

```
.item {
  flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]
}
```

该属性有两个快捷值：auto (1 1 auto) 和 none (0 0 auto)。

### align-self属性

``align-self``属性允许单个子元素有与其他子元素不一样的对齐方式，可覆盖``align-items``属性。默认值为``auto``，表示继承父元素的``align-items``属性，如果没有父元素，则等同于``stretch``。

```
.item {
  align-self: auto | flex-start | flex-end | center | baseline | stretch;
}
```
该属性可能取6个值，除了``auto``，其他都与``align-items``属性完全一致。


# 实例

属性基本都讲完了，下面进入实战。
百度前端学院的其中一个任务：

* 学习如何flex进行布局，学习如何根据屏幕宽度调整布局策略。
* 屏幕宽度小于 640px 时，调整 Flexbox 的属性以实现第四个元素移动到最前面的效果，而不要改动第一个元素的边框颜色与高度实现效果图。

[效果图](http://7xrp04.com1.z0.glb.clouddn.com/task_1_10_1.png)

分析：
1. 屏幕宽度小于640px时，调整主轴对齐方式justify-content属性为space-between，在副轴对齐方式align-items为center
2. 屏幕宽度大于640px，要有换行，并且动态调整``order``属性，调整第四个子元素的排列位置。并且调整align-items为flex-start;

[效果图](http://w4lle.github.io/ife_baidu/task10/index.html)和[代码](https://github.com/w4lle/ife_baidu/tree/master/task10)

# Android开发者的福音


大概在一个月前，Google开源了[flexbox-layout](https://github.com/google/flexbox-layout)项目，用以支持``Flexbox``布局在Android开发中使用，支持源生的``Flexbox``属性。
官方例子，在xml布局文件中使用

```xml
<com.google.android.flexbox.FlexboxLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:flexWrap="wrap"
    app:alignItems="stretch"
    app:alignContent="stretch" >

    <TextView
        android:id="@+id/textview1"
        android:layout_width="120dp"
        android:layout_height="80dp"
        app:layout_flexBasisPercent="50%"
        />

    <TextView
        android:id="@+id/textview2"
        android:layout_width="80dp"
        android:layout_height="80dp"
        app:layout_alignSelf="center"
        />

    <TextView
        android:id="@+id/textview3"
        android:layout_width="160dp"
        android:layout_height="80dp"
        app:layout_alignSelf="flex_end"
        />
</com.google.android.flexbox.FlexboxLayout>
```

在java代码中使用

```java
FlexboxLayout flexboxLayout = (FlexboxLayout) findViewById(R.id.flexbox_layout);
flexboxLayout.setFlexDirection(FlexboxLayout.FLEX_DIRECTION_COLUMN);

View view = flexboxLayout.getChildAt(0);
FlexboxLayout.LayoutParams lp = (FlexboxLayout.LayoutParams) view.getLayoutParams();
lp.order = -1;
lp.flexGrow = 2;
view.setLayoutParams(lp);
```

可以看到使用非常方便，对于熟悉``Flexbox``的开发者来说对于``Android``app也可以快速上手。
Google开发这个开源库我猜想可能一方面看到``React Native``使用``Flexbox``对于前端工程师开发Android app布局的无缝切换；另一方面，``Flexbox``也确实要比Android开发中经常使用的``LinearLayout``和``RelativeLayout``要方便很多，灵活性较两者有大幅提高，特别在动态变化这一块。

# 总结

``Flexbox``布局方式相比传统的盒模型布局方式要快速很多，对于一些复杂的页面也可以很快速的开发。而且由于Google和Facebook的支持和使用，相信会有越来越多的开发者使用这种布局方式进行开发，在跨平台开发框架越来越成熟的现在，``Flexbox``赶紧学起来！
