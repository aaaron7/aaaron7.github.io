---
layout: post
title : "线性回归的数学推导"
date: 2018-01-15 21:06:59 +0800
categories: ML
author: aaron
---

最近花时间认真学习了一下机器学习中的回归问题，虽然回归问题（线性回归+对率回归）在周志华的书里就那么几页，但其实书里对很多公式的推导省略了大量的细节，这次秉着较真的态度看了一下，发现名堂其实不少。写篇文章总结一下。


# 1. 机器学什么

了解过机器学习都应该知道，机器学习的本质就是给出一个包含n条记录的记录集合$$\begin{align*}\mathbf{D}=\{(\mathbf{x_{1}},y_{1}),(\mathbf{x_{2}},y_{2}),...,(\mathbf{x_{n}},y_{n}) \} \end{align*}$$，其中某一条记录$$\begin{align*}\mathbf{x_{i}} \end{align*}$$表示第i条记录的输入，$$\begin{align*} y_{i} \end{align*}$$代表第i条记录的输出标记。而单个$$\begin{align*} x_{i} \end{align*}$$一般为向量，表示每条记录的属性集合，
$$\begin{align*} \mathbf{x_{i}} = \{x_{i1},x_{i2},...,x_{id}\} \end{align*}$$ 表示对于每一条记录的x，包含d个属性。

机器学习的任务都是通过数据集$$\begin{align*} \mathbf{D} \end{align*}$$, 学习得一个$$\begin{align*} f \end{align*}$$
，使得$$\begin{align*} f(x_{i}) \approx y_{i} \end{align*}$$。

# 2. 线性模型
这个$$\begin{align*} f \end{align*}$$长什么样，一般被称作“模型”，线性模型基本是最简单的一种：将属性加权组合。给定一个包含d个属性的记录$$\begin{align*}\mathbf{x}=\{x_{1}; x_{2}; x_{3}...x_{d}\}  \end{align*}$$,线性模型尝试学得:

$$\begin{align*} \mathbf{f(x)} = w_{1}x_{1} + w_{2}x_{2} + ... + w_{d}x_{d} + b\end{align*}$$  

其中，$$\begin{align*} \mathbf{w} = \{w_{1},w_{2},...w_{d}\} \end{align*}$$ 为各个属性的权重，b为偏移(可以类比直线方程，$$\begin{align*} y = kx + b \end{align*}$$)。

这里所说的学习，就是要通过数据集$$\begin{align*} D \end{align*}$$ 来确定 $$\begin{align*} \mathbf{w} 和 b  \end{align*}$$。把w和x写为向量形式，则线性模型可以表示为：

$$\begin{align*} \mathbf{f(x)} = \mathbf{w^{T}x} + b \end{align*}$$

> $$\begin{align*} \mathbf{w} \end{align*}$$ 为什么要转置？机器学习中所有的向量默认都是列向量，所以$$\begin{align*} \mathbf{w} \end{align*}$$需要转置成行向量才可以和$$\begin{align*}\mathbf{x}  \end{align*}$$相乘。 这里如果懵逼的话，建议复习一下矩阵乘法的规则。

$$\begin{align*}  \end{align*}$$

# 3. 线性回归
一句话解释，线性回归就是线性模型的学习过程。在给定输入数据集$$\begin{align*} \mathbf{D} \end{align*}$$的基础上, 找到能够满足要求的 $$\begin{align*} \mathbf{w}和b \end{align*}$$, 这个过程，就是线性回归。

那要怎么求解线性回归呢？不同的老师会用不同的方法，比如周志华会先从一元变量来举例（即假设所有记录都只有一个属性），进而扩展到多元的场景。这里介绍林轩田的求解思路，更nice一些。

## 3.1 等价变换

这里我们把$$\begin{align*} b \end{align*}$$ 记做 $$\begin{align*} w_{0}  \end{align*}$$，再设 $$\begin{align*} x_{0}=1 \end{align*}$$。这样，上面的线性模型可以等价改写为：

$$\begin{align*} 
    \mathbf{f(x)} = w_{0}x_{0} + w_{1}x_{1} + w_{2}x_{2} + ... + w_{d}x_{d}
 \end{align*}$$

等价为向量模式（因为新引入了$$\begin{align*} w_{0} \end{align*}$$和 $$\begin{align*} x_{0} \end{align*}$$,故用大写表示）：

$$\begin{align*} \mathbf{f(x)} = W^{T}X \end{align*}$$

## 3.2 误差衡量

在考虑如何找到最好的$$\begin{align*} W \end{align*}$$之前，我们先考虑我们如何衡量一个$$\begin{align*} W  \end{align*}$$的好坏。在机器学习中，最常见的误差衡量标准就是均方差（Mean Square Erro，MSE）。给定一个$$\begin{align*} W \end{align*}$$时，在样本集$$\begin{align*} \mathbf{D} \end{align*}$$的n个样本的MSE为：

$$\begin{align*} 
    E(W) = \frac{1}{n}\sum_{i=1}^{n}(W^{T}X_{i} - y{i})^{2}
 \end{align*}$$

> 这里的$$\begin{align*} X_{i} \end{align*}$$是向量，代表一条记录的输入，$$\begin{align*} y_{i} \end{align*}$$代表对应的样本标签。

为了方便后面的推导，交换一下X和w的位置，所以变成：

$$\begin{align*} 
    E(W) = \frac{1}{n}\sum_{i=1}^{n}(X_{i}^{T}W - y_{i})^{2}
 \end{align*}$$

> 如果不明白为毛可以交换，你可以展开一下，就会发现是等价的；

