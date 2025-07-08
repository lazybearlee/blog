---
title: GEM算法
date: 2025-07-08
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
### 广义期望最大化（GEM）算法

### 1. 从EM到GEM：引入F函数

在之前的[初识EM算法](Note/统计学习/初识EM算法.md)中，我们证明了EM算法通过迭代最大化Q函数来保证观测数据对数似然函数 $L(\theta)$ 的单调非减性。现在，我们将引入一个更为通用的**F函数**，它将EM算法的E步和M步统一到一个单一的优化框架下。

#### 1.1 F函数的定义

**定义 9.3 (F函数)**：假设隐变量数据 $Z$ 的概率分布为 $\tilde{P}(Z)$，定义分布 $\tilde{P}$ 与参数 $\theta$ 的函数 $F(\tilde{P}, \theta)$ 如下：

$$ F(\tilde{P}, \theta) = E_{\tilde{P}}[\log P(Y, Z|\theta)] + H(\tilde{P})  $$

其中，
*   $E_{\tilde{P}}[\log P(Y, Z|\theta)]$ 是**完整数据对数似然函数**在分布 $\tilde{P}$ 下的**期望**。
    
    $$ E_{\tilde{P}}[\log P(Y, Z|\theta)] = \sum_Z \log P(Y, Z|\theta) \tilde{P}(Z) $$
    
*   $H(\tilde{P})$ 是分布 $\tilde{P}$ 的**熵 (Entropy)**。
    
    $$ H(\tilde{P}) = -E_{\tilde{P}}[\log \tilde{P}(Z)] = -\sum_Z \tilde{P}(Z) \log \tilde{P}(Z) $$

F函数可以看作是观测数据对数似然函数 $L(\theta) = \log P(Y|\theta)$ 的一个**泛函下界（Functional Lower Bound）**。它同时是隐变量的分布 $\tilde{P}$ 和模型参数 $\theta$ 的函数。EM算法可以被看作是交替最大化这个F函数的过程。

### 2. 引理分析

#### 2.1 引理9.1：寻找最优的隐变量分布 $\tilde{P}$

> [!important] **引理 9.1**：
> 对于固定的参数 $\theta$，存在唯一的分布 $\tilde{P}_\theta$ 极大化 $F(\tilde{P}, \theta)$，这时 $\tilde{P}_\theta$ 由下式给出：
> 
> $$ \tilde{P}_\theta(Z) = P(Z|Y, \theta) $$
> 

**详细推导与分析**：
这个引理回答了这样一个问题：如果我们固定了模型参数 $\theta$，我们应该选择什么样的隐变量分布 $\tilde{P}$ 才能使得F函数最大？

我们的目标是：

$$ \arg\max_{\tilde{P}} F(\tilde{P}, \theta) $$

同时，$\tilde{P}$ 必须是一个合法的概率分布，即满足约束：

$$ \sum_Z \tilde{P}(Z) = 1, \quad \tilde{P}(Z) \geq 0 $$

我们将F函数代入，并引入**拉格朗日乘子 $\lambda$** 来处理约束：

$$ \mathcal{L}(\tilde{P}, \lambda) = \sum_Z \tilde{P}(Z) \log P(Y, Z|\theta) - \sum_Z \tilde{P}(Z) \log \tilde{P}(Z) + \lambda \left(\sum_Z \tilde{P}(Z) - 1\right) $$

为了找到最优的 $\tilde{P}$，我们对 $\mathcal{L}(\tilde{P}, \lambda)$ 关于 $\tilde{P}(Z)$ 求偏导，并令其为0：

$$ \frac{\partial \mathcal{L}}{\partial \tilde{P}(Z)} = \log P(Y, Z|\theta) - (\log \tilde{P}(Z) + 1) + \lambda = 0 $$

整理得到：

$$ \log \tilde{P}(Z) = \log P(Y, Z|\theta) - 1 + \lambda $$

两边取指数：

$$ \tilde{P}(Z) = e^{\log P(Y, Z|\theta) - 1 + \lambda} = P(Y, Z|\theta) e^{\lambda - 1} $$

*   $e^{\lambda-1}$ 是一个不依赖于 $Z$ 的常数。

现在，我们利用约束 $\sum_Z \tilde{P}(Z) = 1$ 来求解这个常数：

$$ \sum_Z \tilde{P}(Z) = \sum_Z P(Y, Z|\theta) e^{\lambda-1} = 1 $$

