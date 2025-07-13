---
title: 主成分分析PCA-两个小问题
date: 2025-07-11
slug: blog-post-slug
tags:
  - 机器学习
  - 主成分分析PCA
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
### **问题16.2：证明样本协方差矩阵S是总体协方差矩阵Σ的无偏估计**

**【问题陈述】**
证明：$E[\mathbf{S}] = \mathbf{\Sigma}$。

**【无偏估计 (Unbiased Estimator)】**
在统计学中，“无偏估计”是一个非常重要的概念。它指的是，如果我们使用某个估计量（比如样本协方差 $s_{ij}$）去估计一个总体的真实参数（比如总体协方差 $\sigma_{ij}$），这个估计量的期望值（即在无数次重复抽样后，该估计量的平均值）应该恰好等于那个我们想要估计的真实参数。
证明 $E[\mathbf{S}] = \mathbf{\Sigma}$，等价于证明矩阵中的每一个元素都满足期望相等，即证明 $E[s_{ij}] = \sigma_{ij}$ 对所有 $i, j$ 成立。

---

**【证明过程】**

**第一步：写下定义**
我们从样本协方差 $s_{ij}$ 的定义出发：

$$
s_{ij} = \frac{1}{n-1} \sum_{k=1}^n (x_{ik} - \bar{x}_i)(x_{jk} - \bar{x}_j)
$$

我们的目标是计算 $E[s_{ij}]$。利用期望的线性性质，我们可以将期望算子放入求和号内部：

$$
E[s_{ij}] = \frac{1}{n-1} \sum_{k=1}^n E\left[ (x_{ik} - \bar{x}_i)(x_{jk} - \bar{x}_j) \right]
$$

**第二步：处理期望项**
现在的关键是计算 $E\left[ (x_{ik} - \bar{x}_i)(x_{jk} - \bar{x}_j) \right]$。这是一个技巧性很强的步骤，我们通过引入总体均值 $\mu_i$ 和 $\mu_j$ 来拆解它：

$$
\begin{align*}
(x_{ik} - \bar{x}_i) &= (x_{ik} - \mu_i) - (\bar{x}_i - \mu_i) \\
(x_{jk} - \bar{x}_j) &= (x_{jk} - \mu_j) - (\bar{x}_j - \mu_j)
\end{align*}
$$

将它们相乘并展开，得到四项：

$$
E\left[ \dots \right] = E\left[ (x_{ik}-\mu_i)(x_{jk}-\mu_j) \right] - E\left[ (x_{ik}-\mu_i)(\bar{x}_j-\mu_j) \right] - E\left[ (\bar{x}_i-\mu_i)(x_{jk}-\mu_j) \right] + E\left[ (\bar{x}_i-\mu_i)(\bar{x}_j-\mu_j) \right]
$$

现在我们逐一计算这四项的期望：

1.  **第一项**: $E\left[ (x_{ik}-\mu_i)(x_{jk}-\mu_j) \right]$
    根据总体协方差的定义，这正是 $\text{cov}(x_{ik}, x_{jk}) = \sigma_{ij}$。

2.  **第四项**: $E\left[ (\bar{x}_i-\mu_i)(\bar{x}_j-\mu_j) \right]$
    这正是样本均值 $\bar{x}_i$ 和 $\bar{x}_j$ 的协方差。对于独立同分布的样本，我们有 $\text{cov}(\bar{x}_i, \bar{x}_j) = \frac{\sigma_{ij}}{n}$。
    *(简要证明: $\text{cov}(\frac{1}{n}\sum_k x_{ik}, \frac{1}{n}\sum_l x_{jl}) = \frac{1}{n^2} \sum_k \sum_l \text{cov}(x_{ik}, x_{jl})$。由于样本独立，只有当 $k=l$ 时协方差不为零，为 $\sigma_{ij}$。这样的项共有 $n$ 个，所以结果是 $\frac{1}{n^2} \cdot n \sigma_{ij} = \frac{\sigma_{ij}}{n}$)*

