---
title: UI2Code（三）imgcook
date: 2019-04-08 19:30:15
tags: [机器学习]
thumbnail: https://raw.githubusercontent.com/w4lle/developnote/images/imagesimages%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_eaa2889d-2752-452f-833e-7c42fda07115.png
---

imgcook是阿里实现的基于sketch或Ps设计稿，自动生成布局代码的工具，支持生成支持flexbox布局的代码，包括JARVIS、Vue、微信小程序、React、H5、Rax等等。由两部分组成，一个是sketch(Ps)插件，另外一部分是[imgcook平台](https://imgcook.taobao.org/)。

<!--more-->

系列文章：

- [UI2Code（一）pix2code](http://w4lle.com/2019/03/13/UI2Code-0/)
- [UI2Code（二）pixeltoapp](http://w4lle.com/2019/03/22/UI2Code-1/)
- [UI2Code（三）imgcook](http://w4lle.com/2019/04/08/UI2Code-2/)





经过前面的研究，我们知道，UI2Code作为可以从UI截图生成布局代码的工作，其构建流程大致如下：

1. 版面分析，包含背景分析和前景分析
2. 提取GUI元素
3. 组件识别
4. 属性提取
5. 布局推导
6. DSL 推导
7. 编译，得到目标平台代码

其中难点是版面分析和布局推导。

版面分析，目的是得到相对准确的背景和前景，通过传统的计算机图像计算和机器学习，将UI图片拆分并分层，得到相对独立的控件，为下一步控件识别和属性提取做准备。可以说，这部分做的好坏直接影响到后面所有的流程。

而对于线上业务逻辑来说，UI图千变万化，背景和前景可能存在很复杂的耦合关系，没有统一的规则约束，这就导致版面分析的难度较大。

1. 通过纯图像计算，版面分析不准确，组件间的提取不够独立或被隔断，泛化能力不够
2. 通过机器学习识别，控件属性信息不全，没有坐标、宽度等信息，且准确度不够


所以将图像计算与机器学习相结合，通过上述1、2、3、4步骤得到一定泛化能力且相对准确的版面结构和控件信息。

而对于我们来说，1、2、3、4步想要得到的内容，在Sketch设计稿里面都是有提供的，我们只需要想办法将其提取处理就可以，然后进行下一步操作。

所以经过优化后，整个流程如下：

1. 提取 Sketch 设计稿信息
2. 布局推导
3. DSL 推导
4. 编译

imgcook 做的就是这样的事情。

# 实际效果

页面级别 ：

设计稿 

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1bpzqptv3j30k2120wh6.jpg) 

运行效果 

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1gfnovz73j30u01rcmys.jpg) 

卡片级别 ：

设计稿 

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1gfb9ubk6j30ku098my0.jpg) 

运行效果 

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1gfe6muckj30tu0glmz3.jpg)

# 实现拆解

主要分为几个大的过程：

1. Sketch -> Json
2. Json -> DSL
3. DSL -> Code

分别来看下。

## sketchToJson 

在sketch中选中图层，或者symbol，选中imgcook插件导出，当从sketch导出时，会产生一个json文件，以上面第二个卡片为例，信息如下 

![](https://ws1.sinaimg.cn/large/006tKfTcly1g1gg7w0xdqj31440u0wk9.jpg) 

其中包含的信息包括： 

- artboardImg，选中symbol的渲染图，如https://ai-sample.oss-cn-hangzhou.aliyuncs.com/test/b4dd75404f6e11e9b26815c07ac4b122.png 

- 控件唯一id 

- 控件类型，比如Text、Image、Shape、Repeat 

- 控件属性props，包括xy坐标、宽高、背景色、背景圆角、溢出处理(overflow)、图片内容、文字内容、字体和大小、字体颜色、行高、行数等等 

- children属性，这个一般都是铺平的，当时Type是Repeat时会有该值 

而在插件处理过程中，可能会有以下过程：

- 没有图层信息，图层信息都被过滤掉 

- 平铺的数组，没有层级关系和兄弟关系 

- Repeat包含多个children，内容是Text 

- 完全被遮挡或者不可见的控件被过滤 

- 复杂的mask计算，对于遮罩的处理 

- 控件类型和属性都是依次解析 

相当于把sketch文件中的所有信息都处理过后，得到一份期望的json文件，其中基本没有布局层次属性，仅有控件属性，包括大小、位置、控件特有属性等。

布局层次关系及位置关系由下一步来具体确定。

## JsonToDSL

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1ghk5j5h0j31iw0u0aim.jpg) 

把上面的json复制到imgcook平台，会发起gen-layout-process请求，该请求会把json文件当做请求参数上传，并返回一个dsl的Response，请求抓包文件结果经过缩减后大概如下：