这里我们看到式子里有个求和的符号，看着比较讨厌（不方便后续推导）。能不能干掉呢？观察一下求和内部的式子，是一堆平方的和。数学中有不少的概念最终落地下来是平方和的形式，一个直接的例子就是向量的模。比如一个N维向量$$\begin{align*} \vec a \end{align*}$$ 的模 $$\begin{align*} \|\vec a\| = \sqrt[2]{a_{1}^{2} + a_{2}^{2} + a_{3}^{2}...+ a_{N}^{2}  } \end{align*}$$, 同理，上述的E(W)可以写成：

$$\begin{align*} 
    E(W) = \frac{1}{n}\begin{Vmatrix}
        X_{1}^{T}W - y_{1} \\
        X_{2}^{T}W - y_{2} \\
        ... \\
        X_{n}^{T}W - y_{n}
    \end{Vmatrix}
    ^{2}
 \end{align*}$$


可等价变换为：

$$\begin{align*} 
    E(W) = \frac{1}{n}\begin{Vmatrix}
        \begin{pmatrix}
            X_{1}^{T} \\
            X_{2}^{T} \\
            ... \\
            X_{n}^{T} 
        \end{pmatrix}W - 
        \begin{pmatrix}
            y_{1} \\
            y_{2} \\
            ... \\
            y_{n}
        \end{pmatrix}
    \end{Vmatrix}
    ^{2}
 \end{align*}$$

将X 和 y表示为向量形式，于是有：

$$\begin{align*} 
    E(W) = \frac{1}{n}
    \begin{Vmatrix}
        \mathbf{X^{T}}W - \mathbf{y}
    \end{Vmatrix}
    ^{2}
 \end{align*}$$

## 3.3 误差最小化

当我们得出E(W)的表达式之后，整个线性回归的过程就变成了找到一个W，使E(W)最小化。我们都知道求极值最直接的方法就是对其函数求导，导数为0的点就是极值点。再通过分析左右导数的正负情况即可得到能够使E(W)最小的W。

首先对E(W) 求导可得：

$$\begin{align*} 
    \frac{\partial E(W)}{\partial W} = 
        \frac{\partial \frac{1}{n}(
            \mathbf{W^{T}{X^{T}XW} - 2W^{T}X^{T}y + y^{T}y}
        )}{\partial W}
 \end{align*}$$

> 这里先用了一个简单的公式把上式展开 $$\begin{align*} (a+b)^{2} = a^{2} + 2ab + b^{2} \end{align*}$$

在上式中，X和y都是标量（只有W是变量），为了理解方便，我们假设 $$\begin{align*} A = X^{T}X， b = X^{T}y, c = y^{T}y \end{align*}$$ , 于是上式可以化简为：

$$\begin{align*} 
    \frac{\partial E(W)}{\partial W} = 
        \frac{\partial \frac{1}{n}(
            \mathbf{W^{T}AW - 2W^{T}b + c}
        )}{\partial W}
 \end{align*}$$

 现在的问题来了，W是一个d维向量，代表每个输入记录的d个特征值。我们该如何对向量求导呢？先假设w,a,b,c是普通的标量，则有：

$$\begin{align*} 
    \frac{\partial E(w)}{\partial w} = 
        \frac{\partial \frac{1}{n}(
            aw^{2} - 2wb + c
        )}{\partial w}
    = 
    \frac{1}{n}(2aw - 2b)
\end{align*}$$

标量情况的确是很简单，那向量情况呢？一般来说对矩阵求导都比较复杂，具体有兴趣的同学可以参考MIT的Matrix cookbook。原理其实比较简单，无非就是分别针对向量的各个分量求偏导，最终的结果组成的向量就是求导的结果。但推导起来肯定都比较复杂。cookbook针对常见情况直接给出了公式，非常方便。

要求解上式的公式，主要会用到里面的两个公式：

公式(69):   $$\begin{align*} 
    \frac{\partial \mathbf{x}^{T}\mathbf{a}}{\partial \mathbf{x}} =   
    \frac{\partial \mathbf{a}^{T}\mathbf{x}}{\partial \mathbf{x}} =
    \mathbf{a}
\end{align*}$$

公式(81): $$\begin{align*}
    \frac{\partial \mathbf{x}^{T}B\mathbf{x}}{\partial \mathbf{x}} = (B + B^{T})\mathbf{x}
\end{align*}$$

将(69)和(81)代入上面的向量版微分公式，可以得到：

$$\begin{align*} 
    \frac{\partial E(W)}{\partial W} = 
        \frac{\partial \frac{1}{n}(
            \mathbf{W^{T}AW - 2W^{T}b + c}
        )}{\partial W} 

        = \frac{1}{n}((A + A^{T})W - 2b) 
        = \frac{1}{n}(2AW - 2b) \\

        = \frac{2}{n}(X^{T}XW - 2X^{T}y)
 \end{align*}$$

> 因为$$\begin{align*} A = X^{T}X \end{align*}$$， 所以 $$\begin{align*} A+A^{T} = 2A \end{align*}$$

可以看到形式上，求导结果和标量版的一模一样。这或许就是线性代数美的地方。

令E(W)=0，可以得到此时W的取值为：

$$\begin{align*} 
    W = (X^{T}X)^{-1}X^{T}y
 \end{align*}$$

上式也叫W的闭式解(Closed-form solution)， 代表是从公式精准的推导出的结果，而不是靠一些数值优化方法得出。机器学习中，由于n的数量一般都>d+1(d为属性的个数)，所以$$\begin{align*} X^{T}X \end{align*}$$ 一般都是可逆的。



# 参考资料

1. 林轩田的公开课；
2. Matrix cookbook；

