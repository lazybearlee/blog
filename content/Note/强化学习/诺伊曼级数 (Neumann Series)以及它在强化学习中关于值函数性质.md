---
title: 诺伊曼级数 (Neumann Series)以及它在强化学习中关于值函数性质
date: 2025-10-11
slug: blog-post-slug
tags:
  - 强化学习
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---

## 诺伊曼级数的原理和展开

### 1. 诺伊曼级数的数学基础

诺伊曼级数是几何级数 $1 + x + x^2 + \dots = (1-x)^{-1}$ 在矩阵上的推广。

对于一个 $N \times N$ 矩阵 $\mathbf{B}$，如果其**谱半径** $\rho(\mathbf{B})$（即其特征值绝对值的最大值）满足 $\rho(\mathbf{B}) < 1$，那么矩阵 $(\mathbf{I} - \mathbf{B})$ 是可逆的，并且其逆可以写成收敛的矩阵级数：

$$
(\mathbf{I} - \mathbf{B})^{-1} = \sum_{k=0}^{\infty} \mathbf{B}^k = \mathbf{I} + \mathbf{B} + \mathbf{B}^2 + \mathbf{B}^3 + \dots
$$

### 2. 应用于贝尔曼方程

在我们的贝尔曼方程中，待求逆的矩阵是 $(\mathbf{I} - \gamma \mathbf{P}_{\pi})$。因此，我们设定 $\mathbf{B} = \gamma \mathbf{P}_{\pi}$。

我们知道 $\mathbf{P}_{\pi}$ 是一个随机矩阵。根据 Perron–Frobenius 定理，随机矩阵的最大特征值（即谱半径）是 $\rho(\mathbf{P}_{\pi}) = 1$。

因此，$\mathbf{B} = \gamma \mathbf{P}_{\pi}$ 的谱半径为：
$$\rho(\mathbf{B}) = \rho(\gamma \mathbf{P}_{\pi}) = \gamma \rho(\mathbf{P}_{\pi}) = \gamma \cdot 1 = \gamma$$

由于 $\gamma < 1$，我们有 $\rho(\mathbf{B}) < 1$，所以诺伊曼级数成立并收敛。

$$
(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} = \mathbf{I} + \gamma \mathbf{P}_{\pi} + \gamma^2 \mathbf{P}_{\pi}^2 + \gamma^3 \mathbf{P}_{\pi}^3 + \dots
$$

这个级数展开是正确的，并且是该推导的起点。

---

## 非负性 (Non-negativity) 证明

现在我们来证明结论：$(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} \ge \mathbf{I} \ge \mathbf{0}$。

这里的矩阵不等式 $\mathbf{A} \ge \mathbf{B}$ 是指**逐元素不等式**：矩阵 $\mathbf{A}$ 的每一个元素都大于或等于矩阵 $\mathbf{B}$ 的相应元素。

### 1. 证明 $(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} \ge \mathbf{0}$

$\mathbf{P}_{\pi}$ 是一个转移概率矩阵，其所有元素 $p_{\pi}(s_j|s_i)$ 都是概率，因此：
$$
\mathbf{P}_{\pi} \ge \mathbf{0}
$$

矩阵的乘积和幂：
*   两个非负矩阵相乘，结果仍然是非负矩阵。因此，$\mathbf{P}_{\pi}^k \ge \mathbf{0}$ 对所有 $k \ge 1$ 成立。
*   折扣因子 $\gamma \in [0, 1)$，所以 $\gamma^k \ge 0$。

将这些性质代入诺伊曼级数展开：
$$
(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} = \mathbf{I} + \underbrace{\gamma \mathbf{P}_{\pi}}_{\ge \mathbf{0}} + \underbrace{\gamma^2 \mathbf{P}_{\pi}^2}_{\ge \mathbf{0}} + \underbrace{\gamma^3 \mathbf{P}_{\pi}^3}_{\ge \mathbf{0}} + \dots
$$

由于级数中的所有项都是非负矩阵（或零矩阵），因此它们的和仍然是非负矩阵。

$$
(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} \ge \mathbf{0}
$$

**物理意义：** 这个逆矩阵，有时被称为**基本矩阵**或**折扣化平均时间矩阵**，其 $(i, j)$ 元素表示从状态 $s_i$ 开始，经过所有可能的路径，到达状态 $s_j$ 的折扣访问次数的期望。因为概率和折扣因子都是非负的，所以这个期望次数不可能是负的。

### 2. 证明 $(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} \ge \mathbf{I}$

我们重新审视诺伊曼级数：

$$
(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} = \mathbf{I} + \left( \gamma \mathbf{P}_{\pi} + \gamma^2 \mathbf{P}_{\pi}^2 + \gamma^3 \mathbf{P}_{\pi}^3 + \dots \right)
$$

设括号内的项为 $\mathbf{H}$：
$$
\mathbf{H} = \sum_{k=1}^{\infty} \gamma^k \mathbf{P}_{\pi}^k
$$

我们已经证明了 $\gamma^k \mathbf{P}_{\pi}^k \ge \mathbf{0}$ 对所有 $k \ge 1$ 成立，因此 $\mathbf{H}$ 是一个非负矩阵：
$$\mathbf{H} \ge \mathbf{0}$$

所以：
$$
(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1} = \mathbf{I} + \mathbf{H} \ge \mathbf{I}
$$

**结论：** 矩阵 $(\mathbf{I} - \gamma \mathbf{P}_{\pi})^{-1}$ 的每个对角线元素都大于或等于 $1$ (来自 $\mathbf{I}$ 的对角线元素 $1$ 加上 $\mathbf{H}$ 的非负对角线元素），而每个非对角线元素都大于或等于 $0$ (来自 $\mathbf{I}$ 的非对角线元素 $0$ 加上 $\mathbf{H}$ 的非负非对角线元素）。

---

## 对强化学习的意义

这个结论在 RL 中有深远的意义：

1.  **值函数的性质：** 由于 $\mathbf{v} = (\mathbf{I} - \gamma \mathbf{P})^{-1} \mathbf{r}$，如果即时奖励 $\mathbf{r}$ 是非负的（即 $\mathbf{r} \ge \mathbf{0}$），那么状态值函数 $\mathbf{v}$ 也必然是非负的。这是因为两个非负矩阵/向量相乘的结果仍是非负的。
2.  **累积效应：** 矩阵 $(\mathbf{I} - \gamma \mathbf{P})^{-1}$ 是一个**放大器**。它将即时奖励 $\mathbf{r}$ 放大成了考虑了所有未来步骤和折扣的累积回报 $\mathbf{v}$。表达式 $\mathbf{I} + \gamma \mathbf{P}_{\pi} + \gamma^2 \mathbf{P}_{\pi}^2 + \dots$ 准确地表示了这种累积效应，即从当前状态开始，一步、两步、三步……之后可能获得的折扣期望奖励的总和。