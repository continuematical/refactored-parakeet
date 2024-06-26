又称学生T检验，用于样本容量较小，统计量服从正态分布，但方差未知的情况。

# 单样本T检验
检验单样本的均值是否和已知总体的均值相等。
## 基本思想
首先，计算出样本均值；其次，根据经验或以往的调查结果，对总体的均值提出一个假设，即`μ=μ0` （`μ0`为待检验的总体均值）；然后，分析计算出的样本均值来自均值为`μ0`的总体的概率，如果概率很小，则认为总体的均值不是`μ0` 。
## 步骤
1. 提出原假设和备择假设
> $H_0$ 原假设：样本均值与总体均值相等 
> $H_1$ 备择假设：样本均值与总体均值不等

2. 确定检验统计量
$$
t=\frac{\overline{X}-\mu_0}{S_{\overline{X}}} =\frac{\overline{X}-\mu_0}{s/\sqrt{n}}
$$
   其中$\overline{X}$ 为样本均值，$\mu_0$ 为总体均值，$s$ 为样本标准差，$n$ 为样本量。
3. 查表
给定显著性水平 $\alpha$ ，根据研究的问题判断是双侧检验还是单侧检验，查 $t$ 临界值表自由度 $df=n-1$ ，得到临界值 $t_{\frac{\alpha} 2}$ 。
如果大于，就拒绝原假设。
# 独立样本T检验
检验两独立样本的均值是否相等。
要求：两样本独立，服从正态分布或近似正态。
# 配对样本T检验
对同一个物体进行前后两次测试，检验两者的差异性。
要求：总体方差相等，数据呈正态。与独立样本T检验相比，要求数据是配对的，两个样本的样本量相同，样本先后的顺序是一一对应的。