$$ e^{\lambda-1} \sum_Z P(Y, Z|\theta) = 1 $$

由于 $\sum_Z P(Y, Z|\theta)$ 正是观测数据的边际概率 $P(Y|\theta)$，所以：

$$ e^{\lambda-1} P(Y|\theta) = 1 \implies e^{\lambda-1} = \frac{1}{P(Y|\theta)} $$

将 $e^{\lambda-1}$ 代回 $\tilde{P}(Z)$ 的表达式：

$$ \tilde{P}(Z) = \frac{P(Y, Z|\theta)}{P(Y|\theta)} $$

根据条件概率的定义，$\frac{P(Y, Z|\theta)}{P(Y|\theta)}$ 正是隐变量 $Z$ 在给定观测数据 $Y$ 和参数 $\theta$ 下的后验概率 $P(Z|Y, \theta)$。
因此，我们证明了：

$$ \tilde{P}_\theta(Z) = P(Z|Y, \theta) $$

**引理9.1的意义**：它从数学上证明了，**EM算法的E步选择的后验概率分布 $P(Z|Y, \theta^{(t)})$，正是使F函数在当前参数 $\theta^{(t)}$ 下达到最大的那个隐变量分布**。这为E步的选择提供了理论依据。

#### 2.2 引理9.2：F函数与对数似然函数的关系

> [!important] **引理 9.2**：
> 若 $\tilde{P}_\theta(Z) = P(Z|Y, \theta)$，则
> 
> $$ F(\tilde{P}_\theta, \theta) = \log P(Y|\theta) \quad (9.36) $$
> 

**详细推导与分析**：
这个引理建立了F函数与观测数据对数似然函数之间的直接联系。

**证明**：

我们将 $\tilde{P}_\theta(Z) = P(Z|Y, \theta)$ 代入F函数的定义 $F(\tilde{P}, \theta) = E_{\tilde{P}}[\log P(Y, Z|\theta)] + H(\tilde{P})$。

$$ \begin{aligned}
F(\tilde{P}_\theta, \theta) &= \sum_Z P(Z|Y, \theta) \log P(Y, Z|\theta) - \sum_Z P(Z|Y, \theta) \log P(Z|Y, \theta) \\
&= \sum_Z P(Z|Y, \theta) [\log P(Y, Z|\theta) - \log P(Z|Y, \theta)] && \text{(合并求和项)} \\
&= \sum_Z P(Z|Y, \theta) \log \left( \frac{P(Y, Z|\theta)}{P(Z|Y, \theta)} \right) && \text{(对数性质)}
\end{aligned}
$$

我们知道，根据条件概率的定义，$P(Y, Z|\theta) = P(Y|Z, \theta)P(Z|\theta)$，并且 $P(Z|Y, \theta) = \frac{P(Y|Z, \theta)P(Z|\theta)}{P(Y|\theta)}$。

因此，$\frac{P(Y, Z|\theta)}{P(Z|Y, \theta)} = P(Y|\theta)$。

将此结果代入：

$$ \begin{aligned}
F(\tilde{P}_\theta, \theta) &= \sum_Z P(Z|Y, \theta) \log P(Y|\theta) \\
&= \log P(Y|\theta) \sum_Z P(Z|Y, \theta) && \text{($\log P(Y|\theta)$ 不依赖于 $Z$)} \\
&= \log P(Y|\theta) \cdot 1 && \text{(概率分布之和为1)} \\
&= \log P(Y|\theta)
\end{aligned}
$$

**引理9.2的意义**：它表明，当隐变量的分布被选为后验概率分布时，F函数的值**恰好等于**观测数据的对数似然函数值。这进一步巩固了F函数是 $L(\theta)$ 下界的思想，并且在最优的 $\tilde{P}$ 下，这个下界与 $L(\theta)$ 相等。

### 3. 定理9.3：EM算法与F函数极大化的关系

> [!important] **定理 9.3**：
> 设 $L(\theta) = \log P(Y|\theta)$ 为观测数据的对数似然函数，$\theta^{(t)} (t=1,2,\dots)$ 为EM算法得到的参数估计序列。如果 $F(\tilde{P}, \theta)$ 在 $\tilde{P}^*$ 和 $\theta^*$ 有局部极大值，那么 $L(\theta)$ 也在 $\theta^*$ 有局部极大值。类似地，如果 $F(\tilde{P}, \theta)$ 在 $\tilde{P}^*$ 和 $\theta^*$ 达到全局最大值，那么 $L(\theta)$ 也在 $\theta^*$ 达到全局最大值。

