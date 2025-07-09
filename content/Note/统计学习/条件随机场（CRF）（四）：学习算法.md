---
title: 条件随机场（CRF）（四）：学习算法
date: 2025-07-09
slug: blog-post-slug
tags:
  - 机器学习
  - 条件随机场CRF
categories:
  - 笔记
description: 描述
draft: false
state: "0"
---
### 1. 学习问题概述：目标与思路

CRF的学习是一个典型的**参数估计**问题。我们的目标是，给定一个训练数据集 $\mathcal{D} = \{(X^{(1)}, Y^{(1)}), (X^{(2)}, Y^{(2)}), \dots, (X^{(N)}, Y^{(N)})\}$，其中包含 $N$ 个观测序列和它们对应的真实标签序列，我们需要找到一组参数 $\theta = \{\theta_1, \theta_2, \dots, \theta_K\}$，使得这组参数能够最好地“解释”我们的训练数据。

#### 目标函数：最大化对数似然

在概率模型中，“最好地解释”通常意味着让训练数据出现的概率最大。这个原则被称为**最大似然估计 (Maximum Likelihood Estimation, MLE)**。

对于整个训练集，我们假设每个样本 $(X^{(i)}, Y^{(i)})$ 是独立同分布的。因此，整个数据集的似然函数是所有单个样本条件概率的连乘：

$$
L(\theta) = \prod_{i=1}^{N} p(Y^{(i)}|X^{(i)}; \theta)
$$

直接优化连乘在数学上很困难，而且容易导致数值下溢。因此，我们通常优化其**对数似然 (Log-Likelihood)**，因为对数函数是单调递增的，最大化对数似然等价于最大化原始似然。

$$
\mathcal{L}(\theta) = \log L(\theta) = \sum_{i=1}^{N} \log p(Y^{(i)}|X^{(i)}; \theta)
$$

