---
title: Android Color位移操作
date: 2016-05-22 17:53:01
tags: [Android]
---

# 背景

之前项目中有个改变颜色的透明度的需求，大概的意思是滑动动态的改变透明度，但是不可以改变``View``的透明度，必须改变颜色值得透明度，功能虽然是实现了，但是代码太ugly，所以重新写了一下，复习了下``Color``的相关知识和位移的操作。

# Color简介

我们知道``Color``的每个颜色通道（``channel``）的组成由8位（``bit``）为一个单位，即1个字节（``byte``），10进制的取值范围0~255，我们一般用16进制表示，以``0x``开头，取值范围在0x00~0xFF。

Android中颜色一般有两种表示方法

1. rgb：每8``bit``表示rgb中的一个颜色通道。
2. argb：相比上第一种表示方法多出8``bit``用于表示``alpha``通道
所以Color的表示是这样的：0xffff00ff。

题外话，讲到这里又想到了``Bitmap.Config``，熟悉``bitmap``操作的同学应该对此不陌生，创建``bitmap``时的配置参数。包括：

1. ALPHA_8：只有一个``alpha``颜色通道，图形参数由一个字节来表示,是一种8位的位图。
2. ARGB_4444：图形颜色通道argb分别占用4、4、4、4bit，也就是每个像素暂用2``byte``的内存空间，是一种16``bit``的视图。
3. ARGB_8888：同上，每个像素占用4``byte``的内存空间。是一种32``bit``的视图。
4. RGB_565：没有``alpha``颜色通道，占用16``bit``。
所以在创建视图的时候加入使用ARGB_8888，800*480的视图，那么大概占用的空间就可以算出来了：800*400*4byte/1024=1250kb。

# 实现

之前说了，我们的需求是提取出``alpha``值然后改变它的值，再生成一个新的``color``，为什么说之前的那种实现方式很丑呢，看代码：

```java
if (vertical >= 0) {
    float toolBarAlpha = vertical * 1.0f / ivTop.getHeight();
    if (toolBarAlpha > 1) {
        toolBarAlpha = 1;
    } else if (toolBarAlpha < 0) {
        toolBarAlpha = 0;
    }
    int tmp = (int) (0xff * toolBarAlpha);
    String alpha = Integer.toHexString(tmp);
    if (alpha.length() < 2) {
        alpha = "0" + alpha;
    }
    toolBarColor = Color.parseColor("#" + alpha + "ff4c4b");
    ((MainActivity) getActivity()).setToolBarColor(toolBarColor);
}
```
``toolBarAlpha``是一个变化系数，乘以0xff得到一个新的``alpha``值然后拼接字符串生成一个新的``color``。
想法是好的，但是实现起来太丑陋的，现在我们提供一种更好的实现方法

```java
    public static int getColorWithAlpha(int color, float ratio) {
        int newColor = 0;
        int alpha = Math.round(Color.alpha(color) * ratio);
        int r = Color.red(color);
        int g = Color.green(color);
        int b = Color.blue(color);
        newColor = Color.argb(alpha, r, g, b);
        return newColor;
    }
```

Android SDK已经给我们提供了提取颜色通道的方法，直接提取就好了。而我们关注的不仅仅是好看的代码，而是怎么实现的。

# Color组成

先看``color``的组成代码：

```java
public static int argb(int alpha, int red, int green, int blue) {
        return (alpha << 24) | (red << 16) | (green << 8) | blue;
    }
```

什么意思呢？一张图看懂

![color](http://images.cnitblog.com/blog/325852/201308/12233846-b676cad0e08e4e98a5cc5c85eb78155f.png)
r的二进制向左移动16位，r << 16，g的二进制向左移动8位，而b的二进制则不需要移位操作。如果需要``alpha``通道，那么a的二进制向左移动24位。

# 提取颜色

知道了怎么组成也就好容易理解提取颜色通道

 1. ``blue``：不需要移位，color&0xFF，由于高位还有其余通道值，所以高位要取0 &0xFF，剩余取blue。
 2. ``green``：右移8位，相当于移位到blue的位置，原blue已经移出，然后与0xff相，(color >> 8) & 0xFF。
 3. ``red``：原理同上 (color >> 16) & 0xFF;
 4. ``alpha``：由于``alpha``在最高位，``alpha``移位操作如下
 
```java
0xff000000,那其实二进制是
color :1111  1111  0000  0000  0000  0000  0000  0000

color >>24结果 :1111  1111  1111  1111  1111  1111  1111  1111

color >>>24结果:0000  0000  0000  0000  0000  0000  1111  1111      
```

右移如果最高位为1那么左侧都补1，这样就导致生成的结果有问题，所以我们要用无符号右移操作，color >>> 24，右移完高位都补0，所以也不用&0xff。

# 总结

简单来说，``Color``都是位操作，在计算机的世界里，位移操作是最快的运算方式，并且对于内存的占用相对于基本数据类型也要有优势，比如表示状态，我们一般声明几个int值，但是有没有想过一个int类型就要占4``byte``=32``bit``，如果用位操作的话32``bit``基本就够用了，可以节省一些内存。

熟练掌握位移操作对于开发效率来说也会有相应的提高。在Android的代码中也是频繁的用到位移操作运用，比如刚刚收的一些状态操作，color操作等等。

# 参考

* [位操作也疯狂](http://m.oschina.net/blog/104123)
* [Color](https://developer.android.com/reference/android/graphics/Color.html)