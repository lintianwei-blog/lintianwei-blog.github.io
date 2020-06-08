---
layout:     post
title:      SIGGRAPH 2020 - 交互式视频风格化
subtitle:   Style Transfer 论文笔记
date:       2020-06-07
author:     TV
header-img: img/home-bg.jpg
catalog: true
mathjax: true
tags:
    - 学习笔记
    - style transfer
---


这篇博客主要介绍一篇SIGGRAPH 2020 的交互式视频风格化论文 “Interactive Video Stylization Using Few-Shot Patch-Based Training” 效果很是酷炫（可以youtube 上搜demo）。


# 概要

这篇文章提出了一种few-shot的视频风格化方法。具体而言，输入一段视频，选取其中的一帧图像，由艺术家对其做风格化，从而我们可以得到一个原图-风格化图的pair，基于这个pair本文训练了一个U-net 结构的风格化模型，这个过程在作者的配置下大概需要16s （单卡2080ti）。在训练完成后，可以对视频序列中的其他帧进行实时风格化处理。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img1.png)

如下图所示，是本文提出的两种style 参考帧形式，一种是全图的，另外一种则只对mask部分做风格化。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img2.png)

# Motivation

这篇文章实际上并不是第一个做这样问题的，作者认为此前方法主要存在两点问题做的不好：

1. 很多现有方法是时序推断的，即需要上一帧的结果来生成下一帧的结果，这使得模型的inference 不能并行，不够高效；

2. 视频的风格化常常存在比较明显的抖动问题；


# Model

这篇文章的模型其实非常简单，是一个标准的unet结构模型。比起此前一些方法的全图训练方式，该方法采用了patch训练方式，每个iter都随机地从图片中采样一些patch-pair，再基于patch-pair 训练unet 模型（实际上就是一个pix2pix模型）。而在测试时，则采用全图的inference。



![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img3.png)

如下图所示，全图训练和patch 训练均能够在训练帧获得很好的结果（拟合），但是在infer 其他帧的时候，全图训练的模型明显出现了模糊的情况，这表明模型存在过拟合的问题。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img4.png)

为了获得更好的效果，本文对unet 结构训练的batch size，patch size，block number，learning rate 的配置进行了搜索调优。这里就不详细讨论了。



# Temporal Coherency

这篇文章另外一部分内容主要是在讨论如何降低风格化后的抖动，作者认为风格化视频中的闪烁现象有两点主要的原因：

1. 视频本身在时序上存在噪声，而风格化模型会放大这种噪声，导致一定程度的波动；

2. 图像中存在容易导致歧义的区域，风格化后会容易导致抖动；

下面作者对这两个问题分别提出了策略：

> 原视频噪声问题

作者采用了SIGGRAPH 2005年“Video Enhancement Using Per-Pixel Virtual Exposures” 一文中所采用的时序滤波方法来对视频做预处理滤波，降低噪声。这里时序滤波采用的是一维的bilateral filter。这篇文章草草看了一遍，主要是基于bilateral filter 做暗光视频的去噪和增强，效果很不错，后面有时间单独学习下写篇笔记。

> 歧义区域

如下图所示，舞者身上的黑色T恤就算是一种歧义区域，因为其中缺乏纹理，容易出现抖动。

作者在这里设计了一种额外的网络输入，如下图(b) 所示，首先将参考帧前景划分成grid，在每个grid中生成一个高斯分布的彩色颜色；然后对于需要inference的帧，通过匹配+Warp的方式，得到一个对应的高斯色彩图。通过这种方式将两帧中相应的部分进行了对应，从而使得纹理也能够对应上去。

具体而言，这里匹配+Warp的部分作者采用了作者自己2009 年SIGGRAPH 论文 “As-Rigid-As-Possible Image Registration for Hand-drawn Cartoon Animations” 一文中的方法。感觉实现起来并不容易。当然，这里我认为也是可以借助光流来实现相同的功能。


![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img7.png)

# 效果

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200608-videostyle/img8.png)

# 讨论

* 其实通篇文章看下来，方法是很简单易懂的，就是一个Patch-based 的Single Image Pix2Pix + 时序平滑策略。但文章展现的形式非常酷炫（demo牛逼。。），还是有很多值得学习的地方。
* 最值得学习的地方还是两种时序平滑的方式，此前我也实现过一些时序平滑的方法，主要还是基于光流来做前后帧的平滑。深度学习之前的很多方法还是非常值得好好学习一下的。
* patch-based 的训练方法在单图模型中采用的非常多（比如singan啦），在数据少的情况下能够很有效的减少过拟合。
* 这篇文章的思路其实可以拓展出不同的玩法，比如在端上模型中，从服务器请求一张大模型生成的效果很强的风格化图片，然后在端上做exampler based 的快速inference。当然，前提是模型需要改成feed-forward的形式。