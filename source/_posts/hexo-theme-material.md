---
title: Hexo Material主题 
date: 2016-11-17 10:41:20
tags: [Hexo]
---



# Material

最近把博客主题换成了[Material][1]，该主题刚刚上线，效果还不错。

需要把以前的配置迁移一下，包括留言之类的。

# 多说评论

由于之前用的NexT主题，多说后台每篇文章的thread-key是根据NexT主题来的，现在Material主题的多说配置文件如下：

```java
<div id="comments">
    <!-- 多说评论框 start -->
        <div class="ds-thread" data-thread-key="<% if(theme.comment.duoshuo_thread_key == "id"){ %><%= page.id %><% } else { %><%= page.path %><% } %>" data-url="<%= config.url+ config.root + url_for(path) %>"></div>
    <!-- 多说评论框 end -->
</div>
```
duoshuo_thread_key配成了page.title，就导致我们找不到多说后台的数据。

所以要找到NexT主题相关多说的配置。

```java
cd themes/next/layout
grep -ri "thread-key" ./*
```

结果：

```java
./_macro/post.swig:                  <span class="post-comments-count ds-thread-count" data-thread-key="{{ post.path }}" itemprop="commentsCount"></span>
./_partials/comments.swig:      <div class="ds-thread" data-thread-key="{{ page.path }}"
./_partials/share/duoshuo_share.swig:<div class="ds-share flat" data-thread-key="{{ page.path }}"
```

可以看到NexT配置的是page.path。那么我们找到Material主题的thread-key替换下就好了。主要就是两个地方:

 - ./layout/_widget/duoshuo.ejs 文章评论内容
 - ./layout/_partial/Paradox-post_entry.ejs 首页显示评论数

都修改下就能正常显示以前的评论了。

# 每日一图

Material主题首页有个每日图片，默认是写死的。我们可以改为动态的，使用了bing的每日图片，是一位开发者抓取的然后提供的一个api，地址：[必应美图 API][2]

使用：

```java
./_partial/daily_pic.ejs
<div class="mdl-card__media mdl-color-text--grey-50" style="background-image:url(https://api.i-meto.com/bing)">
```

# backgroud

同上背景图也可以设置成bing的图片。

```java
config_css.ejs
<% if(theme.background.bing.enable == true){ %>
	<style>
		body{
            background-image: url(https://api.i-meto.com/bing?<%= theme.background.bing.parameter %>);
        }
	</style>
```

config配置文件

```java
# Background Settings
# bing available parameter:
#     new | color= | type=
#         color available value: black, blue, brown, green, multi, orange, pink, purple, red, white, yellow
#         type available value: A (animal), C (culture), N (nature), S (space), T (travel)
background:
    purecolor: "#F5F5F5"
    bgimg: blur
    bing:
        parameter: color=white
        enable: true
```

# 不算子统计

站点统计
```java
footer.ejs
		<div>
		<script async src="https://dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js"></script>
		<div>
		本站总访问量 <span id="busuanzi_value_site_pv"></span> &nbsp&nbsp&nbsp
		你是第<span id="busuanzi_value_site_uv"></span>个来到的小伙伴
		</div>
		</div>
```

# 文章摘要

有两种方式设置：
 1. 写死固定大小
 2. more标签
 ```java
 以上是摘要
 <!--more-->
 以下是余下全文
 ```

推荐第二种



  [1]: https://github.com/viosey/hexo-theme-material
  [2]: https://i-meto.com/bing-api/