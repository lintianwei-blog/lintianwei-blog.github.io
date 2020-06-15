---
layout:     post
title:      Arbitrary Style Transfer 01 - AdaIN
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


关于arbitrary style transfer 的论文笔记，打算从比较早的方法写起，虽然早已经有很多人写过相关的笔记，但自己写一下还是能加深点印象。后面写的篇数多了，可以看着整合成一篇小综述。这篇笔记主要讲的就是adain，是17年的ICCV论文：“Arbitrary Style Transfer in Real-time with Adaptive Instance Normalization”。该论文的代码见 https://github.com/xunhuang1995/AdaIN-style


# 归一化方式回顾

> BN and IN



![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/cn1.png)

首先简单回顾一下BN 和 IN，两者的主要区别在于BN 是在batch 内做归一化，而IN则是做instance 内的归一化。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/gs3.png)

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/gs2.png)

本文中的实验表明，IN在style transfer 的实验中比起BN有更好的效果。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/img1.png)

> CIN

ICLR 2017 的 “A learned representation for artistic style” 一文中提出了CIN用于风格迁移，针对每个不同的风格学习 $\gamma$ 和 $\beta$，对应不同的风格。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/gs1.png)





# AdaIN

本文则提出了一种新的IN层，进一步拓展了CIN，将CIN 中依靠学习获得的$\gamma$ 和 $\beta$改为$\sigma(style)$ 和 $\sigma(y)$，即替换为了style 图对应feature map 的均值和方差。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/gs4.png)


# 模型搭建

如下图所示的是AdaIN 网络中所搭建的网络结构。对content 和 style图片分别采用固定参数的VGG模型提取特征（固定vgg网络作为特征提取在style transfer的方法中非常常见）。然后通过AdaIN层来更新content 图片的均值和方差，并通过一个学习的decoder 模块重建出风格化的图片。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/img2.png)

AdaIN 网络的训练损失函数同样由content loss 和 style loss 构成：

$$L = L_c + L_s.$$

其中的content loss为：

$$L_c = \left \| f(g(t)) - t \right \|_2$$

此处 $t$ 为AdaIN 处理后的content 特征（注意不是处理前），即$t$ 在经过decoder 和 encoder 能够重建。Style loss 为：

$$L_s = \sum_{i=1}^{L}\left \|\mu(\phi _i(g(t))) - \mu(\phi_i(s)) \right \|_2 + \sum_{i=1}^{L}\left \|\sigma(\phi _i(g(t))) - \sigma(\phi_i(s)) \right \|_2$$

此处是指生成图和原图在经过VGG网络后，feature map 所对应的均值和方差应当一致。

# 实验

AdaIN 论文中设计了很多种style transfer 的可视化实验，后续很多工作都沿用了下来，这里做一下简单的介绍。

> 风格迁移程度控制

测试时的风格迁移程度控制，在adaIN中所采取的方式是将adain 后的feature map 和content map 加权平均。content的权重越大则越接近原图，adain的权重越大则风格化越强。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/exp1.png)

> 多风格混合

这个实验的效果很酷炫，这里的实现的方式是按照比例混合多个不同style image 做adain 后的feature map，再经过decoder 生成结果，可以看做是不同风格空间的插值。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/exp2.png)

> 颜色控制

这里采用了 “Controlling Perceptual Factors in Neural Style Transfer” 一文中的color control方法。将style image 的颜色分布重新映射成与content image 类似的，然后再用处理后的style image 来做风格化，这样就能够使得只风格化纹理，而保留原始的颜色分布。这里的具体操作还没有太搞懂，需要再研究一下。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/exp3.png)

> 空间控制

空间控制就比较简单了，基于mask前景和背景用不一样的风格来处理，通常能够获得比较好的效果。

![](https://bj.bcebos.com/v1/ltwbucket/blog_images/20200609-adain/exp4.png)

# 小结

* adain作为最早一批做arbitrary style transfer 的论文，虽然效果很多时候并不算太好，但其思路非常简单可靠，AdaIN层也在后面被大量地采用。
* AdaIN的核心思想是风格是由图像特征的均值和方差表示的，记得有一篇文章证明了这个和Gram Matrix 是等价的