---
layout: post
title:  机器学习（十五）——协同过滤的ALS算法（2）, 主成分分析
category: ML 
---

### Kendall秩相关系数（Kendall rank correlation coefficient）

对于秩变量对$$(x_i,y_i),(x_j,y_j)$$：

$$(x_i-x_j)(y_i-y_j)\begin{cases}
>0, & \text{concordant} \\
=0, & \text{neither concordant nor discordant} \\
<0, & \text{discordant} \\
\end{cases}$$

$$\tau = \frac{(\text{number of concordant pairs}) - (\text{number of discordant pairs})}{n (n-1) /2}$$

>注：Sir Maurice George Kendall，1907~1983，英国统计学家。这个人职业生涯的大部分时间都是一个公务员，二战期间出任英国船运协会副总经理。1949年以后担任伦敦大学教授。

参见：

https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient

### Tanimoto系数

$$T(x,y)=\frac{|X\cap Y|}{|X\cup Y|}=\frac{|X\cap Y|}{|X|+|Y|-|X\cap Y|}=\frac{\sum x_iy_i}{\sqrt{\sum x_i^2}+\sqrt{\sum y_i^2}-\sum x_iy_i}$$

该系数由Taffee T. Tanimoto于1960年提出。Tanimoto生平不详，从名字来看，应该是个日本人。在其他领域，它还有另一个名字Jaccard similarity coefficient。（两者的系数公式一致，但距离公式略有差异。）

>注：Paul Jaccard，1868～1944，苏黎世联邦理工学院（ETH Zurich）博士，苏黎世联邦理工学院植物学教授。ETH Zurich可是出了24个诺贝尔奖得主的。

参见：

https://en.wikipedia.org/wiki/Jaccard_index

## ALS算法原理

http://www.cnblogs.com/luchen927/archive/2012/02/01/2325360.html

机器学习相关——协同过滤

https://mp.weixin.qq.com/s/Xgg61ICqkUo9ovdEV7G8Cw

推荐算法综述

上面的网页概括了ALS算法出现之前的协同过滤算法的概况。

ALS算法是2008年以来，用的比较多的协同过滤算法。它已经集成到Spark的Mllib库中，使用起来比较方便。

从协同过滤的分类来说，ALS算法属于User-Item CF，也叫做混合CF。它同时考虑了User和Item两个方面。

用户和商品的关系，可以抽象为如下的三元组：<User,Item,Rating>。其中，Rating是用户对商品的评分，表征用户对该商品的喜好程度。

假设我们有一批用户数据，其中包含m个User和n个Item，则我们定义Rating矩阵$$R_{m\times n}$$，其中的元素$$r_{ui}$$表示第u个User对第i个Item的评分。

在实际使用中，由于n和m的数量都十分巨大，因此R矩阵的规模很容易就会突破1亿项。这时候，传统的矩阵分解方法对于这么大的数据量已经是很难处理了。

另一方面，一个用户也不可能给所有商品评分，因此，R矩阵注定是个稀疏矩阵。矩阵中所缺失的评分，又叫做missing item。

![](/images/article/ALS.png)

针对这样的特点，我们可以假设用户和商品之间存在若干关联维度（比如用户年龄、性别、受教育程度和商品的外观、价格等），我们只需要将R矩阵投射到这些维度上即可。这个投射的数学表示是：

$$R_{m\times n}\approx X_{m\times k}Y_{n\times k}^T\tag{1}$$

这里的$$\approx$$表明这个投射只是一个近似的空间变换。

不懂这个空间变换的同学，可参见《机器学习（十三）》中的“奇异值分解”的内容，或是《机器学习（十五）》中的“主成分分析”的内容。

一般情况下，k的值远小于n和m的值，从而达到了数据降维的目的。

![](/images/article/ALS_2.png)

幸运的是，我们并不需要显式的定义这些关联维度，而只需要假定它们存在即可，因此这里的关联维度又被称为Latent factor。k的典型取值一般是20～200。

这种方法被称为概率矩阵分解算法(probabilistic matrix factorization，PMF)。ALS算法是PMF在数值计算方面的应用。

