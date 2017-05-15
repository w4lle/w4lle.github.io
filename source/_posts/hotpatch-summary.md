---
title: 热修复总结
date: 2017-05-04 18:45:31
tags: [Android, 热补丁]
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/android.jpg
---

# 热修复总结

| 平台 | 阿里 AndFix | 阿里 HotFix1.x | Nuwa | 微信Tinker | 美团Robust | 阿里 HotFix2.x|
| :---: | :----: | :----: |:----: |:----: |:----: |:----: 	
| 即时生效 | yes | yes | no | no | yes | 看情况 | 
| 性能损耗 | 较小   | 较小  | 较大 | 较小 | 较小 |  较小 |
| 补丁包大小 | 一般 | 一般  | 较大 | 较小 | 较小 | 较小 | 
| 占Rom体积  | 较小   | 较小 | 一般 | 较大 | 较小 | 看情况| 
| 接入复杂度 | 简单 | 简单 | 简单 | 复杂 | 简单 | 简单 | 
| 安全校验   | no   | yes  | no   | yes  | no   | yes | 
| 类替换    | no    | no   | yes | yes  | no    | yes | 
| 资源替换  | no    | no   | yes  | yes | 即将支持| yes | 
| so替换    | no    | no   | no   | yes | 即将支持 | yes | 
| 全平台支持 |no    | no   | no   | yes  | yes   | yes | 
| 开发透明  | no    | no   | yes  | yes | no    | yes | 
| gradle支持| no    | no   | yes  | yes | yes   | no | 
| 接口文档   |较少  | 丰富 | 较少 | 丰富 | 丰富 | 丰富 | 
| 成功率    | 较低  | 一般 | 一般 | 较高 | 最高 | 较高 | 
| 后台管理  | no   | yes   | no   | yes  | no   | yes | 
| 加固兼容  | yes   | yes  | no | 部分兼容 | yes | 不确定 | 

上表基本涵盖了具有代表性的各种热修复方案，涉及到的各种关键指标的横向对比。

Slider 中大概总结了各种方案的实现方式，以及常见的问题。

Slider 地址： [http://w4lle.github.io/sliders/hot-fix/index.html](http://w4lle.github.io/sliders/hot-fix/index.html)

详细的各种方案分析：

- [Android热补丁之AndFix原理解析](http://w4lle.github.io/2016/03/03/Android%E7%83%AD%E8%A1%A5%E4%B8%81%E4%B9%8BAndFix%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)
- [从Instant run谈Android替换Application和动态加载机制](http://w4lle.github.io/2016/05/02/%E4%BB%8EInstant%20run%E8%B0%88Android%E6%9B%BF%E6%8D%A2Application%E5%92%8C%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)
- [Android热补丁之Tinker原理解析](http://w4lle.github.io/2016/12/16/tinker/)
- [Android热补丁之Robust原理解析(一)](http://w4lle.github.io/2017/03/31/robust-0/)

水平有限，难免有写的不对的地方，欢迎交流。
