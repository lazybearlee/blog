---
title: Xavier 初始化
date: 2025-07-17
slug: blog-post-slug
tags:
  - 深度学习
  - 权重初始化
categories:
  - Blog
description: 描述
draft: false
state: "0"
---
### “糟糕”的初始化会导致什么

在 Xavier 初始化被提出之前，人们通常使用非常简单的方法，比如从一个很小的正态分布（如 `N(0, 0.01)`）中采样来初始化权重。这在浅层网络中或许可行，但在深度网络中会引发灾难性的**梯度消失（Vanishing Gradients）** 或**梯度爆炸（Exploding Gradients）** 问题。

让我们直观地理解一下：
*   **信号的前向传播**: 每一层的输出（激活值）是下一层的输入。如果权重过小，每次通过一层，信号的“信息量”（用方差来衡量）就会衰减。经过很多层后，信号可能变得微乎其微，导致网络学不到东西。
*   **梯度的反向传播**: 梯度从后向前传播。如果权重过小，梯度每传播一层也会衰减。传到浅层网络时，梯度几乎为零，导致浅层参数无法更新。这就是**梯度消失**。
*   **反之亦然**: 如果权重过大，信号和梯度在逐层传播中会指数级增长，最终导致数值溢出，模型崩溃。这就是**梯度爆炸**。

**初始化的核心目标**：找到一种初始化权重的方法，使得信息（无论是前向的激活值还是反向的梯度）在网络中传播时，其**统计特性（主要是均值和方差）能够保持稳定**。我们希望每一层的输出方差和输入方差大致相同，每一层梯度的方差也和后一层梯度的方差大致相同。

### Xavier初始化的数学推导

Xavier初始化的全部推导，都围绕着这个核心目标：**保持输入和输出的方差一致**。

#### 1. 建立模型和假设

我们来看一个没有激活函数的线性层：

$$
o_i = \sum_{j=1}^{n_{in}} w_{ij}x_j
$$

*   $x_j$: 层的输入，有 $n_{in}$ 个。
*   $w_{ij}$: 权重。
*   $o_i$: 层的输出之一。

现在，我们做出几个关键且合理的**假设**：
*   **H1**: 权重 $w_{ij}$ 和输入 $x_j$ 是**相互独立**的。
*   **H2**: 每一个权重 $w_{ij}$ 都从**同一个分布**中独立采样。这个分布的**均值为0**，**方差为 $\sigma^2$**。即 $E[w_{ij}]=0$，$Var(w_{ij})=\sigma^2$。
*   **H3**: 每一个输入 $x_j$ 也来自同一个分布，其**均值为0**，**方差为 $\gamma^2$**。即 $E[x_j]=0$，$Var(x_j)=\gamma^2$。
    *   (为什么假设均值为0？这在实践中很常见，比如数据经过标准化处理，或前一层是 `tanh` 等零中心激活函数)。

#### 2. 推导前向传播的方差

我们的目标是计算输出 $o_i$ 的方差 $Var(o_i)$，并让它等于输入的方差 $Var(x_j) = \gamma^2$。

**第一步：计算 $o_i$ 的均值 $E[o_i]$**

$$
E[o_i] = E\left[\sum_{j=1}^{n_{in}} w_{ij}x_j\right]
$$

根据期望的线性性质，可以把求和与期望交换：

$$
E[o_i] = \sum_{j=1}^{n_{in}} E[w_{ij}x_j]
$$

因为 $w_{ij}$ 和 $x_j$ 相互独立（H1），所以 $E[w_{ij}x_j] = E[w_{ij}]E[x_j]$。

$$
E[o_i] = \sum_{j=1}^{n_{in}} E[w_{ij}]E[x_j]
$$

根据假设 H2 和 H3，我们知道 $E[w_{ij}]=0$ 且 $E[x_j]=0$。

$$
E[o_i] = \sum_{j=1}^{n_{in}} 0 \cdot 0 = 0
$$

**结论1**: 如果输入和权重的均值都为0，那么输出的均值也为0。

**第二步：计算 $o_i$ 的方差 $Var(o_i)$**
这是推导的核心。我们需要用到方差的两个关键性质：
1.  对于独立变量 $X, Y$，有 $Var(X+Y) = Var(X)+Var(Y)$。
2.  $Var(X) = E[X^2] - (E[X])^2$。

开始计算：

$$
Var(o_i) = Var\left(\sum_{j=1}^{n_{in}} w_{ij}x_j\right)
$$

由于各个 $w_{ij}x_j$ 项之间是相互独立的（因为 $w$ 和 $x$ 都是独立采样的），我们可以将方差的求和变为求和的方差：

$$
Var(o_i) = \sum_{j=1}^{n_{in}} Var(w_{ij}x_j)
$$

现在，我们需要计算 $Var(w_{ij}x_j)$。利用性质2：

$$
Var(w_{ij}x_j) = E[(w_{ij}x_j)^2] - (E[w_{ij}x_j])^2 = E[w_{ij}^2 x_j^2] - (E[w_{ij}]E[x_j])^2
$$

因为 $w_{ij}$ 和 $x_j$ 独立，所以 $E[w_{ij}^2 x_j^2] = E[w_{ij}^2]E[x_j^2]$。

$$
Var(w_{ij}x_j) = E[w_{ij}^2]E[x_j^2] - (0 \cdot 0)^2 = E[w_{ij}^2]E[x_j^2]
$$

我们知道 $Var(X) = E[X^2] - (E[X])^2$，当 $E[X]=0$ 时，$Var(X)=E[X^2]$。所以：
*   $E[w_{ij}^2] = Var(w_{ij}) = \sigma^2$ (来自 H2)
*   $E[x_j^2] = Var(x_j) = \gamma^2$ (来自 H3)

