---
title: EM算法的合理性
date: 2025-07-07
slug: blog-post-slug
tags:
  - 机器学习
  - EM算法
categories:
  - 笔记
description: 描述
draft: false
state: "0"
---
### 1. EM算法的合理性分析

在之前的[初识EM算法](Note/统计学习/初识EM算法.md)中，我们了解到EM算法是用来解决含有**隐变量（Latent Variables）** 或**不完整数据（Incomplete Data）** 的极大似然估计（MLE）问题。这类问题的核心挑战在于，观测数据的对数似然函数往往形如 $\log P(Y|\theta) = \log \sum_Z P(Y, Z|\theta)$，其中求和符号在对数内部，导致函数通常是非凸且难以直接最大化。

EM算法通过一个巧妙的迭代过程来解决这一难题。它的“合理性”体现在两个方面：
1.  **导出**：为什么EM算法的E步和M步能够帮助我们逼近真实观测数据的最大似然估计？它的迭代公式是如何从最大化目标中自然产生的？
2.  **收敛性**：这个迭代过程是否一定会收敛？如果收敛，它会收敛到什么地方？


### 2. 如何导出EM算法？

EM算法的导出，其核心思想是，我们不直接最大化难以处理的观测数据对数似然 $\log P(Y|\theta)$，而是通过构造并迭代最大化它的一个**下界（Lower Bound）**。这个下界与真实似然在当前参数点处相切，并且易于最大化。

#### 2.1 目标函数

我们的最终目标是最大化观测数据 $Y$ 关于模型参数 $\theta$ 的对数似然函数 $L(\theta)$：

$$
L(\theta) = \log P(Y|\theta)
$$

我们知道，观测数据 $Y$ 的概率 $P(Y|\theta)$ 可以通过对**完整数据** $P(Y, Z|\theta)$（即观测数据 $Y$ 和隐变量 $Z$ 的联合概率）对隐变量 $Z$ 进行求和（或积分）得到：

$$
P(Y|\theta) = \sum_Z P(Y, Z|\theta)
$$

因此，我们的目标函数可以写为：

$$
L(\theta) = \log \sum_Z P(Y, Z|\theta)
$$

如前所述，由于求和在对数内部，这通常难以直接求导和最大化。

#### 2.2 引入辅助分布与构造下界

为了解决这个问题，我们引入一个**辅助的概率分布 $Q_Z(Z)$**，它代表隐变量 $Z$ 的某种分布。在EM算法中，这个 $Q_Z(Z)$ 就是我们当前对隐变量的**后验概率分布** $P(Z|Y, \theta^{(t)})$。这里的 $\theta^{(t)}$ 是第 $t$ 次迭代的参数估计值。

现在，我们对目标函数 $L(\theta)$ 进行巧妙的变形：

$$
\begin{aligned}
L(\theta) &= \log P(Y|\theta) \\
&= \log \sum_Z P(Y, Z|\theta) \\
&= \log \sum_Z \left( \frac{P(Y, Z|\theta)}{Q_Z(Z)} \right) Q_Z(Z) && \text{（在求和项内部同时乘以和除以 } Q_Z(Z) \text{）} \\
&= \log E_{Q_Z}\left[ \frac{P(Y, Z|\theta)}{Q_Z(Z)} \right] && \text{（将求和转换为关于 } Q_Z(Z) \text{的期望）}
\end{aligned}
$$

这里，我们将 $\sum_Z \text{某个函数} \cdot Q_Z(Z)$ 视为一个期望 $E_{Q_Z}[\text{某个函数}]$。

接下来，我们利用**Jensen's Inequality（琴生不等式）**。
**琴生不等式**：对于一个**凹函数** $f(x)$（例如 $\log x$），以及一个随机变量 $X$，有 $E[f(X)] \leq f(E[X])$。
在这里，我们的函数是 $\log x$（这是一个凹函数），随机变量是 $\frac{P(Y, Z|\theta)}{Q_Z(Z)}$，其期望是关于分布 $Q_Z(Z)$ 取的。
应用琴生不等式：

$$
\begin{aligned}
L(\theta) &= \log E_{Q_Z}\left[ \frac{P(Y, Z|\theta)}{Q_Z(Z)} \right] \\
&\geq E_{Q_Z}\left[ \log \left( \frac{P(Y, Z|\theta)}{Q_Z(Z)} \right) \right] && \text{（应用琴生不等式，因为 } \log \text{ 是凹函数）} \\
&= \sum_Z Q_Z(Z) \log \left( \frac{P(Y, Z|\theta)}{Q_Z(Z)} \right) && \text{（将期望展开为求和形式）} \\
&= \sum_Z Q_Z(Z) \left( \log P(Y, Z|\theta) - \log Q_Z(Z) \right) && \text{（对数性质 } \log(A/B) = \log A - \log B \text{）} \\
&= \sum_Z Q_Z(Z) \log P(Y, Z|\theta) - \sum_Z Q_Z(Z) \log Q_Z(Z) && \text{（展开）}
\end{aligned}
$$