```json
{
  "type": "Block",
  "id": "Block-661077",
  "__VERSION__": "2.0",
  "mask": true,
  "props": {
    "style": {
      "display": "flex",
      "alignItems": "flex-start",
      "flexDirection": "column",
      "width": 375,
      "height": 164
    },
    "attrs": {
      "x": 0,
      "y": 0,
      "className": "block-661077"
    }
  },
  "children": [
    {
      "__VERSION__": "2.0",
      "props": {
        "style": {
          "display": "flex",
          "alignItems": "flex-start",
          "flexDirection": "row",
          "backgroundColor": "#ffffff",
          "width": 375,
          "height": 68
        },
        "attrs": {
          "x": 0,
          "y": 0,
          "className": "shape-0"
        }
      },
      "children": [
        {
          "__VERSION__": "2.0",
          "props": {
            "style": {
              "marginTop": 6,
              "marginLeft": 6,
              "width": 23,
              "height": 23
            },
            "attrs": {
              "x": 22,
              "y": 22,
              "source": "https://ai-sample.oss-cn-hangzhou.aliyuncs.com/test/b46d27404f6e11e98785ebe830f6201d.png",
              "src": "https://ai-sample.oss-cn-hangzhou.aliyuncs.com/test/b46d27404f6e11e98785ebe830f6201d.png",
              "className": "image-7"
            }
          },
          "children": [],
          "type": "Image",
          "id": "Image-7",
          "componentType": "picture",
          "_jsonId": "Image-7",
          "_jsonElementId": "Image-7",
          "title": "Image",
          "semantic": {
            "dvc_default": [
              {
                "name": "dvc_default",
                "level": 100,
                "result": "img",
                "prefix": "",
                "id": 262862
              }
            ],
            "dvc_picture": [
              {
                "name": "dvc_picture",
                "level": 88,
                "result": "zhaoshangbank",
                "prefix": "",
                "id": 953603,
                "choose": true
              }
            ]
          }
        }
      ],
      "type": "Shape",
      "id": "Shape-0",
      "__ADAPT__": true,
      "componentType": "view",
      "_jsonId": "Shape-0",
      "_jsonElementId": "Shape-0",
      "title": "Shape",
      "semantic": {
        "dvc_layout": []
      }
    }
  ],
  "artboardImg": "https://ai-sample.oss-cn-hangzhou.aliyuncs.com/test/b4dd75404f6e11e9b26815c07ac4b122.png",
  "name": "可提1 copy",
  "componentType": "view",
  "_jsonId": "Block-661077",
  "_jsonElementId": "Block-661077",
  "title": "Block",
  "semantic": {
    "dvc_default": [
      {
        "name": "dvc_default",
        "level": 100,
        "result": "block",
        "prefix": "",
        "id": 613351
      }
    ],
    "dvc_layout": [
      {
        "name": "dvc_layout",
        "level": 100,
        "result": "box",
        "prefix": "",
        "id": 803050,
        "choose": true
      }
    ]
  }
}
```



仔细观察该json文件，得出信息： 

- 完整的DomTree，嵌套关系明确 

- 控件属性props中多了一些信息，包括布局方式("display": "flex")、flex布局方向(flexDirection)、主轴对齐方式(justifyContent)、副轴对齐方式(alignItems)、文本过长处理(text-overflow)、className、margin定位信息(marginTop、marginLeft) 

- 语义semantic，是一个数组，最终产生一个className 

- 组件类型componentType，包括view、text、picture 

这一步完成后，基本就得到了完整可用的布局DSL。

下一步就是compile的过程。

## DSL2Code

DSL compile目标代码的过程，imgcook支持比较多的代码模板，JARVIS、Vue、微信小程序、React、H5、Rax。 

以Vue为例，发起请求，Response 结果这里不再列出，感兴趣的读者可以自行查看。

其中renderCode包含3部分内容： 

- template，DOMTree，完整的布局信息，支持动态数据绑定 

- script，需要绑定的方法逻辑

- style，需要绑定的css样式

分别对应vue中的代码块 

