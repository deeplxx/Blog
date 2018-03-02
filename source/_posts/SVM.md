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
$$y_i(w*x_i+b)-1\ge0 \tag{1}$$
其中使等号成立的样本称为**支持向量**，两个异类支持向量到超平面的距离之和为**间距**：$\gamma=\frac{2}{||w||}$

欲找到最大间隔，即$\max\limits_{w} \frac{2}{||w||}$ 也即
$$\min\limits_{w} \frac{||w||^2}{2}\quad s.t.\,y_i(w*x_i+b)-1\ge0 \tag{2}$$

对于不等式约束下的条件极值问题，可以用拉格朗日方法求解。拉格朗日方程的构造规则是：用约束方程乘以非负的拉格朗日系数，然后从目标函数中减去。如下：
$$L(w,b,\alpha_i)=\frac{1}{2}||w||^2-\sum\alpha_i(y_i(w*x_i+b)-1)\quad s.t.\,\alpha_i>=0\tag{3}$$
这样我们要处理的优化问题就变为：
$$\min\limits_{w,b} \max\limits_{\alpha_i\ge0} L(w,b,\alpha_i)\tag{4}$$

(4)式是一个凸规划问题，其意义是先对$\alpha$求导，令其等于0消掉$\alpha$，然后再对w和b求L的最小值。便捷的解决办法为求(4)的对偶问题，将(4)做一个对偶变换：
$$\max\limits_{\alpha_i\ge0} \min\limits_{w,b} L(w,b,\alpha_i)\tag{5}$$
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
\tag{6}
$$
将(6)带入回(3)可得：
$$\min\limits_{w,b} L(w,b,\alpha_i) = \sum{\alpha_i} - \frac{1}{2}\sum_{i}\sum_{j}\alpha_i\alpha_jy_iy_j(x_ix_j) \tag{7}$$
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
\tag{8}
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

## SMO算法求解SVM
SVM的对偶问题是一个凸二次规划问题，这样的二次规划问题具有全局最优解，可以用SMO算法来求解

求解N个参数($\alpha_1,...,\alpha_i$)，首先想到坐标上升的思路，一次优化一个参数：例如固定除 $\alpha_1$ 以外的参数，求解 $\alpha_1$ 。但是注意到约束条件${8.3}$，因此选择一次优化两个参数，固定其余的 $N-2$ 个参数。
假设选择优化的变量为 $\alpha_1,\alpha_2$，则$8$的等式可化为：
$$\min \Psi{(\alpha_1，\alpha_2)} = \frac{1}{2}K_{11}\alpha_1^2 + \frac{1}{2}K_{22}\alpha_2^2  + y_1y_2K_{12}\alpha_1\alpha_2 + y_1\alpha_1*v_1 + y_2\alpha_2*v_2 - (\alpha_1 + \alpha_2) + constant \tag{9}$$
其中
$$v_i = \sum_{j=3}^N{\alpha_jy_jK(x_i,x_j)} \tag{10}$$

接下来找 $\alpha_1,\alpha_2$ 的关系，由${8.2}$可得 $y_1\alpha_1 + y_2\alpha_2 = -\sum_{j=3}^N{y_j\alpha_j} = \zeta$。 其中 $\zeta$ 是一个常量，由 $y_i^2 = 1$ (定义的样本类别为{-1,1})，等式两边乘以 $y_1$ 可得：
$$\alpha_1 = y_1*(\zeta - y_2\alpha_2) \tag{11}$$
将$11$带入$9$可得待优化一元函数(忽略常数项)
$$\min\Psi{(\alpha_2)} = {\frac{1}{2}K_{11}(\zeta - y_2\alpha_2)^2 + \frac{1}{2}K_{22}\alpha_2^2 + y_2K_{12}(\zeta - y_2\alpha_2)\alpha_2 - (\zeta - y_2\alpha_2)y_1 - \alpha_2 + (\zeta - y_2\alpha_2)*v_1 + y_2\alpha_2*v_2} \tag{12}$$