这个推导表明，我们找到了 $L(\theta)$ 的一个下界：

$$
L(\theta) \geq \sum_Z Q_Z(Z) \log P(Y, Z|\theta) - \sum_Z Q_Z(Z) \log Q_Z(Z)
$$

我们称这个下界为 $\mathcal{L}(Q_Z, \theta)$。

#### 2.3 E步与M步的自然产生

EM算法的巧妙之处在于，它通过交替最大化这个下界 $\mathcal{L}(Q_Z, \theta)$ 来逼近 $L(\theta)$ 的最大值。

**2.3.1 E步（Expectation Step）：固定 $\theta^{(t)}$，选择最佳的 $Q_Z(Z)$**

在E步中，我们固定当前参数 $\theta^{(t)}$，并选择一个“最佳”的辅助分布 $Q_Z(Z)$，使得下界 $\mathcal{L}(Q_Z, \theta^{(t)})$ 尽可能地紧密地逼近真实的对数似然 $L(\theta^{(t)})$。
等价地，我们希望在 $\theta = \theta^{(t)}$ 时，琴生不等式中的等号成立。
**琴生不等式等号成立的条件**：当 $\frac{P(Y, Z|\theta)}{Q_Z(Z)}$ 是一个常数时，即 $P(Y, Z|\theta) \propto Q_Z(Z)$。
因此，我们选择 $Q_Z(Z)$ 为：

$$
Q_Z(Z) = P(Z|Y, \theta^{(t)}) = \frac{P(Y, Z|\theta^{(t)})}{\sum_Z P(Y, Z|\theta^{(t)})} = \frac{P(Y, Z|\theta^{(t)})}{P(Y|\theta^{(t)})}
$$

*   这个 $Q_Z(Z)$ 就是隐变量 $Z$ 在给定观测数据 $Y$ 和当前参数 $\theta^{(t)}$ 时的**后验概率分布**。这正是EM算法E步的计算内容。
*   **Q函数的定义**：当我们用 $P(Z|Y, \theta^{(t)})$ 代替 $Q_Z(Z)$ 时，下界中的第一项就是我们熟悉的**Q函数**：
    
    $$
    Q(\theta, \theta^{(t)}) = \sum_Z P(Z|Y, \theta^{(t)}) \log P(Y, Z|\theta)
    $$
    
    而下界中的第二项 $\sum_Z P(Z|Y, \theta^{(t)}) \log P(Z|Y, \theta^{(t)})$ 是一个关于 $\theta$ 的常数（它只依赖于 $\theta^{(t)}$ 和 $Y$）。

**2.3.2 M步（Maximization Step）：固定 $Q_Z(Z)$，最大化下界关于 $\theta$ 的函数**

在M步中，我们固定E步得到的 $Q_Z(Z) = P(Z|Y, \theta^{(t)})$，然后最大化下界 $\mathcal{L}(P(Z|Y, \theta^{(t)}), \theta)$ 关于 $\theta$ 的值。
由于下界中的第二项 $\sum_Z P(Z|Y, \theta^{(t)}) \log P(Z|Y, \theta^{(t)})$ 是一个常数，所以最大化整个下界等价于最大化其中的第一项，即**Q函数**：

$$
\theta^{(t+1)} = \arg\max_{\theta} Q(\theta, \theta^{(t)})
$$

M步的目标是找到新的参数 $\theta^{(t+1)}$，使得期望的完整数据对数似然（Q函数）达到最大。这一步通常比直接最大化 $L(\theta)$ 容易得多，因为Q函数中没有“对数和”的形式。

**总结导出过程**：
EM算法的E步和M步就是通过**迭代地构造并最大化观测数据对数似然的一个下界**来导出的。E步负责构造这个最紧的下界，M步负责最大化这个下界以更新参数。

### 3. 如何严谨证明EM算法的收敛性？

EM算法一个重要的性质是，它能够**保证**每一步迭代都使观测数据的对数似然函数 $L(\theta) = \log P(Y|\theta)$ 单调不减。这意味着，算法最终会收敛到局部最优解。

我们的目标是证明：

$$
L(\theta^{(t+1)}) \geq L(\theta^{(t)})
$$

**证明步骤：**

**步骤1：利用Q函数作为下界的事实**

我们已经知道，对于任意的 $\theta$ 和 $\theta^{(t)}$：

$$
L(\theta) \geq Q(\theta, \theta^{(t)}) - H_Z(\theta^{(t)})
$$

其中 $H_Z(\theta^{(t)}) = \sum_Z P(Z|Y, \theta^{(t)}) \log P(Z|Y, \theta^{(t)})$ 是一个关于 $\theta$ 的常数（实际上是隐变量 $Z$ 在给定观测数据和 $\theta^{(t)}$ 下的条件熵的负值）。
我们可以将这个不等式写成：