3.  **第二项 (和第三项)**: $E\left[ (x_{ik}-\mu_i)(\bar{x}_j-\mu_j) \right]$
    我们将 $\bar{x}_j$ 的定义代入：
    
    $$
    E\left[ (x_{ik}-\mu_i) \left( \frac{1}{n}\sum_{l=1}^n x_{jl} - \mu_j \right) \right] = E\left[ (x_{ik}-\mu_i) \frac{1}{n}\sum_{l=1}^n (x_{jl} - \mu_j) \right] = \frac{1}{n} \sum_{l=1}^n E\left[ (x_{ik}-\mu_i)(x_{jl} - \mu_j) \right]
    $$
    
    在求和中，由于样本是独立的，只有当 $l=k$ 时，$E[\dots]$ 不为零，其值为 $\sigma_{ij}$。当 $l \ne k$ 时，期望为 $E[x_{ik}-\mu_i]E[x_{jl} - \mu_j] = 0 \cdot 0 = 0$。
    因此，整个求和只有一项非零，结果是 $\frac{1}{n}\sigma_{ij}$。第三项同理也是 $\frac{1}{n}\sigma_{ij}$。

**第三步：组合结果**
将这四项的结果代回，得到核心期望项的值：

$$
E\left[ (x_{ik} - \bar{x}_i)(x_{jk} - \bar{x}_j) \right] = \sigma_{ij} - \frac{\sigma_{ij}}{n} - \frac{\sigma_{ij}}{n} + \frac{\sigma_{ij}}{n} = \sigma_{ij} - \frac{\sigma_{ij}}{n} = \frac{n-1}{n}\sigma_{ij}
$$

**第四步：完成最终计算**
现在，我们将上一步的结果代回到 $E[s_{ij}]$ 的表达式中：

$$
\begin{align*}
E[s_{ij}] &= \frac{1}{n-1} \sum_{k=1}^n \left( \frac{n-1}{n}\sigma_{ij} \right) \\
&= \frac{1}{n-1} \cdot n \cdot \left( \frac{n-1}{n}\sigma_{ij} \right) && \text{（因为求和项与k无关，直接乘以n）} \\
&= \frac{n(n-1)}{n(n-1)} \sigma_{ij} \\
&= \sigma_{ij}
\end{align*}
$$

**证毕。** 这完美地解释了为什么样本协方差的分母是 $n-1$ 而不是 $n$。这个 $n-1$ 因子（贝塞尔校正）正是为了确保样本协方差是总体协方差的一个无偏估计。

---

### **问题16.3：解释PCA为何等价于求解低秩近似问题**

**【问题陈述】**
设 $\mathbf{X}$ 为数据规范化样本矩阵，则主成分分析等价于求解以下最优化问题：

$$
\min_{L} \quad \|\mathbf{X} - \mathbf{L}\|_F^2
$$

$$
\text{s.t.} \quad \text{rank}(\mathbf{L}) \le k
$$

其中，$F$ 是弗罗贝尼乌斯范数，$k$ 是主成分个数。试问为什么？

**【两种视角的统一】**
这个问题揭示了PCA的第二个、也是同样深刻的本质。我们之前学习的PCA，可以被称为“**最大方差理论**”——我们寻找一个子空间，使得数据投影到该子空间后，方差最大。

而这个问题，是从“**最小重构误差理论**”的角度来看待PCA——我们寻找一个“更简单”的矩阵 $\mathbf{L}$（秩不大于 $k$）来近似原始数据矩阵 $\mathbf{X}$，使得近似带来的误差（即原始点与近似点之间的距离平方和）最小。

这个问题的答案，就在于证明这两个视角导向了同一个解。其间的桥梁，正是**奇异值分解（SVD）** 和一个著名的定理。

---

**【证明】**

**第一步：理解优化问题**