为了使低秩矩阵X和Y尽可能地逼近R，需要最小化下面的平方误差损失函数：

$$\min_{x_*,y_*}\sum_{u,i\text{ is known}}(r_{ui}-x_u^Ty_i)^2$$

考虑到矩阵的稳定性问题，使用Tikhonov regularization，则上式变为：

$$\min_{x_*,y_*}L(X,Y)=\min_{x_*,y_*}\sum_{u,i\text{ is known}}(r_{ui}-x_u^Ty_i)^2+\lambda(|x_u|^2+|y_i|^2)\tag{2}$$

优化上式，得到训练结果矩阵$$X_{m\times k},Y_{n\times k}$$。预测时，将User和Item代入$$r_{ui}=x_u^Ty_i$$，即可得到相应的评分预测值。

![](/images/article/ALS_3.png)

同时，矩阵X和Y，还可以用于比较不同的User（或Item）之间的相似度，如下图所示：

![](/images/article/ALS_4.png)

ALS算法的缺点在于：

1.它是一个离线算法。

2.无法准确评估新加入的用户或商品。这个问题也被称为Cold Start问题。

## ALS算法优化过程的推导

公式2的直接优化是很困难的，因为X和Y的二元导数并不容易计算，这时可以使用类似坐标下降法的算法，固定其他维度，而只优化其中一个维度。

对$$x_u$$求导，可得：

$$\begin{align}
\frac{\partial L}{\partial x_u}&=-2\sum_i(r_{ui}-x_u^Ty_i)y_i+2\lambda x_u
\\&=-2\sum_i(r_{ui}-y_i^Tx_u)y_i+2\lambda x_u
\\&=-2Y^Tr_u+2Y^TYx_u+2\lambda x_u
\end{align}$$

令导数为0，可得：

$$Y^TYx_u+\lambda Ix_u=Y^Tr_u\Rightarrow x_u=(Y^TY+\lambda I)^{-1}Y^Tr_u\tag{3}$$

同理，对$$y_i$$求导，由于X和Y是对称的，因此可得类似的结论：

$$y_i=(X^TX+\lambda I)^{-1}X^Tr_i\tag{4}$$

因此整个优化迭代的过程为：

>1.随机生成X、Y。（相当于对迭代算法给出一个初始解。）   
>Repeat until convergence {   
><span style="white-space: pre">	</span>2.固定Y，使用公式3更新$$x_u$$。    
><span style="white-space: pre">	</span>3.固定X，使用公式4更新$$y_i$$。    
>}

一般使用RMSE（root-mean-square error）评估误差是否收敛，具体到这里就是：

$$RMSE=\sqrt{\frac{\sum(R-XY^T)^2}{N}}$$

其中，N为三元组<User,Item,Rating>的个数。当RMSE值变化很小时，就可以认为结果已经收敛。

算法复杂度：

1.求$$x_u$$：$$O(k^2N+k^3m)$$

2.求$$y_i$$：$$O(k^2N+k^3n)$$

可以看出当k一定的时候，这个算法的复杂度是**线性**的。

因为这个迭代过程，交替优化X和Y，因此又被称作交替最小二乘算法（Alternating Least Squares，ALS）。

## 隐式反馈

用户给商品评分是个非常简单粗暴的用户行为。在实际的电商网站中，还有大量的用户行为，同样能够间接反映用户的喜好，比如用户的购买记录、搜索关键字，甚至是鼠标的移动。我们将这些间接用户行为称之为隐式反馈（implicit feedback），以区别于评分这样的显式反馈（explicit feedback）。

隐式反馈有以下几个特点：

1.没有负面反馈（negative feedback）。用户一般会直接忽略不喜欢的商品，而不是给予负面评价。

2.隐式反馈包含大量噪声。比如，电视机在某一时间播放某一节目，然而用户已经睡着了，或者忘了换台。

3.显式反馈表现的是用户的**喜好（preference）**，而隐式反馈表现的是用户的**信任（confidence）**。比如用户最喜欢的一般是电影，但观看时间最长的却是连续剧。大米购买的比较频繁，量也大，但未必是用户最想吃的食物。