$$
L(\theta) - L(\theta^{(t)}) \geq Q(\theta, \theta^{(t)}) - H_Z(\theta^{(t)}) - L(\theta^{(t)})
$$

或者更简洁地，我们可以利用一个关键恒等式：

$$
L(\theta) - L(\theta^{(t)}) = Q(\theta, \theta^{(t)}) - Q(\theta^{(t)}, \theta^{(t)}) + KL(P(Z|Y, \theta^{(t)}) || P(Z|Y, \theta))
$$

其中 $KL(\cdot || \cdot)$ 是Kullback-Leibler散度，衡量两个概率分布之间的差异，且满足 $KL(P||Q) \geq 0$。KL散度是非负的，当且仅当两个分布完全相同时，KL散度为0。它常用于衡量从一个概率分布到另一个概率分布的信息损失。

**步骤2：分析M步的性质**

在M步中，我们通过最大化Q函数来选择 $\theta^{(t+1)}$：

$$
\theta^{(t+1)} = \arg\max_{\theta} Q(\theta, \theta^{(t)})
$$

这意味着，由M步得到的 $\theta^{(t+1)}$ 使得Q函数的值不小于任何其他 $\theta$ 处的值，特别是它不小于 $\theta^{(t)}$ 处的值：

$$
Q(\theta^{(t+1)}, \theta^{(t)}) \geq Q(\theta^{(t)}, \theta^{(t)})
$$

这是M步的直接结果。我们选择的 $\theta^{(t+1)}$ 就是让Q函数当前值最大的那个点。

**步骤3：利用KL散度的非负性**

我们知道KL散度总是非负的：

$$
KL(P(Z|Y, \theta^{(t)}) || P(Z|Y, \theta)) \geq 0
$$

在EM算法的迭代过程中，我们总是希望最大化观测数据的似然。如果 $P(Z|Y, \theta)$ 能够“更好地”解释数据，那么KL散度会变小。它的非负性是证明收敛的基础。

**步骤4：将所有部分结合起来证明单调性**

现在，我们把 $\theta = \theta^{(t+1)}$ 代入到关键恒等式中：

$$
L(\theta^{(t+1)}) - L(\theta^{(t)}) = Q(\theta^{(t+1)}, \theta^{(t)}) - Q(\theta^{(t)}, \theta^{(t)}) + KL(P(Z|Y, \theta^{(t)}) || P(Z|Y, \theta^{(t+1)}))
$$

我们知道：
1.  **$Q(\theta^{(t+1)}, \theta^{(t)}) - Q(\theta^{(t)}, \theta^{(t)}) \geq 0$** (来自M步的定义，因为 $\theta^{(t+1)}$ 是最大化Q函数得到的)。
2.  **$KL(P(Z|Y, \theta^{(t)}) || P(Z|Y, \theta^{(t+1)})) \geq 0$** (来自KL散度的非负性)。

因此，等式右边的两项都大于等于0。所以它们的和也大于等于0。

$$
L(\theta^{(t+1)}) - L(\theta^{(t)}) \geq 0
$$

即：

$$
L(\theta^{(t+1)}) \geq L(\theta^{(t)})
$$

这个推导严格证明了在EM算法的每次迭代中，观测数据的对数似然函数是单调不减的。也就是说，每次迭代后，模型对观测数据的解释能力（似然）都会提高或保持不变。

**收敛性结论**：
由于对数似然函数 $L(\theta)$ 是有上界（它是一个概率的对数，最大值为0）的，并且它在每次迭代中都单调不减，根据实分析的基本定理，**有上界的单调序列必然收敛**。
因此，EM算法得到的参数序列 $\theta^{(t)}$ 也会收敛到一个稳定点。

**定理 9.1 和 9.2 的含义：**

*   **定理 9.1 (单调递增性)**：
    
    $$ P(Y|\theta^{(t+1)}) \geq P(Y|\theta^{(t)}) $$
    
    这正是我们上面证明的**对数似然的单调不减性**。它保证了EM算法不会让模型变差。

*   **定理 9.2 (收敛性质)**：
    1.  **收敛到局部最优或稳定点**：如果观测数据的对数似然函数 $L(\theta)$ 有上界，则EM算法得到的参数估计序列 $\theta^{(t)}$ 对应的对数似然 $L(\theta^{(t)})$ 会收敛到某个值 $L^*$。
    2.  **收敛到稳定点**：在一些额外的正则性条件（如Q函数可微）下，EM算法得到的参数估计序列 $\theta^{(t)}$ 会收敛到 $L(\theta)$ 的一个**稳定点（Stationary Point）**，通常是局部最大值点。这并不保证收敛到全局最优，因为 $L(\theta)$ 可能是非凸的，起始值 $\theta^{(0)}$ 的选择会影响最终收敛到的局部最优。
