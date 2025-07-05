---
title: 拟牛顿法-DFD部分推导
date: 2025-07-04
slug: blog-post-slug
tags:
  - 机器学习
  - 统计学习
  - 最优化
categories:
  - Blog
description: 描述
draft: false
state: "0"
---
## DFP 算法的推导 (Davidon–Fletcher–Powell Update)

### 1. 回顾与符号约定

在[拟牛顿法](Note/统计学习/拟牛顿法.md)中，我们希望用一个矩阵来**近似真实 Hessian 矩阵的逆**。我们将这个逆 Hessian 近似矩阵记为 $G_k$。
*   **$G_k$**: 当前迭代的逆 Hessian 近似矩阵（用 $G$ 表示 $H^{-1}$）。
*   **$\delta_k$** (delta_k): 从当前点 $\mathbf{x}_k$ 到下一个点 $\mathbf{x}_{k+1}$ 的**步长向量**。即 $\delta_k = \mathbf{x}_{k+1} - \mathbf{x}_k$。
*   **$y_k$**: 从当前点 $\mathbf{x}_k$ 到下一个点 $\mathbf{x}_{k+1}$ 的**梯度变化向量**。即 $y_k = \nabla f(\mathbf{x}_{k+1}) - \nabla f(\mathbf{x}_k)$。

### 2. 拟牛顿条件 (Quasi-Newton Condition)

这是所有拟牛顿方法的基础。它来源于对泰勒展开的近似，要求新的逆 Hessian 近似矩阵 $G_{k+1}$ 满足：

$$ G_{k+1} y_k = \delta_k $$

**解释**：这个条件表示，当新的近似逆 Hessian 矩阵 $G_{k+1}$ 作用在梯度变化 $y_k$ 上时，应该得到实际的步长 $\delta_k$。这是为了让 $G_{k+1}$ 更好地模拟真实逆 Hessian 的行为。

### 3. 秩2校正 (Rank-2 Correction)

DFP 算法选择通过在当前近似矩阵 $G_k$ 的基础上，**添加两个简单的校正项**（秩1矩阵）来得到 $G_{k+1}$。这种形式被称为“秩2校正”：

$$ G_{k+1} = G_k + P_k + Q_k $$

**解释**：
*   我们希望 $G_{k+1}$ 尽可能接近 $G_k$，所以不对 $G_k$ 进行大幅度修改。
*   $P_k$ 和 $Q_k$ 是为了满足拟牛顿条件而添加的“修正项”，它们通常被设计成秩为1的矩阵，以确保计算的高效性和矩阵性质（如对称性、正定性）的保持。

### 4. 确定修正项 $P_k$ 和 $Q_k$

我们的目标是让 $G_{k+1}$ 满足拟牛顿条件 $G_{k+1} y_k = \delta_k$。
将 $G_{k+1} = G_k + P_k + Q_k$ 代入条件中：

$$ (G_k + P_k + Q_k) y_k = \delta_k $$
$$ G_k y_k + P_k y_k + Q_k y_k = \delta_k $$

为了让这个等式成立，我们需要巧妙地设计 $P_k$ 和 $Q_k$。

#### 4.1 设计 $P_k$

DFP 中给出 $P_k y_k = \delta_k$，并且 $P_k = \frac{\delta_k \delta_k^T}{\delta_k^T y_k}$。
让我们验证这个 $P_k$ 是否满足条件 $P_k y_k = \delta_k$：

$$ P_k y_k = \left( \frac{\delta_k \delta_k^T}{\delta_k^T y_k} \right) y_k $$
**解释**：将 $P_k$ 的表达式代入。
*   $\delta_k^T y_k$ 是一个标量（向量点积）。
*   矩阵乘法是结合的。所以 $\delta_k^T y_k$ 可以从括号中提取出来。

$$ = \frac{\delta_k (\delta_k^T y_k)}{\delta_k^T y_k} $$
**解释**：矩阵乘法 $\delta_k \delta_k^T y_k$ 可以看作 $\delta_k (\delta_k^T y_k)$，因为 $\delta_k^T y_k$ 是一个标量。
*   由于 $\delta_k^T y_k$ 是一个非零标量（通常要求 $\delta_k^T y_k > 0$ 以保持近似矩阵的正定性），它可以被约分。

$$ = \delta_k $$
**结论**：$P_k = \frac{\delta_k \delta_k^T}{\delta_k^T y_k}$ 确实满足 $P_k y_k = \delta_k$。
**直观作用**：$P_k$ 项是为了直接贡献出我们想要的 $\delta_k$ 项。它是一个秩1矩阵。

#### 4.2 设计 $Q_k$

现在我们回到主要方程 $G_k y_k + P_k y_k + Q_k y_k = \delta_k$。
我们已经设计了 $P_k$ 使得 $P_k y_k = \delta_k$。
所以，代入后方程变为：
$G_k y_k + \delta_k + Q_k y_k = \delta_k$
为了让等式成立，我们需要：
$$ Q_k y_k = -G_k y_k $$

DFP中给出 $Q_k = -\frac{G_k y_k y_k^T G_k}{y_k^T G_k y_k}$。
让我们验证这个 $Q_k$ 是否满足条件 $Q_k y_k = -G_k y_k$:

$$ Q_k y_k = \left( -\frac{G_k y_k y_k^T G_k}{y_k^T G_k y_k} \right) y_k $$
**解释**：将 $Q_k$ 的表达式代入。
*   $y_k^T G_k y_k$ 是一个标量。
*   矩阵乘法结合律。

$$ = -\frac{G_k y_k (y_k^T G_k y_k)}{y_k^T G_k y_k} $$
**解释**：$y_k^T G_k y_k$ 是一个标量，可以被约分。

$$ = -G_k y_k $$
**结论**：$Q_k = -\frac{G_k y_k y_k^T G_k}{y_k^T G_k y_k}$ 确实满足 $Q_k y_k = -G_k y_k$。
**直观作用**：$Q_k$ 项是为了“抵消” $G_k y_k$ 这一项，从而确保整个 $G_{k+1} y_k$ 最终只剩下 $\delta_k$。它也是一个秩1矩阵。

### 5. 最终 DFP 迭代公式

将 $P_k$ 和 $Q_k$ 的表达式代回到 $G_{k+1} = G_k + P_k + Q_k$ 中，我们就得到了 DFP 算法的最终更新公式：

$$ G_{k+1} = G_k + \frac{\delta_k \delta_k^T}{\delta_k^T y_k} - \frac{G_k y_k y_k^T G_k}{y_k^T G_k y_k} $$

**解释**：
*   第一项 $G_k$ 是旧的近似矩阵。
*   第二项 $\frac{\delta_k \delta_k^T}{\delta_k^T y_k}$ 是 $P_k$ 项，它确保了更新后的矩阵将 $\delta_k$ 映射到它自己。
*   第三项 $-\frac{G_k y_k y_k^T G_k}{y_k^T G_k y_k}$ 是 $Q_k$ 项，它“减去”了 $G_k$ 作用在 $y_k$ 上的部分，以保持拟牛顿条件。

### 6. 额外重要性质

DFP 算法被设计为不仅满足拟牛顿条件，还能保持近似矩阵的**对称性**和**正定性**（如果初始矩阵 $G_0$ 是正定对称的，并且 $\delta_k^T y_k > 0$）。这对于优化算法的稳定收敛至关重要。