将我们在[简化形式](Note/统计学习/条件随机场（CRF）（二）：模型定义与参数化.md#简化形式)中推导出的CRF条件概率公式代入：

$$
p(Y|X; \theta) = \frac{\exp \left( \sum_{t=1}^{T} \sum_{k=1}^{K} \theta_k f_k(y_{t-1}, y_t, X, t) \right)}{Z(X; \theta)}
$$

代入对数似然函数中，利用 $\log(a/b) = \log(a) - \log(b)$ 和 $\log(\exp(c))=c$，我们得到：

$$
\mathcal{L}(\theta) = \sum_{i=1}^{N} \left( \sum_{t=1}^{T_i} \sum_{k=1}^{K} \theta_k f_k(y^{(i)}_{t-1}, y^{(i)}_t, X^{(i)}, t) - \log Z(X^{(i)}; \theta) \right)
$$

(为了简洁，后续的推导将省略对样本索引 $i$ 的求和，只关注单个样本的对数似然，最后的结果对所有样本求和即可。)

#### 正则化：防止过拟合

在实际应用中，特征数量 $K$ 可能非常大，直接最大化对数似然容易导致模型**过拟合**。为了防止这种情况，我们通常会在目标函数中加入一个**正则化项 (Regularization Term)**，对参数的复杂度进行惩罚。最常用的是 **L2正则化**，它惩罚参数的平方和：

$$
\mathcal{L}_{\text{reg}}(\theta) = \mathcal{L}(\theta) - \frac{1}{2\sigma^2} \sum_{k=1}^{K} \theta_k^2
$$

这里的 $\sigma^2$ 是一个超参数，控制正则化的强度。加入正则化项后，目标函数变成了一个**严格凸函数**，这意味着它有唯一的全局最优解，这对优化算法非常友好。

#### 优化思路：梯度上升法

我们的目标函数 $\mathcal{L}_{\text{reg}}(\theta)$ 是一个关于 $\theta$ 的连续可导函数。解决这类优化问题的标准方法是**基于梯度的优化算法**。最简单的就是**梯度上升法 (Gradient Ascent)**，其思想是：

1.  随机初始化参数 $\theta$。
2.  计算目标函数在当前 $\theta$ 点的**梯度 (gradient)** $\nabla \mathcal{L}_{\text{reg}}(\theta)$。梯度是一个向量，指向函数值增长最快的方向。
3.  沿着梯度方向更新参数：$\theta_{\text{new}} \leftarrow \theta_{\text{old}} + \eta \nabla \mathcal{L}_{\text{reg}}(\theta)$，其中 $\eta$ 是学习率。
4.  重复步骤2和3，直到收敛。

现在，整个学习问题的核心就变成了：**如何计算目标函数的梯度？**

### 2. 对数似然函数的梯度推导

这是整个CRF学习理论的关键了。我们需要对单个样本的对数似然函数（加上正则化项）关于某一个参数 $\theta_j$ 求偏导数。

目标函数（单个样本，为简化书写，省略 $X, Y$ 等的上标 $(i)$ 和下标 $t$）：

$$
\mathcal{L}(\theta) = \left( \sum_{k=1}^{K} \theta_k \sum_{t=1}^{T} f_k(y_{t-1}, y_t, X, t) \right) - \log Z(X; \theta) - \frac{1}{2\sigma^2} \sum_{k=1}^{K} \theta_k^2
$$

我们对 $\theta_j$ 求偏导：

$$
\frac{\partial \mathcal{L}(\theta)}{\partial \theta_j} = \frac{\partial}{\partial \theta_j} \left( \sum_{k} \theta_k \sum_{t} f_k \right) - \frac{\partial}{\partial \theta_j} \log Z(X; \theta) - \frac{\partial}{\partial \theta_j} \left( \frac{1}{2\sigma^2} \sum_{k} \theta_k^2 \right)
$$

这个式子可以分解为三个部分进行计算：

#### 第一部分：对第一项求导

$$
\frac{\partial}{\partial \theta_j} \left( \sum_{k} \theta_k \sum_{t} f_k(y_{t-1}, y_t, X, t) \right)
$$

这是一个简单的线性求导。只有当 $k=j$ 时，导数才不为零。

$$
\begin{align*}
\text{Part 1} &= \sum_{t=1}^{T} f_j(y_{t-1}, y_t, X, t) && \text{（$\theta_k$的系数，只有当k=j时留下）}
\end{align*}
$$

这个结果有一个非常直观的解释：它是在**给定的真实标签序列 $Y$** 上，特征 $f_j$ 被激活的总次数。我们称之为特征 $f_j$ 的**经验期望 (Empirical Expectation)**。

#### 第二部分：对 $\log Z(X)$ 求导
这是最关键也最复杂的一步。我们需要用到链式法则和对数函数的求导法则 $\frac{d}{dx}\log u = \frac{1}{u}\frac{du}{dx}$。

$$
\frac{\partial}{\partial \theta_j} \log Z(X; \theta) = \frac{1}{Z(X; \theta)} \frac{\partial Z(X; \theta)}{\partial \theta_j}
$$

现在问题变成了求 $\frac{\partial Z(X; \theta)}{\partial \theta_j}$。我们先把 $Z(X)$ 的定义写出来：

$$
Z(X; \theta) = \sum_{Y'} \exp \left( \sum_{k=1}^{K} \theta_k \sum_{t=1}^{T} f_k(y'_{t-1}, y'_t, X, t) \right)
$$

其中 $Y'$ 遍历所有可能的标签序列。我们对它求导：

$$
\begin{align*}
\frac{\partial Z(X; \theta)}{\partial \theta_j} &= \frac{\partial}{\partial \theta_j} \sum_{Y'} \exp \left( \dots \right) && \text{（准备求导）} \\
&= \sum_{Y'} \frac{\partial}{\partial \theta_j} \exp \left( \dots \right) && \text{（求和与求导可交换顺序）} \\
&= \sum_{Y'} \exp \left( \dots \right) \cdot \frac{\partial}{\partial \theta_j} \left( \sum_{k} \theta_k \sum_{t} f_k(y'_{t-1}, y'_t, X, t) \right) && \text{（应用指数函数链式法则 $\frac{d}{dx}e^u = e^u \frac{du}{dx}$）} \\
&= \sum_{Y'} \exp \left( \dots \right) \cdot \left( \sum_{t=1}^{T} f_j(y'_{t-1}, y'_t, X, t) \right) && \text{（对内部的线性项求导，只有k=j时留下）}
\end{align*}
$$

现在，我们把这个结果代回到 $\frac{1}{Z(X)} \frac{\partial Z(X)}{\partial \theta_j}$ 中：

$$
\begin{align*}
\text{Part 2} &= \frac{1}{Z(X; \theta)} \sum_{Y'} \exp \left( \dots \right) \cdot \left( \sum_{t=1}^{T} f_j(y'_{t-1}, y'_t, X, t) \right) && \text{（代入结果）} \\
&= \sum_{Y'} \frac{\exp \left( \dots \right)}{Z(X; \theta)} \cdot \left( \sum_{t=1}^{T} f_j(y'_{t-1}, y'_t, X, t) \right) && \text{（将分母移入求和号）} \\
&= \sum_{Y'} p(Y'|X; \theta) \cdot \left( \sum_{t=1}^{T} f_j(y'_{t-1}, y'_t, X, t) \right) && \text{（$\frac{\exp(\dots)}{Z(X)}$正是$p(Y'|X)$的定义）}
\end{align*}
$$

这个结果同样有非常漂亮的解释：它是特征 $f_j$ 在**当前模型 $p(Y|X; \theta)$ 所定义的概率分布下**，被激活次数的**期望值**。我们称之为**模型期望 (Model Expectation)**。

**关键连接**：如何计算这个模型期望？
直接按定义遍历所有 $Y'$ 是不可行的。但我们可以利用求和的线性性质进行变换：

$$
\begin{align*}
E_{p(Y'|X)}[f_j] &= \sum_{Y'} p(Y'|X) \sum_{t=1}^{T} f_j(y'_{t-1}, y'_t, X, t) \\
&= \sum_{t=1}^{T} \sum_{Y'} p(Y'|X) f_j(y'_{t-1}, y'_t, X, t) \\
&= \sum_{t=1}^{T} \sum_{i,m} p(y_{t-1}=i, y_t=m | X) \cdot f_j(y_{t-1}=i, y_t=m, X, t)
\end{align*}
$$

看！最后这个式子中的边缘概率 $p(y_{t-1}=i, y_t=m | X)$，**正是我们在[结合前向-后向变量计算边缘概率](Note/统计学习/条件随机场（CRF）（三）：三大核心问题之解码与概率计算.md#结合前向-后向变量计算边缘概率)中学习的、可以用前向-后向算法高效计算出来的量！** 这就将学习算法和推断算法完美地连接在了一起。

#### 第三部分：对正则化项求导
这是最简单的部分：

$$
\frac{\partial}{\partial \theta_j} \left( \frac{1}{2\sigma^2} \sum_{k=1}^{K} \theta_k^2 \right) = \frac{1}{2\sigma^2} \cdot 2\theta_j = \frac{\theta_j}{\sigma^2}
$$

#### 梯度最终形式

将三部分组合起来，我们得到梯度向量的第 $j$ 个分量：

$$
\boxed{
\frac{\partial \mathcal{L}(\theta)}{\partial \theta_j} = \left( \sum_{t=1}^{T} f_j(y_{t-1}, y_t, X, t) \right) - \left( \sum_{t=1}^{T} \sum_{i,m} p(y_{t-1}=i, y_t=m | X) \cdot f_j(i, m, X, t) \right) - \frac{\theta_j}{\sigma^2}
}
$$

这个公式可以概括为：

$$
\text{梯度} = \text{经验期望} - \text{模型期望} - \text{正则化惩罚}
$$

当梯度为0时（即找到最优解时），模型期望就等于经验期望。这意味着，最优的模型参数会让模型预测出的特征期望与真实数据中的特征期望完全一致。

### 3. 具体的优化算法

有了计算梯度的能力，我们就可以使用任何基于梯度的优化器来训练CRF了。
*   **改进的迭代尺度法 (Improved Iterative Scaling, IIS)**：这是早期CRF论文中提出的一种方法。它不直接使用梯度，而是通过一种迭代的方式保证每一步更新都让似然函数值增加。但它要求特征函数 $f_k$ 非负，且收敛速度较慢，现在已不常用。
*   **拟牛顿法 (Quasi-Newton Methods)，特别是 L-BFGS**：
    *   简单的梯度上升法只利用了一阶导数（梯度），收敛较慢。牛顿法利用二阶导数（Hessian矩阵）来更准确地找到下降方向，收敛快，但计算和存储Hessian矩阵的代价 ($O(K^2)$) 极其高昂。
    *   **L-BFGS (Limited-memory Broyden–Fletcher–Goldfarb–Shanno)** 是一种拟牛顿法。它通过存储过去几次的梯度和参数更新信息，来**近似**模拟Hessian矩阵的作用，而无需显式计算它。
    *   L-BFGS在收敛速度和内存开销之间取得了绝佳的平衡，是目前训练CRF以及其他大规模机器学习模型的**黄金标准**和首选算法。

**CRF的完整学习流程如下：**

1.  给定训练集 $\mathcal{D}$，定义好所有特征函数 $f_k$。
2.  初始化参数 $\theta$ (通常为全零)。
3.  进入优化循环（例如L-BFGS的每一次迭代）：
    a. 计算目标函数 $\mathcal{L}(\theta)$ 和梯度 $\nabla\mathcal{L}(\theta)$。
        i. **(对每个训练样本)**：
        ii. 运行**前向-后向算法**计算出 $Z(X)$ 和所有边的边缘概率 $p(y_{t-1}, y_t|X)$。
        iii. 根据这些结果计算出该样本贡献的**模型期望**。
        iv. **(汇总所有样本)**：将所有样本的经验期望和模型期望相加，得到总的梯度。
    b. L-BFGS算法根据当前函数值和梯度，计算出下一步的更新方向和步长。
    c. 更新参数 $\theta$。
4.  重复步骤3，直到满足收敛条件（例如梯度足够小，或函数值变化不大）。

### 4. 总结一下

本次，我们完成了CRF学习理论的最后一块拼图：
1.  **明确了学习目标**：通过最大化带正则项的对数似然函数来学习参数。
2.  **推导了核心公式**：详细推导了对数似然函数的梯度，并揭示了其“经验期望 - 模型期望”的深刻含义。
3.  **连接了理论与实践**：阐明了模型期望的计算依赖于前向-后向算法，从而将CRF的三大问题融会贯通。
4.  **介绍了优化算法**：了解了L-BFGS是当前训练CRF的主流高效算法。

至此，关于条件随机场的系列文章就全部完成了。从直觉、定义，到推断和学习，我们已经构建了一个关于CRF的完整知识体系。