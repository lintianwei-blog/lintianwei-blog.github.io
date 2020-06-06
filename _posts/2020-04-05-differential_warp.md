---
layout:     post
title:      可微分的样条插值算法 - 原理篇
subtitle:   arXiv论文笔记
date:       2020-04-05
author:     TV
header-img: img/home-bg.jpg
catalog: true
mathjax: true
tags:
    - 学习笔记
---

最近在学习GAN论文中一些涉及到基于控制点实现图像变形的论文，这些论文基本上都采用了differentiable spline interpolation module(可微分样条插值模块) 来实现图像变形。由于这个模块是可微分的，因此可以直接嵌入网络中进行学习，对于变形比较大的image-to-image translation 来说，是个很有价值的模块。这篇笔记主要是大致介绍样条插值算法的基础，后续会继续介绍其算法实现和应用的论文。根据插值函数的不同，有很多种样条插值算法，这里主要介绍**薄板样条插值（thin-plane spline interpolation，TPS)**。介绍TPS的博文很多，这里主要按照自己的理解把之前的博文做了一些整理，所参考的博文见文末所示。

## 插值

假设我们已知一些观测点$(x_1,y_1),(x_2,y2),...(x_n,y_n)$ ，而我们不知道生成这组观测值的生成函数$Y=f(X)$，那么从这一组观测点来拟合出接近真实生成函数的表达式，就称为插值函数。


## TPS 插值

TPS 是常用的2D插值方法，假设原图中有N个点$A_N$，而在变形后的图中有N个对应的点$B_N$，TPS即将这个2D的形变模拟为一个薄板发生形变的过程，其目标是让这N组匹配点正确匹配的前提下，使得薄板的弯曲能量最小。这里不给出这里的求解和证明方式（因为我也不会。。），直接给出TPS插值函数的形式：

$$\Phi(x) = w^Ts(x) + a^Tx + c$$

这里先解释一下在2D图像变形中插值函数的定义：此处的$x$指的是形变后图像上的坐标值，而$\Phi(x)$是$x$这个位置对应的形变量。我们获得插值函数的最终目的是获得形变后图像上每个点的形变量，从而从原图中采样到对应的像素值。在上式中$w\in R^{N \times 1}$, $a\in R^{D \times 1}$, $c\in R^{1 \times 1}$, 这里N是观测点的数量，D是观测点的维度，对于二维图像，D=2。可以看出，插值函数的输出是标量，因此不同的维度需要分别求解插值函数。上式中的$s(x)$的定义为：

$$s(x) = (\sigma(x-x_1),...,\sigma(x-x_n))^T$$

在TPS插值中，$\sigma(x)$的定义为：

$$\sigma(x) = \left \| x \right \|^2_2log\left \| x \right \|_2$$

在实际的算法中通常采用等价的计算方式：

$$\sigma(x) = 0.5 \cdot \left \| x \right \|^2_2log\left \| x \right \|_2^2$$

我们需要求解$w, a, c$ 共 $N+D+1$个参数，但我们只有由$N$个观测点提供的$N$个约束，因此我们需要额外添加$D+1$ 个约束条件。

$$\sum_{n=1}^{N}w_n = 0$$
$$\sum_{n=1}^{N}w_nx_n^1 = 0$$
$$\vdots$$
$$\sum_{n=1}^{N}w_nx_n^D = 0$$

基于上述的定义和约束条件，我们可以令

$$ X = \begin{bmatrix}
x_1^1 & x_1^2 & \cdots & x_1^D\\ 
x_2^1 & x_2^2 & \cdots & x_2^D\\ 
\vdots & \vdots & \ddots & \vdots \\ 
x_N^1 & x_N^2 & \cdots & x_N^D
\end{bmatrix},  Y = \begin{bmatrix} 
y_1 \\ 
y_2 \\ 
\vdots\\ 
y_N 
\end{bmatrix}$$

$$ S = \begin{bmatrix}
\sigma(x_1 -x_1) & \sigma(x_1 -x_2) & \cdots & \sigma(x_1 -x_N)\\ 
\sigma(x_2 -x_1) & \sigma(x_2 -x_2) & \cdots & \sigma(x_2 -x_N)\\ 
\vdots & \vdots & \ddots & \vdots \\ 
\sigma(x_N -x_1) & \sigma(x_N -x_2) & \cdots & \sigma(x_N -x_N)
\end{bmatrix}$$ 

可以将上述约束改写为矩阵形式：

$$ \begin{bmatrix}
S & 1_N & X \\ 
1_N^T & 0 & 0\\ 
X^T & 0 & 0 \\ 
\end{bmatrix} \begin{bmatrix}
w \\ 
c \\ 
a \\ 
\end{bmatrix} = \Gamma\begin{bmatrix}
w \\ 
c \\ 
a \\ 
\end{bmatrix} = \begin{bmatrix}
Y \\ 
0 \\ 
0 \\ 
\end{bmatrix}$$ 

当$\Gamma$ 非奇异时，可以得到唯一解。

在网络中加入TPS插值时，可以直接通过深度学习框架实现方程组求解，整体的采样插值过程时可微分的。在后续的笔记中，会介绍pytorch版本实现的TPS算法。



## 参考内容
1. https://en.wikipedia.org/wiki/Thin_plate_spline
2. https://blog.csdn.net/xholes/article/details/84560816
3. https://www.cnblogs.com/xiaotie/archive/2009/10/15/1583730.html
4. https://blog.csdn.net/VictoriaW/article/details/70161180