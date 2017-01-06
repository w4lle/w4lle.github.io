---
title: 前端基础(零)--CSS基础
date: 2016-03-19 22:52:40
tags: [前端]
---

# 背景

最近跟小伙伴们在[百度前端技术学院](ife.baidu.com)学习前端相关的知识，接下来一系列博客会记录学习的过程和知识点，这是第一篇文章CSS基础。

开发工具：sublime text 3

主要插件：[Emmet](http://docs.emmet.io/)

# CSS

## 什么是CSS

> 层叠样式表 (Cascading Style Sheets)，常缩写为 CSS， 是一种 样式表 
> (stylesheet) 语言，用来描述 HTML、XML（包括各种 XML 语言如 SVG、XHTML）文档
> 的呈现。CSS 描述怎样在屏幕上渲染结构化元素。

## 为什么使用CSS
我们知道，html文件主要携带内容，虽然html也可以进行布局相关的操作，但是当浏览器的默认渲染规则不能很好的将我们定义好的布局展示出来，可能会导致显示不统一的问题，所以就将内容和表现进行分离，由CSS专门负责变现，html负责携带内容，可以很好的解除耦合，方便扩展，例如，需要全局的更新界面布局，那么也只需要修改css。可以说，css极大的提高了工作效率。就有点类似于android开发中的style，只负责配置各种view相关的属性值，而xml声明就有些类似于html，只是声明ImageView代表一个特定的view，表现形式完全可以交给style去处理。但是，个人感觉由于css的选择器的存在，css要比android中的style高明许多。


# CSS基础语法

看图

![image](http://7xs23g.com1.z0.glb.clouddn.com/CSS.png)

示例

```css
body {
     font-family: Verdana, sans-serif;
     }
```
body元素的所有文字的字体都是Verdana。
CSS默认有集成的属性，也就是说默认情况下body下的所有字元素都会继承body的属性，就像android布局，父控件的属性也会它所有的字控件一样，例如在根布局设置``backgroud="#eee"``那么，该界面的基础色就是``#eee``, css也是这样的。

# 选择器

css的强大之处在于，选择器的可配置性，对于复杂的界面显示来说这点非常有用

## 选择器的分组

```css
h1,h2,h3,h4,h5,h6 {
	color: green;
  }
```
上面的代码表示，每个选择器都不相互产生影响，只是写在了一起而已。

## 派生选择器

> 派生选择器就是通过依据元素在其位置的上下文关系来定义样式，以达到使标记更加简洁的
> 目的。

```css
div p {
    font-style: italic;
    font-weight: normal;
  }
```
在上面的例子中，只有 div 元素中的 p 元素的样式为斜体字，无需为 strong 元素定义特别的 class 或 id，代码更加简洁。当然，单独出现的div 和 p并不会受到影响。

## id 选择器

> id 选择器可以为标有特定 id 的 HTML 元素指定特定的样式。以``#``来定义

示例


```css
#red {color:red;}
#green {color:green;}
```
html这样写
```css
<p id="red">这个段落是红色。</p>
<p id="green">这个段落是绿色。</p>
```
但是要注意，同一个html文件中同一个id只能使用一次，具体的原因请看[这里](http://www.w3school.com.cn/xhtml/xhtml_structural_02.asp)

## 类选择器

通过设置元素的 class 属性，可以为元素指定类名。与id不同的是，同一个class可以在同一个html文件中复用。以``.``来定义

示例

```css
.center {text-align: center}
```
html代码中这样用

```css
<h1 class="center">
	This heading will be center-aligned
</h1>

<p class="center">
	This paragraph will also be center-aligned.
</p>
```

## 基于关系的选择器

CSS有多种基于元素关系的选择器。通过它们我们可以更精确的选择元素。
常见的关系选择器

| 选择器 | 选择的元素 | 
| ------------ | ------------- | 
| A E | 任何是元素A的后代元素E (后代节点指A的子节点，子节点的子节点，以此类推)  | 
| A > E | 任何元素A的子元素  | 
|E:first-child | 任何元素的第一个子元素E |
|B + E | 任何元素B的下一个兄弟元素E |
|B ~ E | B元素后面的拥有共同父元素的兄弟元素E|


## 伪类选择器

CSS伪类（pseudo-class）是加在选择器后面的用来指定元素状态的关键字。

示例

```css
a:link {color: #FF0000}		/* 未访问的链接 */
a:visited {color: #00FF00}	/* 已访问的链接 */
a:hover {color: #FF00FF}	/* 鼠标移动到链接上 */
a:active {color: #0000FF}	/* 选定的链接 */
```
上面表示a链接的各种状态的表示。

## 属性选择器

属性选择器可以根据元素的属性及属性值来选择元素。
简单的示例
```css
*[title] {color:red;}
```
上面表示把包含标题（title）的所有元素变为红色
在css中 ``*``可以替代任何元素

上面所讲到的任意选择器，都可以通过组合的方式表达更复杂的元素间的关系，产生复杂的显示效果。

# 总结

css是前端的基础，对于界面显示和布局来说至关重要，特别是css的选择器，相互组合可以产生非常复杂的效果。

最后附上ife的提交任务二
[代码地址](https://github.com/w4lle/ife_baidu/blob/master/task2%2Findex.html)
[demo](http://w4lle.github.io/ife_baidu/task2/index.html)