代入上式，得到：

$$
Var(w_{ij}x_j) = \sigma^2 \gamma^2
$$

最后，我们把它代回到 $Var(o_i)$ 的求和式中：

$$
Var(o_i) = \sum_{j=1}^{n_{in}} \sigma^2 \gamma^2 = n_{in} \cdot \sigma^2 \gamma^2 = n_{in} \cdot Var(w_{ij}) \cdot Var(x_j)
$$

**结论2**: 输出的方差是输入方差的 $n_{in} \cdot Var(w_{ij})$ 倍。

#### 3. 得到前向传播的条件

我们的目标是保持方差不变，即 $Var(o_i) = Var(x_j)$。

$$
\gamma^2 = n_{in} \cdot Var(w_{ij}) \cdot \gamma^2
$$

两边消去 $\gamma^2$，得到前向传播的稳定条件：

$$
1 = n_{in} \cdot Var(w_{ij}) \quad \implies \quad Var(w_{ij}) = \frac{1}{n_{in}}
$$

#### 4. 推导反向传播的方差

梯度也是信号，我们也希望它在反向传播时方差保持稳定。假设该层的输出是 $n_{out}$ 维。反向传播时，层的输入梯度 $\frac{\partial L}{\partial x_j}$ 是由输出梯度 $\frac{\partial L}{\partial o_i}$ 计算得来的：

$$
\frac{\partial L}{\partial x_j} = \sum_{i=1}^{n_{out}} \frac{\partial L}{\partial o_i} \cdot \frac{\partial o_i}{\partial x_j} = \sum_{i=1}^{n_{out}} \frac{\partial L}{\partial o_i} \cdot w_{ij}
$$

这个公式的结构和前向传播惊人地相似！我们可以用完全相同的逻辑进行推导，只是角色互换：
*   输入变成了 $n_{out}$ 个输出梯度 $\frac{\partial L}{\partial o_i}$。
*   输出是输入梯度 $\frac{\partial L}{\partial x_j}$。
*   权重仍然是 $w_{ij}$。

直接套用我们刚刚得到的**结论2**的结构：

$$
Var\left(\frac{\partial L}{\partial x_j}\right) = n_{out} \cdot Var(w_{ij}) \cdot Var\left(\frac{\partial L}{\partial o_i}\right)
$$

为了让反向传播的梯度方差也保持稳定，即 $Var(\frac{\partial L}{\partial x_j}) = Var(\frac{\partial L}{\partial o_i})$，我们得到反向传播的稳定条件：

$$
1 = n_{out} \cdot Var(w_{ij}) \quad \implies \quad Var(w_{ij}) = \frac{1}{n_{out}}
$$

#### 5. 正向传播与反向传播稳定的最终妥协：Xavier 初始化

现在我们面临一个困境：
*   为了前向传播稳定，需要 $Var(w_{ij}) = \frac{1}{n_{in}}$。
*   为了反向传播稳定，需要 $Var(w_{ij}) = \frac{1}{n_{out}}$。

当 $n_{in} \ne n_{out}$ 时，这两个条件是矛盾的。Xavier 和 Bengio 提出，一个合理的**妥协**是使用这两者的**调和平均数**。

$$
Var(w_{ij}) = \frac{2}{\frac{1}{1/n_{in}} + \frac{1}{1/n_{out}}} = \frac{2}{n_{in} + n_{out}}
$$

**这就是 Xavier 初始化的核心思想和最终公式！** 它试图同时兼顾前向和反向传播的稳定性。

### 实践与局限

#### 如何使用这个方差？
我们已经知道了权重的理想方差，但如何生成符合这个方差的随机数呢？通常有两种方法：

1.  **Xavier 正态分布初始化 (Xavier Normal)**:
    直接从一个均值为0，方差为 $Var(w_{ij})$ 的正态分布中采样。
    
    $$
    w_{ij} \sim \mathcal{N}\left(0, \sigma^2=\frac{2}{n_{in} + n_{out}}\right)
    $$
    

2.  **Xavier 均匀分布初始化 (Xavier Uniform)**:
    从一个均匀分布 $\mathcal{U}[-a, a]$ 中采样。均匀分布的方差是 $\frac{(a - (-a))^2}{12} = \frac{(2a)^2}{12} = \frac{a^2}{3}$。
    我们令这个方差等于我们想要的方差：
    
    $$
    \frac{a^2}{3} = \frac{2}{n_{in} + n_{out}} \quad \implies \quad a = \sqrt{\frac{6}{n_{in} + n_{out}}}
    $$
    
    所以，我们从均匀分布 $\mathcal{U}\left[-\sqrt{\frac{6}{n_{in} + n_{out}}}, \sqrt{\frac{6}{n_{in} + n_{out}}}\right]$ 中采样。
    这是 PyTorch 中 `nn.init.xavier_uniform_` 的默认实现。

#### Xavier 的局限性
我们整个推导是基于一个**线性**单元的，没有考虑激活函数。Xavier 的原论文分析了 `tanh` 和 `sigmoid` 这类“类线性”的激活函数（它们在0附近近似于线性函数 $f(x) \approx x$），并发现这个初始化效果很好。

但是，对于现代神经网络中最常用的 **ReLU** 激活函数，Xavier 初始化就不再完美了。因为 ReLU 会将所有负数输入强制变为0，这会改变输出的分布，使其方差大约减半。为了解决这个问题，Kaiming He 等人提出了**Kaiming (He) 初始化**，它专门针对 ReLU 及其变体设计，其推导过程与 Xavier 类似，但考虑了 ReLU 的影响，最终得到 $Var(w_{ij}) = \frac{2}{n_{in}}$。
