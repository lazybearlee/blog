---
title: LSTM
date: 2025-08-01
slug: blog-post-slug
tags:
  - 深度学习
  - LSTM
  - 自然语言处理
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---

### 长短期记忆网络 (LSTM) 的形式化分析

#### 1. 核心组件与状态

LSTM的核心是一个精巧的门控机制，它围绕一个**细胞状态 $\mathbf{C}_t$ 进行构建。这个细胞状态作为信息传递的高速公路，其内容可以被精确地增加或移除。整个过程由三个关键的**门（Gates）** 来控制。

在时刻 $t$，LSTM单元接收三个输入：
*   当前时刻的输入向量: $\mathbf{X}_t \in \mathbb{R}^{d_x}$
*   前一时刻的隐藏状态: $\mathbf{H}_{t-1} \in \mathbb{R}^{d_h}$
*   前一时刻的细胞状态: $\mathbf{C}_{t-1} \in \mathbb{R}^{d_h}$

它将产出两个输出：
*   当前时刻的隐藏状态: $\mathbf{H}_t \in \mathbb{R}^{d_h}$
*   当前时刻的细胞状态: $\mathbf{C}_t \in \mathbb{R}^{d_h}$

其中，$d_x$ 是输入特征维度，$d_h$ 是隐藏单元维度。

#### 2. 门控机制的数学定义

这三个门——**遗忘门（Forget Gate）**、**输入门（Input Gate）** 和**输出门（Output Gate）**——都是通过一个`sigmoid`激活函数实现的。`sigmoid`函数输出值在 $(0, 1)$ 之间，这使其能够作为控制信息通过量的“阀门”：0代表完全关闭，1代表完全打开。

**2.1 遗忘门 $\mathbf{F}_t$**

遗忘门决定了应从前一时刻的细胞状态 $\mathbf{C}_{t-1}$ 中丢弃哪些信息。它将当前输入 $\mathbf{X}_t$ 和前一时刻的隐藏状态 $\mathbf{H}_{t-1}$ 作为输入。

$$
\mathbf{F}_t = \sigma(\mathbf{W}_f \cdot [\mathbf{H}_{t-1}, \mathbf{X}_t] + \mathbf{b}_f)
$$

*   $\mathbf{W}_f$: 遗忘门的权重矩阵。
*   $\mathbf{b}_f$: 遗忘门的偏置向量。
*   $[\mathbf{H}_{t-1}, \mathbf{X}_t]$: 表示将两个向量进行拼接（concatenation）。
*   $\sigma(\cdot)$: `sigmoid`激活函数。

$\mathbf{F}_t$ 是一个与 $\mathbf{C}_{t-1}$ 维度相同的向量，其每个元素的值都在 $(0, 1)$ 之间。

**2.2 输入门 $\mathbf{I}_t$ 与候选细胞状态 $\tilde{\mathbf{C}}_t$**

输入门负责决定哪些新的信息需要被存入细胞状态。这个过程分为两步：

1.  **确定要更新的值 (输入门)**:
    输入门 $\mathbf{I}_t$ 决定了我们将更新哪些维度的信息。其结构与遗忘门类似。
    
    $$
    \mathbf{I}_t = \sigma(\mathbf{W}_i \cdot [\mathbf{H}_{t-1}, \mathbf{X}_t] + \mathbf{b}_i)
    $$
    
2.  **创建候选更新向量**:
    一个`tanh`激活函数层创建一个新的候选向量 $\tilde{\mathbf{C}}_t$，其中包含了可能被加入到细胞状态的新信息。`tanh`函数的输出在 $(-1, 1)$ 之间。
    
    $$
    \tilde{\mathbf{C}}_t = \tanh(\mathbf{W}_C \cdot [\mathbf{H}_{t-1}, \mathbf{X}_t] + \mathbf{b}_C)
    $$

#### 3. 细胞状态的更新

现在，我们可以结合遗忘门和输入门来更新细胞状态，从 $\mathbf{C}_{t-1}$ 得到 $\mathbf{C}_t$。

$$
\mathbf{C}_t = \mathbf{F}_t \odot \mathbf{C}_{t-1} + \mathbf{I}_t \odot \tilde{\mathbf{C}}_t
$$

*   $\odot$: 表示逐元素乘法（Hadamard product）。

这个更新过程分为两个明确的部分：
*   $\mathbf{F}_t \odot \mathbf{C}_{t-1}$: 将旧的细胞状态 $\mathbf{C}_{t-1}$ 乘以遗忘门向量 $\mathbf{F}_t$，实现了对旧信息的选择性遗忘。
*   $\mathbf{I}_t \odot \tilde{\mathbf{C}}_t$: 将候选向量 $\tilde{\mathbf{C}}_t$ 乘以输入门向量 $\mathbf{I}_t$，实现了对新信息的选择性输入。

这两部分的加和，构成了新的细胞状态 $\mathbf{C}_t$。这种**加性**的更新机制是LSTM能够缓解梯度消失的关键，因为它为梯度提供了一条不经过多个矩阵连乘的、更直接的传播路径。

#### 4. 隐藏状态的更新

最后，我们需要确定当前时刻的输出，即隐藏状态 $\mathbf{H}_t$。这个输出是基于当前细胞状态 $\mathbf{C}_t$ 的一个过滤版本。

1.  **确定要输出的部分 (输出门)**:
    输出门 $\mathbf{O}_t$ 决定了我们将从细胞状态中输出哪些部分。
    
    $$
    \mathbf{O}_t = \sigma(\mathbf{W}_o \cdot [\mathbf{H}_{t-1}, \mathbf{X}_t] + \mathbf{b}_o)
    $$
    
2.  **生成最终隐藏状态**:
    将更新后的细胞状态 $\mathbf{C}_t$ 通过一个`tanh`函数进行压缩（将其值缩放到-1到1之间），然后乘以输出门 $\mathbf{O}_t$ 的输出。
    
    $$
    \mathbf{H}_t = \mathbf{O}_t \odot \tanh(\mathbf{C}_t)
    $$
    

$\mathbf{H}_t$ 将作为当前时刻的输出，并传递给下一时刻的LSTM单元。
