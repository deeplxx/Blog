---
title: SVM详解
tag:
  - ML
categories: ML
mathjax: true
---

SVM的原理主要是VC维理论和最小化结构风险。在一个样本空间中，每一个样本都是空间中的一个点。SVM的任务就是寻找一个超平面将样本分割为不同的类，并且保证支持向量间隔最大。

![SVM图示1](http://s2.sinaimg.cn/middle/4298002etbb8e72012971&690)

## SVM优化方程的由来
如上图，假设有一个超平面H：$w*x+b=0$ 可以将这些样本准确的分割开来，同时满足所有的正样本落在$w*x+b\ge1$ ，负样本落在 $w*x+b\le-1$ 内。写成统一的式子就是
$$y_i(w*x_i+b)-1\ge0 \qquad(1)$$
其中使等号成立的样本称为**支持向量**，两个异类支持向量到超平面的距离之和为**间距**：$\gamma=\frac{2}{||w||}$

欲找到最大间隔，即$\max\limits_{w} \frac{2}{||w||}$ 也即
$$\min\limits_{w} \frac{||w||^2}{2}\quad s.t.\,y_i(w*x_i+b)-1\ge0 \qquad(2)$$

对于不等式约束下的条件极值问题，可以用拉格朗日方法求解。拉格朗日方程的构造规则是：用约束方程乘以非负的拉格朗日系数，然后从目标函数中减去。如下：
$$L(w,b,\alpha_i)=\frac{1}{2}||w||^2-\sum\alpha_i(y_i(w*x_i+b)-1)\quad s.t.\,\alpha_i>=0\qquad(3)$$
这样我们要处理的优化问题就变为：
$$\min\limits_{w,b} \max\limits_{\alpha_i\ge0} L(w,b,\alpha_i)\qquad(4)$$

(4)式是一个凸规划问题，其意义是先对$\alpha$求导，令其等于0消掉$\alpha$，然后再对w和b求L的最小值。便捷的解决办法为求(4)的对偶问题，将(4)做一个对偶变换：
$$\max\limits_{\alpha_i\ge0} \min\limits_{w,b} L(w,b,\alpha_i)\qquad(5)$$
这样就转换为先消去w和b然后再对$\alpha$求L的最大值。下面求解(5)：
$$
\left\{
\begin{array}{c}
\frac{\partial L(w,b,\alpha_i)}{\partial w} = w - \sum\alpha_iy_ix_i = 0 \\
\frac{\partial L(w,b,\alpha_i)}{\partial b} = - \sum\alpha_iy_i = 0
\end{array}
\right.
$$
得到
$$
\left\{
\begin{array}{c}
w = \sum{\alpha_iy_ix_i} \\
\sum{\alpha_iy_i} = 0
\end{array}
\right.
\qquad(6)
$$
将(6)带入回(3)可得：
$$\min\limits_{w,b} L(w,b,\alpha_i) = \sum{\alpha_i} - \frac{1}{2}\sum_{i}\sum_{j}\alpha_i\alpha_jy_iy_j(x_ix_j) \qquad(7)$$
将(7)带入(5)可得对偶问题：

**$$\max\limits_{\alpha} \sum{\alpha_i} - \frac{1}{2}\sum_{i}\sum_{j}\alpha_i\alpha_jy_iy_j(x_ix_j)$$
$$
s.t.
\left\{
\begin{array}{c}
\alpha_i\ge0 \\
\sum{\alpha_iy_i} = 0 \\
\alpha_i(y_i(w*x_i+b)-1) = 0
\end{array}
\right.
\qquad(8)
$$**

其中最后一个约束条件是隐含的，因为要(2)和(4)等效则需 $\min \sum{\alpha_i(y_i(w*x_i+b)-1)} = 0$。又由(2)和(3)的约束知 $\alpha_i(y_i(w*x_i+b)-1)\ge0$，故 $\alpha_i(y_i(w*x_i+b)-1) = 0$

约束(8)的意义是：**如果一个样本不是支持向量，则其对应的拉格朗日系数为0**。这显示出SVM的一个重要性质以及名字的由来：训练完成后，大部分样本都不需保留，最终模型仅与支持向量有关！

## 核函数
上述都是建立在样本线性可分的情况下，对于线性不可分的问题，希望将样本从原始的低维空间映射到高维特征空间，使得样本在高维空间内线性可分。即定义高维空间映射$\phi(x)$，映射后的超平面 $w*\phi(x)+b$

根据(6)，
$$
\begin{align}
f(x) &= w*\phi(x)+b \\
     &= \sum{\alpha_iy_i\phi(x_i)^T\phi(x)+b} \\
     &= \sum{\alpha_iy_i\kappa(x,x_i)+b}
\end{align}
$$
这里的函数 $\kappa(x,x_i)$ 就是核函数。

现实任务中我们通常不知道合适的 $\phi(x)$ 的形式，该如何确定核函数呢？根据核函数定理，**只要一个对称函数对应的核矩阵半正定，该函数就可以作为核函数使用**。由于不知道映射形式，所以我们并不知道什么样的核函数是最合适的，因此，核函数的选择成为SVM最大的变数。

几种常用的核函数：
    * 线性核： $\kappa = x_i^Tx_j$
    * 多项式核： $\kappa = (x_i^Tx_j)^d$ 其中d为多项式次数
    * 高斯核： $\kappa = exp(-\frac{||x_i-x_j||^2}{2\sigma^2})$
    * 拉普拉斯核： $\kappa = exp(-\frac{||x_i-x_j||}{\sigma})$
    * Sigmoid核： $tanh(\beta{x_i^Tx_j}+\theta)$

对于任意核函数，以下组合也是核函数：
    * 线性组合 $a\kappa_1 + b\kappa_2$
    * 直积 $\kappa_1*\kappa_2$
    * $f(x)\kappa(x,y)f(y)$ 其中f(x)为任意函数