*   **数据矩阵 X ($m \times n$)**: 每一列是一个样本点。
*   **近似矩阵 L ($m \times n$)**: 我们希望找到的、对 $\mathbf{X}$ 的最佳近似。
*   **约束 rank(L) ≤ k**: 这是问题的核心。“秩”可以理解为矩阵所包含的线性无关信息的维度。约束 $\mathbf{L}$ 的秩不大于 $k$ (其中 $k < m, n$)，意味着我们强迫 $\mathbf{L}$ 是一个“简单”的、“低维”的矩阵。它的所有列向量都必须位于一个维度不超过 $k$ 的子空间中。
*   **目标函数 $\|\mathbf{X} - \mathbf{L}\|_F^2$**: 弗罗贝尼乌斯范数的平方，$\|\mathbf{A}\|_F^2 = \sum_i \sum_j a_{ij}^2$。它衡量了两个矩阵之间所有元素差异的平方和。在几何上，这完全等价于数据矩阵 $\mathbf{X}$ 的所有样本点与近似矩阵 $\mathbf{L}$ 对应的所有点之间的欧氏距离的平方总和。我们的目标就是**最小化这个总的重构误差**。

**第二步：引入定理——Eckart-Young-Mirsky定理**

这个著名的定理直接给出了上述低秩近似问题的解析解。

**定理内容**: 对于任意矩阵 $\mathbf{X}$，其SVD分解为 $\mathbf{X} = \mathbf{U\Sigma V}^T$。在所有秩不大于 $k$ 的矩阵 $\mathbf{L}$ 中，能够最小化弗罗贝尼乌斯范数 $\|\mathbf{X} - \mathbf{L}\|_F^2$ 的最优解 $\mathbf{L}^*$ 是由 $\mathbf{X}$ 的**截断SVD (Truncated SVD)** 给出的：

$$
\mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T
$$

其中，$\mathbf{U}_k$ 是 $\mathbf{U}$ 的前 $k$ 列，$\mathbf{\Sigma}_k$ 是 $\mathbf{\Sigma}$ 的左上角 $k \times k$ 子矩阵，$\mathbf{V}_k$ 是 $\mathbf{V}$ 的前 $k$ 列。

SVD将矩阵 $\mathbf{X}$ 分解为一系列“秩为1的层”的叠加。这个定理告诉我们，要想得到最佳的低秩近似，只需保留那些最重要的层（对应最大的 $k$ 个奇异值），然后将其他次要的层全部丢弃。

**第三步：建立与PCA的联系**

现在，我们只需要证明这个由最小重构误差得到的最优解 $\mathbf{L}^*$，与我们通过最大化方差得到的PCA结果是等价的。

1.  **PCA的重构**: 在PCA中，我们首先找到前 $k$ 个主成分方向，它们构成了矩阵 $\mathbf{V}_k$ 的列（我们在16.2.3节证明了主成分方向就是SVD的右奇异向量）。然后，我们将原始数据 $\mathbf{X}$ 投影到这 $k$ 个主成分张成的子空间中。这个投影后的数据（即重构的数据）可以表示为：
    
    $$
    \mathbf{X}_{\text{recon}} = (\mathbf{V}_k \mathbf{V}_k^T) \mathbf{X}
    $$
    
    *(注：$\mathbf{V}_k\mathbf{V}_k^T$ 是一个投影算子，它将任何向量投影到 $\mathbf{V}_k$ 的列空间上)*

