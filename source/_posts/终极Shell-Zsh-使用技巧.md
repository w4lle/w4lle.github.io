---
title: 终极Shell--Zsh 使用技巧
date: 2016-02-01 01:51:36
tags:
---
## 为什么要用Zsh
请参考这篇文章[终极 Shell](http://macshuo.com/?p=676)

## Zsh使用技巧
### 巧用``tab``
#### 自动完成
``tab``用到最多的就是自动完成,比如``cd``进入某个目录，可以输入该目录的中的几个字母，然后``tab``自动补全。
你不必输入整个目录名称，只需输入初始几个可以唯一区别与其他目录的字母，Zsh会自动匹配出剩余部分。
![此处输入图片的描述](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/QQ20160131-1%402x.png)
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/tab.png)
#### 环境变量展开
在Zsh中，你可以按下<TAB>键来展开shell中的环境变量值
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/PWD-tab.png)
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/PWD-tab-done.png)
#### kill命令补全
通常我们想要杀死某个进程，一般都要先``ps``下查看进程，然后``kill``杀掉。使用``zsh``可以这样
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/kill_tab.png)
#### help命令
对于我们不熟悉的命令行，一般都会``--help``查看帮助文档，而zsh可以直接敲你想要的命令，比如这样
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/git-tab.png)

### 强大的历史记录
类UNIX系统通常都习惯于``ctrl+r``的方式查找命令行的历史记录，挺好用的。但是``zsh``有更强大的历史搜索，比如``UP``
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/UP1.png)
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/UP2.png)
意思是上方向键能帮你找到最近使用的以``./gradlew``开头的命令，``UP`` ``DOWN``可以循环查找
### 强大的``alias``别名
平时工作基本都用``git``管理项目代码，每个人都有习惯使用的``git``别名，``zsh``为我们提供了一套通用的``alias``,即使换了工作环境，只要有``zsh``那么一套``alias``全部搞定。在该文件下可以看到``~/.oh-my-zsh/plugins/git/git.plugin.zsh``
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/alias_git.png)
除了``git``别名，还有好多有用的别名，``alias``命令可以列出全部的别名
![](https://raw.githubusercontent.com/w4lle/w4lle.github.io/post/source/uploads/alias.png)
### 智能跳转
首先需要安装插件``aotojump``，zsh会自动记录你访问过的目录，通过 ``j + 目录名``可以直接进行目录跳转，而且目录名支持模糊匹配和自动补全，例如你访问过``Develop``目录，输入``j develo`` 即可正确跳转。
