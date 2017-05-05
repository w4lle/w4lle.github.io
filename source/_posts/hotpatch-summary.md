---
title: hotpatch-summary
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