2.  **证明 $\mathbf{L}^* = \mathbf{X}_{\text{recon}}$**: 我们需要证明 Eckart-Young-Mirsky 定理给出的解 $\mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T$ 和 PCA 的重构矩阵 $\mathbf{X}_{\text{recon}} = (\mathbf{V}_k \mathbf{V}_k^T) \mathbf{X}$ 是同一个矩阵。
    从 $\mathbf{X}$ 的SVD分解 $\mathbf{X} = \mathbf{U\Sigma V}^T$ 出发：
    
    $$
    \begin{align*}
    \mathbf{X}_{\text{recon}} &= (\mathbf{V}_k \mathbf{V}_k^T) (\mathbf{U\Sigma V}^T) \\
    &= \mathbf{V}_k (\mathbf{V}_k^T \mathbf{U}) \mathbf{\Sigma} \mathbf{V}^T
    \end{align*}
    $$
    
    由于 $\mathbf{V}$ 和 $\mathbf{U}$ 的列向量各自构成标准正交基，$\mathbf{V}_k^T \mathbf{U}$ 这一项并不好直接化简。我们换个思路。
    我们知道 $\mathbf{X V} = \mathbf{U\Sigma}$。只取前 $k$ 列，我们有 $\mathbf{X V}_k = \mathbf{U}_k \mathbf{\Sigma}_k$。
    现在，让我们从 $\mathbf{L}^*$ 出发：
    
    $$
    \mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T
    $$
    
    将 $\mathbf{U}_k \mathbf{\Sigma}_k = \mathbf{X V}_k$ 代入：
    
    $$
    \mathbf{L}^* = (\mathbf{X V}_k) \mathbf{V}_k^T
    $$
    
    这个形式 $\mathbf{X} (\mathbf{V}_k \mathbf{V}_k^T)$ 似乎是投影到行空间，而不是列空间。
    
    让我们重新审视一下投影的定义。PCA是将样本点（$\mathbf{X}$的**列向量**）投影到由主成分（$\mathbf{V}_k$的**列向量**）张成的子空间中。一个列向量 $\mathbf{c}$ 投影到由标准正交基 $\mathbf{V}_k$ 张成的空间，其结果是 $(\mathbf{V}_k \mathbf{V}_k^T)\mathbf{c}$。将此应用于 $\mathbf{X}$ 的所有列，得到重构矩阵 $\mathbf{X}_{\text{recon}} = (\mathbf{V}_k \mathbf{V}_k^T)\mathbf{X}$。
    
    Eckart-Young-Mirsky定理的解 $\mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T$ 是一个秩为 $k$ 的矩阵，它的**行空间**由 $\mathbf{V}_k$ 的列张成，**列空间**由 $\mathbf{U}_k$ 的列张成。
    
    $\mathbf{X}_{\text{recon}}$ 的列向量是原始列向量的投影，所以 $\mathbf{X}_{\text{recon}}$ 的列也位于由 $\mathbf{V}_k$ 张成的子空间中。但这是不对的，样本点是 $m$ 维的，主成分方向也是 $m$ 维的，所以列空间是 $m$ 维的。
    
    现在，让我们回到最根本的定义上。
    *   $\mathbf{X}$ 的列是 $m$ 维样本。
    *   $\mathbf{S} = \frac{1}{n-1}\mathbf{X}\mathbf{X}^T$ 是 $m \times m$ 的协方差矩阵。
    *   $\mathbf{S}$ 的特征向量（主成分方向）是 $m$ 维的，我们记为 $\mathbf{A}$ 矩阵。
    *   $\mathbf{X}$ 的SVD是 $\mathbf{X} = \mathbf{U\Sigma V}^T$。$\mathbf{U}$ 是 $m \times m$ 的，$\mathbf{V}$ 是 $n \times n$ 的。
    *   协方差矩阵 $\mathbf{S} = \frac{1}{n-1}\mathbf{X}\mathbf{X}^T = \frac{1}{n-1}(\mathbf{U\Sigma V}^T)(\mathbf{U\Sigma V}^T)^T = \frac{1}{n-1} \mathbf{U\Sigma\Sigma}^T\mathbf{U}^T$。
    *   所以，主成分方向矩阵 $\mathbf{A}$ 其实是SVD中的**左奇异向量矩阵 U**！
    
    这是之前讨论中一个非常关键且容易混淆的点，取决于数据矩阵的组织方式！李航老师在 16.2.3 节中定义的 $\mathbf{X}'$ 是 $n \times m$ 的，所以其右奇异向量是 $m \times m$ 的，对应主成分。但在这里的习题中，$\mathbf{X}$ 是 $m \times n$ 的。
    
    **让我们以 $\mathbf{X}$ 是 $m \times n$ 的标准设定重新梳理：**
    1.  **PCA的目标**: 找到 $k$ 个 $m$ 维的标准正交基向量 $\mathbf{A}_k=[\mathbf{a}_1, \dots, \mathbf{a}_k]$，它们是协方差矩阵 $\mathbf{S}=\frac{1}{n-1}\mathbf{XX}^T$ 的前 $k$ 个特征向量。
    2.  **SVD与PCA的关系**: 对 $\mathbf{X}$ 做SVD, $\mathbf{X} = \mathbf{U\Sigma V}^T$。协方差矩阵 $\mathbf{S} = \frac{1}{n-1}\mathbf{U\Sigma\Sigma}^T\mathbf{U}^T$。这是一个标准的特征值分解形式，所以主成分方向 $\mathbf{A}_k$ 就是**左奇异向量矩阵 $\mathbf{U}_k$**。
    3.  **PCA重构**: 原始数据投影到主成分子空间上，重构后的数据为 $\mathbf{X}_{\text{recon}} = (\mathbf{U}_k\mathbf{U}_k^T)\mathbf{X}$。
    4.  **低秩近似解**: Eckart-Young-Mirsky定理的解是 $\mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T$。
    5.  **证明等价**: 我们来证明 $\mathbf{X}_{\text{recon}} = \mathbf{L}^*$。
        
        $$
        \begin{align*}
        \mathbf{X}_{\text{recon}} &= (\mathbf{U}_k\mathbf{U}_k^T)\mathbf{X} \\
        &= (\mathbf{U}_k\mathbf{U}_k^T) (\mathbf{U\Sigma V}^T) \\
        &= \mathbf{U}_k (\mathbf{U}_k^T \mathbf{U}) \mathbf{\Sigma} \mathbf{V}^T
        \end{align*}
        $$
        
        由于 $\mathbf{U}$ 的列是标准正交的，$\mathbf{U}_k^T \mathbf{U}$ 的结果是一个 $k \times m$ 的矩阵，其形式为 $[\mathbf{I}_k | \mathbf{0}]$。
        所以 $(\mathbf{U}_k^T \mathbf{U}) \mathbf{\Sigma}$ 的结果就是将 $\mathbf{\Sigma}$ 的前 $k$ 行保留，其余行置零。这恰好可以表示为 $\mathbf{\Sigma}_k \mathbf{V}_k^T$ 的母体形式。
        更简洁地：$(\mathbf{U}_k\mathbf{U}_k^T)\mathbf{U} = \mathbf{U}_k$。不对。
        
        让我们用一个更清晰的视角：
        $\mathbf{L}^* = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T$。这是截断SVD。
        $\mathbf{X}_{\text{recon}}$ 是 $\mathbf{X}$ 在由 $\mathbf{U}_k$ 的列向量张成的子空间上的投影。一个矩阵 $\mathbf{X}$ 在一个正交基 $\mathbf{U}_k$ 上的投影的公式是 $\mathbf{U}_k (\mathbf{U}_k^T \mathbf{X})$。
        
        $$
        \mathbf{X}_{\text{recon}} = \mathbf{U}_k (\mathbf{U}_k^T (\mathbf{U\Sigma V}^T)) = \mathbf{U}_k ([\mathbf{I}_k|\mathbf{0}] \mathbf{\Sigma V}^T) = \mathbf{U}_k (\mathbf{\Sigma}_k \mathbf{V}_k^T) = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T = \mathbf{L}^*
        $$
        **证明完成！**

**【结论】**
主成分分析等价于求解低秩近似问题，原因如下：
1.  PCA的“最大化方差”目标，其解是由协方差矩阵的前 $k$ 个特征向量张成的子空间。在数据矩阵 $\mathbf{X}$ ($m \times n$) 的SVD中，这个子空间由前 $k$ 个**左奇异向量** $\mathbf{U}_k$ 的列向量张成。
2.  低秩近似的“最小化重构误差”目标，根据Eckart-Young-Mirsky定理，其解 $\mathbf{L}^*$ 是 $\mathbf{X}$ 的截断SVD。这个最优的近似矩阵 $\mathbf{L}^*$ 正是原始数据矩阵 $\mathbf{X}$ 在由其前 $k$ 个左奇异向量 $\mathbf{U}_k$ 张成的子空间上的正交投影。
3.  **两个过程殊途同归，都指向了由 $\mathbf{U}_k$ 定义的同一个 $k$ 维子空间**。因此，PCA可以被视为一种通过低秩近似来寻找数据内在结构、实现降维和去噪的强大工具。这两种视角是PCA这枚硬币的两个面，相辅相成，共同构成了我们对它的完整理解。