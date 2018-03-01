---
title: xgboost详解
tag: ML
categories: ML
mathjax: true
---

## RDT(回归树)
回归树的总体流程类似于分类树，区别在于，**回归树的每个节点都会返回一个预测值，该预测值等于该节点所有样本label的平均值**。分支时穷举每个feature的每个阈值找最好的分割点，衡量标准不再是最大熵，而是**最小化平方误差**。分支终止条件为**每个叶子节点都为同类样本，或是达到预设的叶子树或树深度上限**
![RDT](http://img.blog.csdn.net/20160418093633792)
> 分类树和回归树区别就在于分类树靠熵来划分，而回归树靠定义节点的预测值，根据最小化损失函数来划分

## GBDT
GBDT是迭代多颗回归树来共同决策。当采用平方误差损失函数时，每颗回归树学习的是之前所有树的结论和残差，拟合得到一个当前的残差回归树

## xgboost
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

对 $f$ 的定义做一下细化，将树拆分为结构部分 $q$ 和叶子权重部分 $w$。结构函数 $q$ 把输入映射到叶子的索引号上去(**表示输入的样本属于哪一片叶子，由上一次迭代的训练结果得到**)，而 $w$ 给定了**每个索引号对应的叶子分数**。
$$f_t(x) = w*q(x) \quad w\in R^T, q: R^d \to \left\{1,2,...,T \right\} \tag{4}$$
然后定义树的复杂度如下，包含了一棵树中**节点的个数**，以及每个数叶子节点上面输出分数的 $L2$ 模平方：
$$\Omega(f_t) = \gamma T + \frac{1}{2}\lambda\sum_{j=1}^T{w_j^2} \tag{5}$$

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

xgboost的流程就是这样的：
* 从深度为0的树开始，对每个叶节点枚举所有可用特征
* 对每个特征，求出该特征最大增益，选择增益最大的特征作为分裂特征，用该特征的最优分裂点最为分裂点进行分裂
* 一直分裂直到达到终止条件，得到一颗新树并加入集成模型中。
* 为最终的叶子节点求 $w$，根据新的集成模型更新 $G,H,T$ 。然后进行下一次迭代(其中，w用来最终模型的预测，GHT用来进行下一次迭代)
![xgboost流程](https://pic3.zhimg.com/80/f7273396ac92eae3eca2f7d89650a233_hd.jpg)

xgboost对比GBDT的优点：
* 损失函数用泰勒展开二项逼近，不像GBDT是一阶导数
* 对树结构进行了正则化约束，降低了过拟合
* 节点分裂方式，gbdt是用的gini系数，xgboost是经过优化得到的一个增益

-----------------
参考资料：
1. [陈天奇：xgboost](http://www.52cs.org/?p=429)
2. [集成学习总结](https://xijunlee.github.io/2017/06/03/%E9%9B%86%E6%88%90%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93/)