![](https://ws4.sinaimg.cn/large/006tKfTcly1g1ghaq0z2wj315e068401.jpg) 

实际的vue代码： 

```javascript
<template>
  <div class="box">
    <div class="bd">
      <div class="zhaoshangbank-wrap">
        <img
          class="zhaoshangbank"
          src="https://ai-sample.oss-cn-hangzhou.aliyuncs.com/test/b46d27404f6e11e98785ebe830f6201d.png"
          @click="onClick_2"
        />
      </div>
      <div class="block">
        <span class="organization">招商银行 6889 </span>
        <div class="wrap">
          <span class="info">当前额度 </span>
          <span class="text">110,000</span>
        </div>
      </div>
      ...
  </div>
</template>

<script>
export default {
  name: "DvcComponent",
  methods: {
    onClick_2(){
    	console.log("test")
    },
  }
}
</script>
<style scoped>
.box {
  display: flex;
  align-items: flex-start;
  flex-direction: column;
  width: 375px;
  height: 164px;
...
</style>
```

从生成的代码中可以看出，div一般使用flex布局。 这里的图片也已经被上传的图床，虽然不可线上使用，但是可以即时的预览。后面可以通过插件导出到本地。

到这里生成的代码基本是可用的，并且在imgcook平台上可以调整样式，绑定方法等操作。

这里的分析是很早以前进行的，数据结构可能已经变更，但大体上应该是相同的。

# 实现难点

经过上面的分析，基本知道了imgcook的大致流程，个人其中的难点在如下几个方面：

一是布局推导的过程，这块也是整个UI2Code的核心内容。

从准确性和还原过程中的可选项来看，应该是通过计算得到的，其中的布局细节较多，这块也是最重要的点，其结果直接导致页面布局的好坏。

imgcook平台对于这块的实现猜测是基于切割规则加算法来实现flex布局的，整体上来说基本能够实现设计稿上所体现的布局变化，但是有时候也会显得不够灵活，下面会详细说到。

具体如何实现布局推导，可以期待imgcook官方后续有没有说明。

这里只是引用一下闲鱼之前关于切割方面的论述，基本就是先横切再竖切，递归这个过程，在切割之前可以处理坐标嵌套，得到父子信息，然后在父布局中递归切割过程，代码如下：

```python
'''
切割方法
---------------------------|
|                          | 
|    --------- 30          | 
|    |        |            | 
|    --------- 60          | 
|                          | 
|    —————————— 120        | 
|    |         |           | 
|    ---------- 130        | 
|                          | 
|__________________________|

如上图所示，第一个切线是60，第二个切线是130
'''
def cut_by_col(cut_num, _im_mask):
    zero_start = None
    zero_end = None
    end_range = len(_im_mask)
    for x in range(0, end_range):
        im = _im_mask[x]
        if len(np.where(im == 255)[0]) == len(im):
            # 判断是否贯穿整个区域
            if zero_start == None:
                zero_start = x
        elif zero_start != None and zero_end == None:
            zero_end = x
        if zero_start != None and zero_end != None:
            start = zero_start
            if start > 0:
                # 首次非联通区域过滤掉
                cut_num.append(start)
            zero_start = None
            zero_end = None
        if x == end_range - 1 and zero_start != None and zero_end == None and zero_start > 0:
            zero_end = x
            start = zero_start
            if start > 0:
                cut_num.append(start)
            zero_start = None
            zero_end = None
```

切割过后，根据各种优化算法得到实际的布局信息。

当然，实际生产过程肯定要比我猜测的要复杂的多得多。比如什么时候适合flex布局，什么时候适合相对布局，以及谁是优先级更高的，还有是否可以阈值控制来使相对布局变成flex布局等等。



第二个难点是ClassName的语义分析。

猜测是通过机器学习进行OCR图片识别和NLP自然语言处理翻译，这块属于易用性方面的优化。



第三个难点是配套设施，上面分析的都是如何得到一个相对准确的布局DSL，imgcook平台除了这个核心之外，平台给开发者提供了更好的拓展，根据得到的DSL信息，开发者可以根据自己的喜好编译成各种平台、各种语言。另外还有工程化配套的工具，比如导出插件，可以把生成的文件全部导出到本地，不依赖线上图床服务等等。

# 存在的问题

- 数据绑定，需要手动绑定数据，并且data名字要先想好 

- 图床问题，默认是上传至阿里的图床，不可以直接使用，可以使用导出的zip包中包含所有的icon 

- 界面布局，简版 flex，基本只有方向，默认margin定位 

- flex 的对齐方式，比如justify-content: space-between属性阈值较低，大概2个像素，不容易被触发 

不过imgcook团队一直在做优化工作，相信可以做的越来越好用，造福开发者。

# u51-weex

基于以上的分析，和imgcook平台的开放能力，我这边第一时间体验了自定义DSL的功能，由于我们团队内使用weex较多，所以做了一个自定义weex DSL的功能，地址在：https://github.com/imgcook-dsl/u51-weex

具体的使用方式可以参考官方文档：https://imgcook.taobao.org/docs?slug=dsl-dev

理论上来说可以对任意平台的布局进行拓展。



这块也在团队内部做了一些推广，整体上来说可以提高UI方面的开发效率，当然也存在着一些问题，这里就不具体展开了。

# 总结

整体来讲imgcook是一个基本可用的布局代码生成平台，对于提升工作效率方面有一些帮助。

同时，对于拓展平台支持的兼容性也比较不错。

虽然对于页面级别也可以生成，但是对于代码层级关系、后期代码维护上都不太建议直接上页面级别；比较建议的方式是生成卡片item的布局，导出作为一个component使用。 

最后希望更多开发者可以使用imgcook，提出一些建议，使得他们团队可以进行更多的优化。

这篇文章写的时间比较久，当文章发布的时候发现imgcook平台又新添加了[Lab](https://imgcook.taobao.org/labs)，期待更多有意思的产品。