**分析**：
这个定理建立了EM算法的收敛点与真实似然函数 $L(\theta)$ 的局部/全局最大值之间的关系。它告诉我们，通过交替最大化F函数找到的解，确实是原始优化问题的一个解。

**证明思路（反证法）**：
我们证明第一部分：如果 $(\tilde{P}^*, \theta^*)$ 是 $F$ 的局部最大值，那么 $\theta^*$ 也是 $L$ 的局部最大值。

1.  **假设** $\theta^*$ 不是 $L(\theta)$ 的局部最大值。
    这意味着在 $\theta^*$ 的邻域内，存在另一个参数 $\theta^{**}$，使得 $L(\theta^{**}) > L(\theta^*)$。
2.  根据引理9.2，我们知道 $L(\theta) = F(P(Z|Y, \theta), \theta)$。
    所以， $L(\theta^{**}) = F(P(Z|Y, \theta^{**}), \theta^{**})$ 和 $L(\theta^*) = F(P(Z|Y, \theta^*), \theta^*)$。
3.  因此，假设 $L(\theta^{**}) > L(\theta^*)$ 意味着 $F(P(Z|Y, \theta^{**}), \theta^{**}) > F(P(Z|Y, \theta^*), \theta^*)$。
4.  但是，我们已知 $(\tilde{P}^*, \theta^*)$ 是 $F(\tilde{P}, \theta)$ 的一个局部最大值。这意味着在 $(\tilde{P}^*, \theta^*)$ 的邻域内，不存在其他的 $(\tilde{P}', \theta')$ 使得 $F(\tilde{P}', \theta') > F(\tilde{P}^*, \theta^*)$。
5.  从引理9.1我们知道，当 $\theta=\theta^*$ 时，最大化 $F(\tilde{P}, \theta^*)$ 的 $\tilde{P}$ 就是 $\tilde{P}^* = P(Z|Y, \theta^*)$。
6.  因此，我们发现 $F(P(Z|Y, \theta^{**}), \theta^{**})$ 大于 $F$ 在局部最大值 $(\tilde{P}^*, \theta^*)$ 处的值。
7.  如果 $\theta^{**}$ 足够接近 $\theta^*$，那么 $P(Z|Y, \theta^{**})$ 也应该足够接近 $P(Z|Y, \theta^*)=\tilde{P}^*$（假设连续性）。这样，点 $(P(Z|Y, \theta^{**}), \theta^{**})$ 就位于 $(\tilde{P}^*, \theta^*)$ 的邻域内，这与 $(\tilde{P}^*, \theta^*)$ 是局部最大值相矛盾。

**定理9.3的意义**：它确保了EM算法最终收敛到的解是**有意义的**，即对应于原始优化问题的一个局部（或全局）最优解。这为EM算法的合理性提供了最终的保障。

### 4. EM算法的统一视角

EM算法可以被看作是**在函数 $F(\tilde{P}, \theta)$ 上的坐标上升法（Coordinate Ascent）**。

*   **F函数有两个“坐标”**：一个是分布 $\tilde{P}$，另一个是参数 $\theta$。
*   **E步**：固定参数 $\theta = \theta^{(t)}$，然后最大化 $F(\tilde{P}, \theta^{(t)})$ 关于分布 $\tilde{P}$。根据引理9.1，我们知道最优的 $\tilde{P}^{(t+1)}$ 是 $P(Z|Y, \theta^{(t)})$。
*   **M步**：固定分布 $\tilde{P} = \tilde{P}^{(t+1)}$（即 $P(Z|Y, \theta^{(t)})$），然后最大化 $F(\tilde{P}^{(t+1)}, \theta)$ 关于参数 $\theta$。
    
    $$ \begin{aligned}
    \arg\max_{\theta} F(\tilde{P}^{(t+1)}, \theta) &= \arg\max_{\theta} \left( \sum_Z \tilde{P}^{(t+1)}(Z) \log P(Y, Z|\theta) - \sum_Z \tilde{P}^{(t+1)}(Z) \log \tilde{P}^{(t+1)}(Z) \right) \\
    &= \arg\max_{\theta} \sum_Z P(Z|Y, \theta^{(t)}) \log P(Y, Z|\theta) && \text{(第二项是与$\theta$无关的常数)} \\
    &= \arg\max_{\theta} Q(\theta, \theta^{(t)})
    \end{aligned}
    $$
    
    这表明，最大化 $F$ 关于 $\theta$ 等价于最大化Q函数。

