---
title: 前端基础(一)--CSS布局基础
date: 2016-04-17 22:26:23
tags: [CSS, 前端]
---

# 盒模型

CSS中， Box Model叫盒模型（或框模型），Box Model规定了元素框处理元素内容（element content）、内边距（padding）、边框（border） 和 外边距（margin） 的方式。这种方式基本类似于Android开发中的布局方式，所以对于Android developer学习前端布局方式可以很快的入门。但是有一点，在Android中设置margin和padding的顺序是left、top、right、bottom，比如``setMargin(10, 20, 30, 20)``分别代表左上右下的间距分别为10px,20px,30px,20px;但是在CSS中的顺序是top、right、bottom、left，比如``margin: [10px, 20px, 30px, 20px]``分别代表左上右下间距分别为20px,10px,20px,30px。
概述图如下

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/box-model.png)

# 定位基础

## ``position``定位

``position``包括以下几种类型的定位

* static 默认值。任意 position: static; 的元素不会被特殊的定位。一个 static 元素表示它不会被“positioned”，一个 position 属性被设置为其他值的元素表示它会被“positioned”。

* relative 相对布局，在原有基础上偏离使框偏离某个方向固定距离。跟Android中的布局方式很像
例子

```css
.relative2 {
  position: relative;
  top: -20px; //在原有top位置上向上偏移-20px
  left: 20px; //在原有left位置上向左偏移20px
  background-color: white;
  width: 500px;
}
```
* absolute 绝对布局，向上寻找使用过``position``定位过(除了默认值static外)的祖先元素，然后依据该元素进行定位。

```css
.relative {
  position: relative;
  width: 600px;
  height: 400px;
}
.absolute {
  position: absolute;
  top: 120px;
  right: 0;
  width: 300px;
  height: 200px;
}
```

如图

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/absolute.png)

* fixed 固定定位，相对于视窗来定位，这意味着即便页面滚动，它还是会停留在相同的位置。相当于在Android开发中``FrameLayout``中的某个元素指定``layout_gravity``使其固定在根布局的某个固定的位置。
例子

```css
.fixed {
  position: fixed;
  bottom: 0;
  right: 0;
  width: 200px;
  background-color: white;
}
```
该元素的位置始终在右下角保持不变。

## ``float``浮动

> 浮动的框可以向左或向右移动，直到它的外边缘碰到包含框或另一个浮动框的边框为止。

可以这样理解，比如``float: left``就是向左移动，直到坐边缘碰到根元素或者另外一个佛洞的边框的边缘。也就是说，如果好好几个向左浮动的元素，那么它们是从左到右依次排列的。
如下面的图

![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/float1.png)
![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/float2.png)
![image](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/float3.png)

## ``clear``清除浮动

clear 属性定义了元素的哪边上不允许出现浮动元素。
具体的例子参考[这里](http://zh.learnlayout.com/clear.html)

# 三栏式布局

ife其中的一个任务[三栏式布局](http://ife.baidu.com/task/detail?taskId=3)
就是通过CSS的布局基础知识来写的。包括position和float。

代码在这里[三栏式布局](https://github.com/w4lle/ife_baidu/tree/master/task3)
demo [demo](http://w4lle.github.io/ife_baidu/task3/task3.html)

# 参考

* [learnlayout](http://zh.learnlayout.com/)
* [w3school](http://www.w3school.com.cn/css/css_positioning.asp)