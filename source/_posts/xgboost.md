---
title: xgboost的推导
tag: ML
categories: ML
mathjax: true
---

xgboost的目标函数分为两部分，第一部分为**训练误差的和**，第二部分为**每棵树的复杂度的和**。形式如下：
$$
Obj(\Theta) = \sum_i^n{l(y_i,\hat{y}_i)} + \sum_{k=1}^K{\Omega}(f_k) \tag{1}
$$

定义模型：
$$
\begin{align}
\hat{y}_i^{(0)} &= 0 \\
\hat{y}_i^{(1)} &= f_1(x_i) = \hat{y}_i^{(0)} + f_1(x_i) \\
...  \\
\hat{y}_i^{(t)} &= \sum_{k=1}^t{f_k(x_i)} = \hat{y}_i^{(t-1)} + f_t(x_i)
\end{align}
\tag{2}
$$
每一轮应该加入什么 $f$ 呢？答案非常直接：选取一个 $f$ 使得目标函数 $Obj(\Theta)$ 尽量最大的降低。

对于一般的情况，采用如下的泰勒展开来定义一个近似的目标函数来进行计算：
$$
\begin{align}
Obj^{(t)} &= \sum_i^n{l(y_i, \hat{y}_i^{(t-1)} + f_t(x_i))} + \sum_{k-1}^K{\Omega(f_k)}  \\
泰勒展开&：f(x+\Delta{x}) = f(x) + f'(x)\Delta{x} + \frac{f''(x)}{2}\Delta{x} \\
定义&：g_i = \frac{\partial l(y_i, \hat{y}^{(t-1)})}{\partial \hat{y}^{(t-1)}} \quad h_i = \frac{\partial{^2} l(y_i, \hat{y}^{(t-1)})}{\partial \hat{y}^{(t-1)}{^2}} \\
将常数项移除后得到一般形式&： \\
Obj^{(t)} &\simeq \sum_{i=1}^n{g_if_t(x_i) + \frac{1}{2}h_if_t^2(x_i)} + \Omega(f_t) \tag{3}
\end{align}
$$

对 $f$ 的定义做一下细化，将树拆分为结构部分 $q$ 和叶子权重部分 $w$。结构函数把输入映射到叶子的索引号上去(**表示输入的样本属于哪一片叶子，由上一次迭代的训练结果得到**)，而 $w$ 给定了**每个索引号对应的叶子分数**。
$$f_t(x) = w*q(x) \quad w\in R^T, q: R^d \to \left\{1,2,...,T \right\} \tag{4}$$
然后定义树的复杂度如下，包含了一棵树中**节点的个数**，以及每个数叶子节点上面输出分数的 $L2$ 模平方：
$$\Omega(f_t) = \gamma T + \frac{1}{2}\lambda\sum_{j=1}^T{w_j^2} \tag{5}$$

## 正式推导
将 ${4}$, $(5)$ 代入 $(3)$可得(**注意下标**)：
$$
\begin{align}
Obj^{(t)} &\simeq \sum_{i=1}^n{g_if_t(x_i) + \frac{1}{2}h_if_t^2(x_i)} + \Omega(f_t) \\
&= \sum_{j=1}^T [(\sum_{i\in I_j}g_i)w_j + \frac{1}{2}(\sum_{i\in I_j}h_i + \lambda)w_j^2] + \gamma T
\end{align}
\tag{6}
$$

令 $G_j = \sum_{i\in I_j}g_i, H_j = \sum_{i\in I_j}h_i$ ，对目标函数求导可得最优的 $w$ 以及这个值对应的目标函数值：
$$w_j^* = -\frac{G_j}{H_j + \lambda} \tag{7}$$
$$Obj = -\frac{1}{2}\sum_{j=1}^T{\frac{G_j^2}{H_j + \lambda}} + \gamma T \tag{8}$$

Obj代表了**当我们指定一个树结构的时候，我们在目标上面最多减少多少**，我们可以把它叫做**结构分数**,可以将他认为是类似于基尼指数一样的更加一般化的对于树结构的打分函数！

最终的xgboost就是这样的一个算法：基于贪心算法，不断的枚举不同的树结构(每次尝试对已有的叶子加入一个分割，分割为对叶子中的所有可行方案进行分割。对于一个具体的分割方案，可以获得的增益由 ${9}$ 计算)。利用这个打分函数来寻找一个最优的树，并加入到模型中。
$$Gain = \frac{1}{2}[\frac{G_L^2}{H_L + \lambda} + \frac{G_R^2}{H_R + \lambda} - \frac{(G_L + G_R)^2}{H_L + H_R + \lambda}] - \gamma  \tag{9}$$
其中四个分量分别为左子树分数，右子树分数， 不分割的分数，加入新叶子节点引入的复杂度惩罚。

这样，xgboost的流程就是这样的：先初始化一个树，求最优分割，然后根据最优的一个分割对树进行分割，得到一颗新的树，然后对这颗新的树求 $w,q(x),G,H,T$,求下一个最优分割。
![xgboost流程](https://pic3.zhimg.com/80/f7273396ac92eae3eca2f7d89650a233_hd.jpg)