4.隐式反馈非常难以量化。

## ALS-WR

针对隐式反馈，有ALS-WR算法（ALS with Weighted-$$\lambda$$-Regularization）。

首先将用户反馈分类：

$$p_{ui}=\begin{cases}
1, & \text{preference} \\
0, & \text{no preference} \\
\end{cases}$$

但是喜好是有程度差异的，因此需要定义程度系数：

$$c_{ui}=1+\alpha r_{ui}$$

这里的$$r_{ui}$$表示原始量化值，比如观看电影的时间；

这个公式里的1表示最低信任度，$$\alpha$$表示根据用户行为所增加的信任度。

最终，损失函数变为：

$$\min_{x_*,y_*}L(X,Y)=\min_{x_*,y_*}\sum_{u,i}c_{ui}(p_{ui}-x_u^Ty_i)^2+\lambda(\sum_u|x_u|^2+\sum_i|y_i|^2)$$

除此之外，我们还可以使用指数函数来定义$$c_{ui}$$：

$$c_{ui}=1+\alpha \log(1+r_{ui}/\epsilon)$$

ALS-WR没有考虑到时序行为的影响，时序行为相关的内容，可参见：

http://www.jos.org.cn/1000-9825/4478.htm

基于时序行为的协同过滤推荐算法

## 参考

参考论文：

《Large-scale Parallel Collaborative Filtering forthe Netflix Prize》

《Collaborative Filtering for Implicit Feedback Datasets》

《Matrix Factorization Techniques for Recommender Systems》

其他参考：

http://www.jos.org.cn/html/2014/9/4648.htm

基于大规模隐式反馈的个性化推荐

http://www.fuqingchuan.com/2015/03/812.html

协同过滤之ALS-WR算法

http://www.docin.com/p-714582034.html

基于矩阵分解的协同过滤算法

http://www.tuicool.com/articles/fANvieZ

Spark MLlib中的协同过滤

http://www.68idc.cn/help/buildlang/ask/20150727462819.html

Alternating Least Squares(ASL) 的数学推导

https://mp.weixin.qq.com/s/bRhIm8Xvlb51zE2HpDO5Og

一文读懂推荐系统知识体系

http://mp.weixin.qq.com/s/QhP3wRGbrO7sYSDNm8z0gQ

常用推荐算法（50页干货）

https://zhuanlan.zhihu.com/p/23036112

推荐系统常用的推荐算法

# 主成分分析

真实的训练数据总是存在各种各样的问题。

比如拿到一个汽车的样本，里面既有以“千米/每小时”度量的最大速度特征，也有“英里/小时”的最大速度特征。显然这两个特征有一个是多余的，我们需要找到，并去除这个冗余。

再比如，针对飞行员的调查，包含两个特征：飞行的技能水平和对飞行的爱好程度。由于飞行员是很难培训的，因此如果没有对飞行的热爱，也就很难学好飞行。所以这两个特征实际上是强相关的（strongly correlated）。如下图所示：

![](/images/article/PCA.png)

我们的目标就是找出上图中所示的向量$$u_1$$。

为了实现这两个目标，我们可以采用PCA（Principal components analysis）算法。

## 数据的规则化处理

在进行PCA算法之前，我们首先要对数据进行预处理，使之规则化。其方法如下：

>1.$$\mu=\frac{1}{m}\sum_{i=1}^mx^{(i)}$$   
>2.$$x^{(i)}:=x^{(i)}-\mu$$   
>3.$$\sigma_j^2=\frac{1}{m}\sum_i(x^{(i)})^2$$   
>4.$$x_j^{(i)}:=x_j^{(i)}/\sigma_j$$   

多数情况下，特征空间中，不同特征向量所代表的维度之间，并不能直接比较。

比如，摄氏度和华氏度，虽然都是温度的单位，但两种温标的原点和尺度都不相同，因此需要规范化之后才能比较。

步骤1和2，用于消除原点偏差（常数项偏差）。步骤3和4，用于统一尺度（一次项偏差）。

虽然上面的办法，对于二次以上的偏差无能为力，然而多数情况下，这种处理，已经比原始状态好多了。