接下来对$12$求导可得
$$\frac{\partial\Psi(\alpha_2)}{\partial\alpha_2} = (K_{11} + K_{22} - 2K_{12})\alpha_2 - K_{11}\zeta y_2 + K_{12}\zeta y_2 + y_1y_2 - 1 - y_2 * v_1 + y_2 * v_2 = 0 \tag{13}$$
假设上式求得了 $\alpha_2$ 的解，代入$11$可得 $\alpha_1$，分别记为 $\alpha_1^{new},\alpha_2^{new}$，优化前的解为 $\alpha_1^{old}, \alpha_2^{old}$，则
$$\alpha_1^{new}y_1 + \alpha_2^{new}y_2 = \zeta = \alpha_1^{old}y_1 + \alpha_2^{old}y_2 \tag{14}$$
根据$10$和 $SVMmodel：f(x) = \sum{\alpha_iy_iK(x_i,x_) + b}$:
$$
\begin{align}
v_1 &= f(x_1) - \sum_{j=1}^{2}y_j\alpha_jK_{1j} - b \\
v_2 &= f(x_2) - \sum_{j=1}^{2}y_j\alpha_jK_{2j} - b
\end{align}
$$
带入$13$可得未考虑约问题的解：
$$(K_{11} + K_{22} - 2K_{12})\alpha_2^{new, unclipped} = (K_{11} + K_{22} - 2K_{12})\alpha_2^{old} + y_2(y_2 - y_1 + f(x_1) - f(x_2))$$
定义 $E_i = f(x_i) - y_i$，并记 $\eta = K_{11} + K_{22} - 2K_{12}$，可得：
$$\alpha_2^{new, unclipped} = \alpha_2^{old} + \frac{y_2(E_1 - E_2)}{\eta} \tag{15}$$

式$14$未考虑约束条件：
    * $0 \le \alpha_{i=1, 2} \le C$ 其中 $C$ 为惩罚系数，为用户自己设定
    * $\alpha_1y_1 + \alpha_2y_2 = \zeta$

二维平面上看：
![SMO原始解约束](http://cjxuyshare.me/pic/SMO1)
最优解必须在方框内的直线上取得，因此 $L \le \alpha_2^{new} \le H$：
$$
\begin{align}
y_1 \ne y_2 \to L = max(0, \alpha_2^{old} - \alpha_1^{old});H = min(C, C + \alpha_2^{old} - \alpha_1^{old}) \\
y_2 = y_2 \to L = max(0, \alpha_1^{old} + \alpha_2^{old} - C);H = min(C, \alpha_1^{old} + \alpha_2^{old})
\end{align}
$$
经过上述约束的修剪，可得最终的最优解：
$$
a_2^{new} = \left\{
\begin{array}{c}
\begin{align}
H &,\quad \alpha_2^{new, unclipped} > H \\
\alpha_2^{new, unclipped} &,\quad L \le \alpha_2^{new, unclipped} \le H \\
L &,\quad \alpha_2^{new, unclipped} < L
\end{align}
\end{array}
\right.
\tag{16}
$$
根据$14$可得：
$$\alpha_1^{new} = \alpha_1^{old} + y_1y_2(\alpha_2^{old} - \alpha_2^{new}) \tag{17}$$

一般情况下，有 $\eta > 0$ 但是在两种情况下 $\alpha_2^{new}$ 会取到临界值

1. $\eta < 0$ 时，核函数 $K$ 不满足Mercer定理，矩阵 $K$ 非正定
2. $\eta = 0$ 时，样本 $x_1$ 与 $x_2$ 输入特征相同

因为对待优化函数$12$求二阶导的结果就是 $\eta$，此时

1. $12$为凸函数，没有极小值，极值在边界取得
2. $12$为单调函数，极值同样在边界取得

最后，该如何选取要优化的两个变量呢？

1. 第一个变量的选择为**外循环**，首先遍历整个样本集，选择违反KKT条件的 $\alpha_i$ 为第一个变量，然后选择第二个变量进行优化。**遍历完整个样本集后** 遍历非边界样本集($0 < \alpha_i < C$)中第一个违反KKT条件的 $\alpha_i$，再选择第二个变量进行优化。直到**遍历完非边界样本集**。这样，在整个样本集和非边界样本集上来回切换，直到没有违反KKT条件的变量，然后退出。
2. 第二个变量的选择称为**内循环**，第二个变量的选择希望能使选择的变量由较大的变化，由于 $\alpha_2$ 的选择依赖 $|E_1 - E_2|$，当 $E_1$ 为正，选择最小的 $E_i$ 为 $E_2$，反之选最大。通常每个样本的 $E_i$ 保存在一个列表中，选择最大的 $|E_1 - E_2|$ 来最大化步长