所以，EM算法的E步和M步可以被统一理解为在F函数上交替进行坐标上升，从而逐步逼近F函数的最大值，进而也逼近了 $L(\theta)$ 的最大值。

### 5. 广义EM（GEM）算法的三种变体

标准EM算法的M步要求完全最大化Q函数，这在某些情况下可能很困难或计算成本高昂。广义EM（GEM）算法放宽了这个要求，只需要在M步中**提升**Q函数的值，而不是必须找到最大值。

#### 5.1 GEM算法1（基于F函数）

**算法 9.3 GEM算法1**
这是EM算法的F函数形式，我们上面已经详细分析过。

1.  **初始化**：$\theta^{(0)}$。
2.  **第 $t+1$ 次迭代**：
    *   **步骤1 (E步)**：求 $\tilde{P}^{(t+1)}$ 极大化 $F(\tilde{P}, \theta^{(t)})$。
        结果：$\tilde{P}^{(t+1)}(Z) = P(Z|Y, \theta^{(t)})$。
    *   **步骤2 (M步)**：求 $\theta^{(t+1)}$ 极大化 $F(\tilde{P}^{(t+1)}, \theta)$。
        这等价于 $\arg\max_{\theta} Q(\theta, \theta^{(t)})$。
3.  **重复**直到收敛。

#### 5.2 GEM算法2（标准Q函数形式，放宽M步）

**算法 9.4 GEM算法2**
这是最常见的GEM形式，它直接在Q函数上操作，并放宽了M步的条件。

1.  **初始化**：$\theta^{(0)}$。
2.  **第 $t+1$ 次迭代**：
    *   **步骤1 (E步)**：计算 $Q(\theta, \theta^{(t)}) = E_Z[\log P(Y, Z|\theta)|Y, \theta^{(t)}]$。
    *   **步骤2 (M步)**：求 $\theta^{(t+1)}$ 使得
        $$ Q(\theta^{(t+1)}, \theta^{(t)}) > Q(\theta^{(t)}, \theta^{(t)}) $$
        *   **解释**：这里不要求 $\theta^{(t+1)}$ 是Q函数的**最大值**，只需要找到一个比当前值 $Q(\theta^{(t)}, \theta^{(t)})$ **更大**的值即可。
        *   **应用**：当Q函数很复杂，难以找到解析的最大值时，我们可以只运行几步梯度上升或其他数值优化方法来“提升”Q函数的值，而不是完全最大化它。这可以显著节省计算时间。例如，在支持向量机（SVM）的SMO算法中，就使用了类似的坐标上升思想，每次只优化两个变量。

#### 5.3 GEM算法3（坐标上升M步）

**算法 9.5 GEM算法3**
这是GEM算法2的一个特例，它将M步分解为对参数 $\theta = (\theta_1, \dots, \theta_d)$ 的**分量进行坐标上升**。

1.  **初始化**：$\theta^{(0)} = (\theta_1^{(0)}, \dots, \theta_d^{(0)})$。
2.  **第 $t+1$ 次迭代**：
    *   **步骤1 (E步)**：计算 $Q(\theta, \theta^{(t)})$。
    *   **步骤2 (M步)**：进行 $d$ 次条件极大化：
        *   首先，保持 $\theta_2^{(t)}, \dots, \theta_d^{(t)}$ 不变，最大化Q函数关于 $\theta_1$，得到 $\theta_1^{(t+1)}$。
        *   然后，保持 $\theta_1^{(t+1)}, \theta_3^{(t)}, \dots, \theta_d^{(t)}$ 不变，最大化Q函数关于 $\theta_2$，得到 $\theta_2^{(t+1)}$。
        *   ...如此继续，直到所有参数分量都被更新一次。
        *   最终得到 $\theta^{(t+1)} = (\theta_1^{(t+1)}, \dots, \theta_d^{(t+1)})$。
    *   **解释**：这种方法将一个高维的优化问题分解为一系列低维（甚至一维）的优化问题，通常更容易求解。在GMM的M步中，我们实际上就是这样做的：我们分别对 $\alpha_k, \mu_k, \sigma_k^2$ 进行最大化，而它们之间在Q函数中是可分离的，因此可以独立优化。
