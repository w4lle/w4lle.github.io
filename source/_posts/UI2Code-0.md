---
title: UI2Code（一）pix2code
date: 2019-03-13 16:27:22
tags: [机器学习]
thumbnail: https://raw.githubusercontent.com/w4lle/developnote/images/images20191115163604.png
---

pix2code 项目通过机器学习，支持输入一张图片输出实际的布局代码，同时支持生成三端(Android、iOS、web)布局代码。

pix2code作为UI2Code的先驱项目，后续的相关项目或多或少的都有参考该项目的实现。

<!--more-->

系列文章：

- [UI2Code（一）pix2code](http://w4lle.com/2019/03/13/UI2Code-0/)
- [UI2Code（二）pixeltoapp](http://w4lle.com/2019/03/22/UI2Code-1/)
- [UI2Code（三）imgcook](http://w4lle.com/2019/04/08/UI2Code-2/)

# 概述

[项目地址](https://github.com/tonybeltramelli/pix2code)
[论文地址](https://arxiv.org/abs/1705.07962)
[论文翻译](https://mp.weixin.qq.com/s?__biz=MzA3MzI4MjgzMw==&mid=2650728064&idx=1&sn=adb744e599299916faa23545c2ab436e&chksm=871b22feb06cabe83ba0e3268a7a8c0c486cf6f42427ab41814e69242f919b25bc7de06ea258&scene=21#wechat_redirect)

使用双显卡服务器跑出模型，耗时大概1.5小时，模型数量1500张图片，1500个对应DSL，模型大小450M左右

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191115164209.png)

然后使用验证集生成DSL，比对原始DSL

对比1：

![](https://raw.githubusercontent.com/w4lle/developnote/images/images03BC80FA-5959-4432-9622-19958492D0E7.png)

对比2：

![](https://raw.githubusercontent.com/w4lle/developnote/images/images61B2D439-EC8E-4574-8BB9-D7D427B1BDF0.png)

对比3：

![](https://raw.githubusercontent.com/w4lle/developnote/images/images77658E09-48B9-489D-B69A-69C6DB57AD75.png)

验证集共生成了250张图片对应的DSL，总体来看不能做到100%还原DSL，作者称可以做到77%左右的准确率。

以其中一个图片为例

![](https://raw.githubusercontent.com/w4lle/developnote/images/imagesCD7F3D19-3D1B-4070-833C-54AE57B6AC9E.png)

生成的DSL 

```

stack{ 

  row{ 

    label,slider,label 

  } 
} 

footer{ 

	btn-search,btn-search,btn-search,btn-search 
} 
```

编译compile后生成的Android xml 布局文件

![](https://raw.githubusercontent.com/w4lle/developnote/images/images7B1049AC-A185-4F64-945D-32CDB2EF7C57.png)

结果是有10个控件，其中的8个是正确的，2个是识别错误的。

# 实现原理 

基于图像标记（image caption）构建一种把图像和文本连接在一起的模型，用于生成源图像内容的描述。 

pix2code是一个基于卷及神经网络(CNN)和循环神经网络(LSTM，长短时神经单元)能够由单个GUI屏幕截图生成源代码的项目。 

![](https://raw.githubusercontent.com/w4lle/developnote/images/images20191115163604.png)

模型采用监督学习，有连个输入，一个数GUI截图，另外一个是对应的布局DSL。这些布局文件都足够简单，DSL也仅仅对控件进行了描述，不涉及位置信息和控件属性。
首先将GUI图像 I 通过CNN网络生成特征向量P，然后DSL 符号T 描述切割成一个序列 X（xt, t ∈ {0 . . . T − 1}）通过第一个语言模型得到特征向量qt（该语言模型由两个 LSTM 层、每层带有128个神经单元来实现），视觉编码向量 p 和语言编码向量 qt 可以级联为单一向量 rt，该级联向量 rt 随后可以投送到基于 LSTM 的模型来解码。因此解码器就学到了输入 GUI 图像中的对象和 DSL 代码中的符号间的关系，因此也就可以对这一关系进行建模。我们的解码器由两个 LSTM 层、每层带有 512 个单元的堆栈而实现。最后通过一个softmax进行一个分类对当前项进行预测，并把结果作为下一项的输入。

核心代码：

```
image_model = Sequential()
image_model.add(Conv2D(32, (3, 3), padding='valid', activation='relu', input_shape=input_shape))
image_model.add(Conv2D(32, (3, 3), padding='valid', activation='relu'))
image_model.add(MaxPooling2D(pool_size=(2, 2)))
image_model.add(Dropout(0.25))

image_model.add(Conv2D(64, (3, 3), padding='valid', activation='relu'))
image_model.add(Conv2D(64, (3, 3), padding='valid', activation='relu'))
image_model.add(MaxPooling2D(pool_size=(2, 2)))
image_model.add(Dropout(0.25))

image_model.add(Conv2D(128, (3, 3), padding='valid', activation='relu'))
image_model.add(Conv2D(128, (3, 3), padding='valid', activation='relu'))
image_model.add(MaxPooling2D(pool_size=(2, 2)))
image_model.add(Dropout(0.25))

image_model.add(Flatten())
image_model.add(Dense(1024, activation='relu'))
image_model.add(Dropout(0.3))
image_model.add(Dense(1024, activation='relu'))
image_model.add(Dropout(0.3))

image_model.add(RepeatVector(CONTEXT_LENGTH))

visual_input = Input(shape=input_shape)
encoded_image = image_model(visual_input)

language_model = Sequential()
language_model.add(LSTM(128, return_sequences=True, input_shape=(CONTEXT_LENGTH, output_size)))
language_model.add(LSTM(128, return_sequences=True))

textual_input = Input(shape=(CONTEXT_LENGTH, output_size))
encoded_text = language_model(textual_input)

decoder = concatenate([encoded_image, encoded_text])

decoder = LSTM(512, return_sequences=True)(decoder)
decoder = LSTM(512, return_sequences=False)(decoder)
decoder = Dense(output_size, activation='softmax')(decoder)

self.model = Model(inputs=[visual_input, textual_input], outputs=decoder)

optimizer = RMSprop(lr=0.0001, clipvalue=1.0)
self.model.compile(loss='categorical_crossentropy', optimizer=optimizer)
```

具体原理参考[论文翻译](https://mp.weixin.qq.com/s?__biz=MzA3MzI4MjgzMw==&mid=2650728064&idx=1&sn=adb744e599299916faa23545c2ab436e&chksm=871b22feb06cabe83ba0e3268a7a8c0c486cf6f42427ab41814e69242f919b25bc7de06ea258&scene=21#wechat_redirect)。



生成 DSL 后，通过compiler编译成三端代码 UI -> DSL -> Code。

其中compile过程，就是替换DSL描述到实际控件的过程，这些实际控件的属性和布局位置信息全是写死的，存在放dsl-mapping文件中，以其中一个控件为例，对照关系如下：

```
"check": 
"<CheckBox android:id=\"@+id/[ID]\" 
	android:layout_width=\"wrap_content\" 
	android:layout_height=\"wrap_content\" 
	android:paddingRight=\"10dp\" 
	android:text=\"[TEXT]\"/>",
```

所以pix2code的核心是通过机器学习得到对应布局的DSL，compiler 是规则替换的过程。

# 优化

pix2code具有70%左右的准确率， 基于该项目，西安交大通过一些优化使准确率进一步提高。

![](https://raw.githubusercontent.com/w4lle/developnote/images/images4E51A20B-550F-4016-94FF-9C718DDA2B2F.png)

参考文章 https://www.jiqizhixin.com/articles/2018-11-05-12?from=synced&keyword=pix2Code

# 总结

pix2code项目本身作为一个实验性质的项目，论证了GUI截图通过AI手段自动生成布局代码的可行性。 



但是由于其也存在一些问题： 

- 模型准确率不高，修改模型或者重新构建模型成本高 
- 中间过程不允许人工干预，其结果就是完整的DSL，人工干预是在DSL生成后 
- 训练素材标准成本高，这也是深度学习都会碰到的一个实际问题 
- 切割精准度不够，实际上它是以CNN网络提取图片特征，精准度不大达到像素级别的要求 
- 控件位置和属性信息缺失，不能准确还原布局 

基于以上调研分析，pix2code项目仅仅作为一个研究性项目进行开源，并不能实际使用在生产环境中，作者明确表述该项目和论文仅仅作为实验，并且不会再进行进一步的平台拓展。 

我们可以探索在pix2code基础上，通过图像分析+pix2code控件识别 来进行UI2Code的工作。 

后面会去调研下pixelToApp这个项目，其基于传统的计算机视觉识别技术进行代码生成。 

论文：

[UI Design to Code Skeleton](https://cs.anu.edu.au/courses/CSPROJECTS/18S2/initialTalks/u6013787.pdf)

参考文章：

- [深度学习助力前端开发：自动生成GUI图代码](https://mp.weixin.qq.com/s?__biz=MzA3MzI4MjgzMw==&mid=2650728064&idx=1&sn=adb744e599299916faa23545c2ab436e&chksm=871b22feb06cabe83ba0e3268a7a8c0c486cf6f42427ab41814e69242f919b25bc7de06ea258&scene=21#wechat_redirect)
- [基于AI的移动端自动化测试框架的设计](https://www.jiqizhixin.com/articles/2018-12-29-5) 
- [前端设计图转代码，西安交大表示复杂界面也能一步步搞定](https://www.jiqizhixin.com/articles/2018-11-05-12?from=synced&keyword=pix2Code)
- [前端利器！让AI根据手绘原型生成HTML | 教程+代码](https://mp.weixin.qq.com/s?__biz=MzIzNjc1NzUzMw==&mid=2247496681&idx=3&sn=885b227400bdc8c81d74feffcb1b6c5c&scene=0#wechat_redirect)
- [How you can train an AI to convert your design mockups into HTML and CSS](https://medium.freecodecamp.org/how-you-can-train-an-ai-to-convert-your-design-mockups-into-html-and-css-cc7afd82fed4)