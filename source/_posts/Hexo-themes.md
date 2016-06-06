---
title: Hexo主题同步
date: 2016-06-06 18:48:22
tags: [Hexo]
---

# 问题

由于升级``Node.js``导致要重新安装``Hexo``，安装完新版的``Hexo``发现有些东西不能用，搞来搞去把本地仓库搞乱了，就想着把本地的仓库删掉吧，在从远程``clone``一份下来。
博客用的是[Next主题][1]，结果之前的主题配置``themes/next/``目录根本就没有上传到远程分支，导致主题所有的定制化修改全部丢失。此问题已经提交到[Next主题][3]的[Issues][2]中。

# 原因

先看下[Next主题][4]的使用方法
```
git clone https://github.com/iissnan/hexo-theme-next themes/next
```
这样配置完其实``thems/next/``就是一个包含``.git/``的子项目仓库。所以在``push``主项目的时候不会上传子项目，子项目的文件夹是灰的，并且里面是空的。如图
![此处输入图片的描述][5]

所以从远程仓库拉取的项目中是没有[Next主题][6]的。

# 解决

解决方法在[Issues][7]里也说了，用``fork + subtree``。

首先要``fork`` ``Next``，然后拉取到本地做修改，修改好后``push``到远程仓库。
然后用``git subtree``把``themes/next/``当做子项目来统一管理。``subtree``的用法可以看[使用GIT SUBTREE集成项目到子目录][8]。

具体步骤：

 1. 删除``themes/next/``并``push``到远程
    
    ```java
      rm -rf themes/next
      gst
      ga .
      git add --all
      gcmsg "delete next"
      ggpush
    ```
 2. 绑定子项目
    
    ```java
        git remote add -f next git@github.com:w4lle/hexo-theme-next.git
        git subtree add --prefix=themes/next next master --squash
    ```
 3. 更新子项目
    
    ```java
        git fetch next master
        git subtree pull --prefix=themes/next next master --squash
    ```
 4. 从子目录push到远程仓库
    
    ```java
        git subtree push --prefix=themes/next next master
    ```

现在再去远程仓库看``themes/next/``就有内容了，并且跟子项目的远程仓库可以保持更新，在主项目中修改也可以``push``到子项目的远程。再也不用担心主题配置丢失了。

# 参考

* [使用GIT SUBTREE集成项目到子目录][8]
* [hexo用subtree同步主题](http://tidus.xyz/2016/01/29/hexo%E7%94%A8subtree%E5%90%8C%E6%AD%A5%E4%B8%BB%E9%A2%98/)


  [1]: https://github.com/iissnan/hexo-theme-next
  [2]: https://github.com/iissnan/hexo-theme-next/issues/932
  [3]: https://github.com/iissnan/hexo-theme-next
  [4]: https://github.com/iissnan/hexo-theme-next
  [5]: https://cloud.githubusercontent.com/assets/7310246/15810485/b86f642a-2bd1-11e6-9bea-c8c603e9ba35.png
  [6]: https://github.com/iissnan/hexo-theme-next
  [7]: https://github.com/iissnan/hexo-theme-next/issues/932
  [8]: http://aoxuis.me/post/2013-08-06-git-subtree