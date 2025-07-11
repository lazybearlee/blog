---
title: 主成分分析PCA-总体（一）
date: 2025-07-10
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
### **基本想法**

这一节的核心是回答两个问题：**我们为什么要做主成分分析？它究竟在做什么？**

#### **1. 解决“变量相关性”带来的分析难题**

*   **书中所述**：“数据的变量之间可能存在相关性，以致增加了分析的难度。”
*   假设一个数据集，记录了中学生的身高（厘米）、体重（公斤）和立定跳远成绩（米）。这三个变量很可能不是独立的。身高越高、体重越重的学生，可能跳得更远（也可能跳不远，但它们之间存在某种关系）。如果我们直接用这三个相关的变量去建模，信息是有冗余的。比如，身高和体重可能共同反映了一个更本质的特征，不妨称之为“身体素质”。
*   **PCA的目标**：我们希望能找到一个新的变量，比如就叫“身体素质综合得分”（$y_1$），它是一个由身高、体重、跳远成绩线性组合而成的**新变量**。这个新变量应该尽可能多地包含原来三个变量中的信息。如果一个新变量不够，我们就找第二个、第三个，但要求这些新变量之间**互相独立、没有冗余信息**。这就是书中所说的“由少数不相关的变量来代替相关的变量”。

#### **2. 坐标系旋转与最大方差**

在《机器学习方法》中，李航老师用了一个非常精彩的类比：“**主成分分析等价于进行坐标系旋转变换**”。

*   **原始数据**：假设一下二维平面上有一堆散点，代表了学生的（身高，体重）数据。这些点可能呈一个斜向上的椭圆形分布，这正体现了身高和体重的相关性。我们用原始的坐标轴（身高轴，体重轴）来描述这些点。

*   **PCA要做什么？** PCA不改变数据点本身，而是试图找到一个**新的坐标系**来更好地描述这堆数据。
    *   **寻找新坐标轴 (主成分)**：
        1.  **第一主成分 ($y_1$)**：寻找一个新的坐标轴方向，当我们把所有数据点都投影到这个新轴上时，这些投影点的**方差最大**。在我们的例子里，这个新轴的方向大致就是椭圆的长轴方向==。为什么是方差最大？因为方差衡量了数据的离散程度。方差最大的方向，就是数据变化最剧烈、包含信息最丰富的方向。==
        2.  **第二主成分 ($y_2$)**：寻找第二个坐标轴方向。这个方向必须与第一个坐标轴**正交（垂直）**，并且在所有与第一主成分正交的方向中，它同样使得数据投影后的方差最大。在我们的例子里，这基本就是椭圆的短轴方向。
        3.  以此类推，直到找到m个新的坐标轴。

*   **“信息保持”与“降维”**
    *   **信息保持**：书中提到“要求能够保留数据中的大部分信息”。在PCA的语境下，**“信息”由方差来度量**。所有新坐标轴上的方差之和，等于原始坐标轴上的方差之和。这意味着坐标旋转本身没有丢失任何信息。
    *   **降维**：但我们发现，第一个主成分（椭圆长轴）方向上的方差远大于第二个主成分（椭圆短轴）方向上的方差。这意味着，如果我们只保留第一主成分，用一个一维的“身体素质综合得分”来代替二维的（身高，体重），我们可能已经保留了原始数据95%的信息。这就是降维。

![](Note/统计学习/assets/Pasted%20image%2020250710212623.png)

PCA的本质，是通过**正交变换（坐标系旋转）**，将一组可能相关的原始变量，转化为一组**线性无关**的新变量（主成分）。这些新变量按照**方差**从大到小排列，使得我们可以用方差最大的前几个新变量来近似代表整个数据集，从而达到降维和去相关的目的。

---

### **定义和导出 (The "How", in Math)**

现在，我们把上面这些直观的想法，用精确的数学语言来描述。这就是书中16.1.2节的核心任务。

#### **1. 数学设定**

*   **随机向量 $\mathbf{x}$**:
    
    $$
    \mathbf{x} = (x_1, x_2, \dots, x_m)^T
    $$
    
    这不再是具体的数据点了，而是代表一个包含 $m$ 个随机变量的向量。比如，$\mathbf{x}$ 可以是（身高, 体重, 跳远成绩）这三个随机变量的集合。

*   **均值向量 $\mathbf{\mu}$**:
    
    $$
    \mathbf{\mu} = E(\mathbf{x}) = (\mu_1, \mu_2, \dots, \mu_m)^T
    $$
    
    这是随机向量 $\mathbf{x}$ 的期望值，包含了每个变量的平均值。

*   **协方差矩阵 $\mathbf{\Sigma}$**:
    
    $$
    \mathbf{\Sigma} = \text{cov}(\mathbf{x}, \mathbf{x}) = E[(\mathbf{x}-\mathbf{\mu})(\mathbf{x}-\mathbf{\mu})^T]
    $$
    
    这是**至关重要的概念**。这是一个 $m \times m$ 的对称矩阵。
    *   对角线元素 $\Sigma_{ii}$ 是第 $i$ 个变量自身的方差，即 $\text{var}(x_i)$。
    *   非对角线元素 $\Sigma_{ij}$ 是第 $i$ 和第 $j$ 个变量的协方差，即 $\text{cov}(x_i, x_j)$。
    *   **协方差矩阵完美地描述了原始变量间的相关性**。==我们的目标就是通过一个变换，得到一组新变量，使得它们的新协方差矩阵变成一个**对角矩阵**（即非对角线元素全为0，表示新变量间线性无关）。==

#### **2. 线性变换**

我们寻找的新变量 $y_i$ 是原始变量 $x_j$ 的线性组合。

$$
y_i = \alpha_{i1}x_1 + \alpha_{i2}x_2 + \dots + \alpha_{im}x_m
$$

写成向量形式就是：

$$
y_i = \mathbf{\alpha}_i^T \mathbf{x}
$$

*   $\mathbf{x}$: 原始的 $m$ 维随机向量。
*   $\mathbf{\alpha}_i = (\alpha_{i1}, \alpha_{i2}, \dots, \alpha_{im})^T$: 这是一个 $m$ 维的**系数向量**或**权重向量**。它定义了一个方向。PCA的核心任务，就是要**找到这一组最佳的 $\mathbf{\alpha}_i$**。
*   $y_i$: 变换后的新变量，是一个标量。它就是 $\mathbf{x}$ 在 $\mathbf{\alpha}_i$ 方向上的投影（如果 $\mathbf{\alpha}_i$ 是单位向量）。

#### **3. 新变量的统计性质**

一旦定义了变换 $y_i = \mathbf{\alpha}_i^T \mathbf{x}$，利用随机变量的性质，我们可以马上推导出新变量的期望、方差和协方差：

*   **期望 (式16.2)**:
    
    $$
    E(y_i) = E(\mathbf{\alpha}_i^T \mathbf{x}) = \mathbf{\alpha}_i^T E(\mathbf{x}) = \mathbf{\alpha}_i^T \mathbf{\mu}
    $$
    
    (因为期望是线性运算)

*   **方差 (式16.3)**:
    
    $$
    \text{var}(y_i) = \text{cov}(y_i, y_i) = \text{cov}(\mathbf{\alpha}_i^T \mathbf{x}, \mathbf{\alpha}_i^T \mathbf{x}) = \mathbf{\alpha}_i^T \text{cov}(\mathbf{x}, \mathbf{x}) \mathbf{\alpha}_i = \mathbf{\alpha}_i^T \mathbf{\Sigma} \mathbf{\alpha}_i
    $$
    
    ==这个公式是**核心中的核心**。它告诉我们，新变量的方差，完全由原始数据的协方差矩阵 $\mathbf{\Sigma}$ 和我们选择的方向 $\mathbf{\alpha}_i$ 决定。 我们的目标就是最大化这个值！==

*   **协方差 (式16.4)**:
    
    $$
    \text{cov}(y_i, y_j) = \mathbf{\alpha}_i^T \mathbf{\Sigma} \mathbf{\alpha}_j
    $$
    
    这个公式衡量了任意两个新变量 $y_i$ 和 $y_j$ 之间的相关性。==**我们的目标就是让这个值在 $i \ne j$ 时等于0！**==

#### 4. 定义16.1 (总体主成分) 

这个定义将我们前面所有的直观想法和数学推导，总结成了三条严格的规则。这三条规则合在一起，就构成了一个**带约束的序贯优化问题**。

*   **条件(1): $\mathbf{\alpha}_i^T \mathbf{\alpha}_i = 1$**
    *   **数学意义**: 系数向量 $\mathbf{\alpha}_i$ 是单位向量，它的长度为1。
    *   **为什么需要这个条件?** 如果没有这个约束，我们可以通过单纯地把 $\mathbf{\alpha}_i$ 的所有元素乘以100，来让方差 $\text{var}(y_i) = \mathbf{\alpha}_i^T \mathbf{\Sigma} \mathbf{\alpha}_i$ 变成原来的10000倍。这样最大化方差就没意义了。这个条件确保我们只关心**方向**，而不是向量的长度。

*   **条件(2): $y_i$ 与 $y_j$ 互不相关，即 $\text{cov}(y_i, y_j) = 0 \quad (i \ne j)$**
    *   **数学意义**: 根据式(16.4)，这就等价于 $\mathbf{\alpha}_i^T \mathbf{\Sigma} \mathbf{\alpha}_j = 0 \quad (i \ne j)$。
    *   **直观联系**: 这就是我们“新坐标轴要正交”这个思想在统计上的体现。注意，这里的正交是“关于 $\mathbf{\Sigma}$ 的正交”，它保证了新变量在统计意义上的线性无关。

*   **条件(3): $y_1$ 方差最大，$y_2$ 是与 $y_1$ 无关的方差最大者，以此类推。**
    *   **这是一个逐级寻找（贪心）的过程**:
        1.  **求第一主成分 $y_1$**: 我们要求解的优化问题是：
            
            $$
            \max_{\mathbf{\alpha}_1} \quad \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1 \quad \quad \text{s.t.} \quad \mathbf{\alpha}_1^T \mathbf{\alpha}_1 = 1
            $$
            
        2.  **求第二主成分 $y_2$**: 在所有与 $y_1$ 不相关的线性变换中，找方差最大的。优化问题是：
            
            $$
            \max_{\mathbf{\alpha}_2} \quad \mathbf{\alpha}_2^T \mathbf{\Sigma} \mathbf{\alpha}_2 \quad \quad \text{s.t.} \quad \mathbf{\alpha}_2^T \mathbf{\alpha}_2 = 1 \quad \text{and} \quad \text{cov}(y_1, y_2) = \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_2 = 0
            $$
            
        3.  **求第 $k$ 主成分 $y_k$**: 以此类推。

---

至此，**“问题是什么”** 已经定义得非常清楚了。接下来的内容，就是去 **“如何求解”** 这个优化问题了。而这个求解过程，将不可避免地与**协方差矩阵 $\mathbf{\Sigma}$ 的特征值分解**联系在一起。

---
### **核心：定理16.1 的证明**

> [!important] 定理 16.1
> 设 $\boldsymbol{x}$ 是 $m$ 维随机变量，$\boldsymbol{\Sigma}$ 是 $\boldsymbol{x}$ 的协方差矩阵，$\boldsymbol{\Sigma}$ 的特征值分别是 $\lambda_1 \geq \lambda_2 \geq \cdots \geq \lambda_m \geq 0$，特征值对应的单位特征向量分别是 $\boldsymbol{\alpha}_1, \boldsymbol{\alpha}_2, \cdots, \boldsymbol{\alpha}_m$，则 $\boldsymbol{x}$ 的第 $k$ 主成分是 
> 
> $$y_k = \boldsymbol{\alpha}_k^T \boldsymbol{x} = \alpha_{1k}x_1 + \alpha_{2k}x_2 + \cdots + \alpha_{mk}x_m, \quad k = 1,2,\cdots,m$$
> 
>  $\boldsymbol{x}$ 的第 $k$ 主成分的方差是 
>  
>  $$\text{var}(y_k) = \boldsymbol{\alpha}_k^T \boldsymbol{\Sigma} \boldsymbol{\alpha}_k = \lambda_k, \quad k = 1,2,\cdots,m$$ 
>  
>  即协方差矩阵 $\boldsymbol{\Sigma}$ 的第 $k$ 个特征值。

**定理内容简述**:
这个定理告诉我们一个惊人的结论：==我们费尽心思定义的、通过“序贯最大化方差”得到的主成分，其方向向量 $\mathbf{\alpha}_k$ 恰好就是协方差矩阵 $\mathbf{\Sigma}$ 的**特征向量**，而该主成分的方差 $\text{var}(y_k)$ 恰好就是对应的**特征值** $\lambda_k$。==

这个定理把一个复杂的优化问题，转化成了一个我们非常熟悉的线性代数问题——求解矩阵的特征值和特征向量。

---

### **第一步：证明第一主成分**

我们的目标是求解下面这个带约束的优化问题：

$$
\max_{\mathbf{\alpha}_1} \quad \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1 \quad \quad \text{s.t.} \quad \mathbf{\alpha}_1^T \mathbf{\alpha}_1 = 1
$$

#### **1. 构造拉格朗日函数 (Why Lagrange Multipliers?)**

*   **回顾**：拉格朗日乘子法是解决“带等式约束的优化问题”的标准工具。它的核心思想是，将约束条件乘以一个拉格朗日乘子（比如 $\lambda$），然后从目标函数中减去它，从而构造一个新的、无约束的拉格朗日函数。通过对这个新函数求导并令其为0，我们就能找到原问题的可能极值点。

*   **我们的问题**:
    *   目标函数: $f(\mathbf{\alpha}_1) = \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1$
    *   等式约束: $g(\mathbf{\alpha}_1) = \mathbf{\alpha}_1^T \mathbf{\alpha}_1 - 1 = 0$

*   **构造拉格朗日函数 $L(\mathbf{\alpha}_1, \lambda)$**:
    
    $$
    L(\mathbf{\alpha}_1, \lambda) = f(\mathbf{\alpha}_1) - \lambda g(\mathbf{\alpha}_1) = \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1 - \lambda (\mathbf{\alpha}_1^T \mathbf{\alpha}_1 - 1)
    $$

#### **2. 对 $\mathbf{\alpha}_1$ 求导并令其为0 (The Calculus)**

现在，我们将 $L(\mathbf{\alpha}_1, \lambda)$ 看作是关于变量 $\mathbf{\alpha}_1$ 的函数，并对它求梯度。这里需要用到两个矩阵求导的常用公式：
*   $\frac{\partial (\mathbf{x}^T \mathbf{A} \mathbf{x})}{\partial \mathbf{x}} = 2\mathbf{A}\mathbf{x}$ (当 $\mathbf{A}$ 是对称矩阵时，而在我们本次求解中 $\mathbf{\Sigma}$ 就是对称的)
*   $\frac{\partial (\mathbf{x}^T \mathbf{x})}{\partial \mathbf{x}} = 2\mathbf{x}$

*   **求导过程**:
    
    $$
    \begin{align*}
    \frac{\partial L}{\partial \mathbf{\alpha}_1} &= \frac{\partial}{\partial \mathbf{\alpha}_1} \left( \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1 - \lambda (\mathbf{\alpha}_1^T \mathbf{\alpha}_1 - 1) \right) \\
    &= \frac{\partial (\mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1)}{\partial \mathbf{\alpha}_1} - \lambda \frac{\partial (\mathbf{\alpha}_1^T \mathbf{\alpha}_1)}{\partial \mathbf{\alpha}_1} - \frac{\partial(-\lambda)}{\partial \mathbf{\alpha}_1} && \text{（对各项求导）} \\
    &= 2\mathbf{\Sigma}\mathbf{\alpha}_1 - \lambda (2\mathbf{\alpha}_1) - 0 && \text{（应用矩阵求导公式）} \\
    &= 2(\mathbf{\Sigma}\mathbf{\alpha}_1 - \lambda \mathbf{\alpha}_1)
    \end{align*}
    $$

*   **令导数为0**:
    
    $$
    2(\mathbf{\Sigma}\mathbf{\alpha}_1 - \lambda \mathbf{\alpha}_1) = \mathbf{0} \\
    \implies \mathbf{\Sigma}\mathbf{\alpha}_1 = \lambda \mathbf{\alpha}_1
    $$

#### **3. 解读结果**

$\mathbf{\Sigma}\mathbf{\alpha}_1 = \lambda \mathbf{\alpha}_1$ 这个方程眼熟吗？这不就是线性代数中**特征值和特征向量的定义**！

*   $\lambda$ 是协方差矩阵 $\mathbf{\Sigma}$ 的一个**特征值**。
*   $\mathbf{\alpha}_1$ 是与该特征值 $\lambda$ 对应的**特征向量**。

我们还没结束。我们要求的是目标函数 $\mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1$ 的**最大值**。我们来看看这个最大值是什么：

$$
\begin{align*}
\text{var}(y_1) &= \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_1 && \text{（目标函数）} \\
&= \mathbf{\alpha}_1^T (\mathbf{\Sigma} \mathbf{\alpha}_1) && \text{（矩阵乘法结合律）} \\
&= \mathbf{\alpha}_1^T (\lambda \mathbf{\alpha}_1) && \text{（因为 $\mathbf{\Sigma}\mathbf{\alpha}_1 = \lambda \mathbf{\alpha}_1$）} \\
&= \lambda (\mathbf{\alpha}_1^T \mathbf{\alpha}_1) && \text{（提出标量 $\lambda$）} \\
&= \lambda \cdot 1 && \text{（根据约束条件 $\mathbf{\alpha}_1^T \mathbf{\alpha}_1 = 1$）} \\
&= \lambda
\end{align*}
$$

这说明，目标函数的值（也就是第一主成分的方差）就等于特征值 $\lambda$。为了让这个方差最大化，我们显然应该选择**最大的那个特征值**，我们称之为 $\lambda_1$。

妙！第一主成分 $y_1 = \mathbf{\alpha}_1^T \mathbf{x}$ 的系数向量 $\mathbf{\alpha}_1$ 是**协方差矩阵 $\mathbf{\Sigma}$ 的最大特征值 $\lambda_1$ 所对应的单位特征向量**。第一主成分的方差就是 $\lambda_1$。

---

### **第二步：证明第二主成分**

现在的问题变得更复杂了，我们有两个约束条件：


$$
\max_{\mathbf{\alpha}_2} \quad \mathbf{\alpha}_2^T \mathbf{\Sigma} \mathbf{\alpha}_2
$$

$$
\text{s.t.} \quad
\begin{cases}
\mathbf{\alpha}_2^T \mathbf{\alpha}_2 = 1 & \text{(约束1: 单位向量)} \\
\text{cov}(y_1, y_2) = \mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_2 = 0 & \text{(约束2: 与第一主成分无关)}
\end{cases}
$$

#### **1. 简化约束2 (The Trick)**

李航老师在书中用了一个非常巧妙的简化。让我们看看约束2：$\mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_2 = 0$。

因为我们已经知道 $\mathbf{\alpha}_1$ 是 $\mathbf{\Sigma}$ 关于 $\lambda_1$ 的特征向量，所以 $\mathbf{\Sigma}\mathbf{\alpha}_1 = \lambda_1 \mathbf{\alpha}_1$。由于 $\mathbf{\Sigma}$ 是对称矩阵，它的转置等于自身 ($\mathbf{\Sigma}^T = \mathbf{\Sigma}$)。所以，我们可以对 $\mathbf{\Sigma}\mathbf{\alpha}_1 = \lambda_1 \mathbf{\alpha}_1$ 两边同时取转置：
$(\mathbf{\Sigma}\mathbf{\alpha}_1)^T = (\lambda_1 \mathbf{\alpha}_1)^T \implies \mathbf{\alpha}_1^T \mathbf{\Sigma}^T = \lambda_1 \mathbf{\alpha}_1^T \implies \mathbf{\alpha}_1^T \mathbf{\Sigma} = \lambda_1 \mathbf{\alpha}_1^T$。

现在把这个代入约束2：

$$
\mathbf{\alpha}_1^T \mathbf{\Sigma} \mathbf{\alpha}_2 = (\lambda_1 \mathbf{\alpha}_1^T) \mathbf{\alpha}_2 = \lambda_1 (\mathbf{\alpha}_1^T \mathbf{\alpha}_2) = 0
$$

由于 $\lambda_1$ 是最大特征值（通常大于0，除非数据完全没有变化），所以上式成立的唯一可能是：

$$
\mathbf{\alpha}_1^T \mathbf{\alpha}_2 = 0
$$

这意味着，**“新变量 $y_1$ 和 $y_2$ 线性无关”这个统计条件，等价于“方向向量 $\mathbf{\alpha}_1$ 和 $\mathbf{\alpha}_2$ 几何正交”这个几何条件！**

这极大地简化了问题！李航老师在书中直接使用了这个正交条件，但这是背后的推导。

#### **2. 构造新的拉格朗日函数**

现在我们的优化问题是：

$$
\max_{\mathbf{\alpha}_2} \quad \mathbf{\alpha}_2^T \mathbf{\Sigma} \mathbf{\alpha}_2 \quad \quad \text{s.t.} \quad \mathbf{\alpha}_2^T \mathbf{\alpha}_2 = 1 \quad \text{and} \quad \mathbf{\alpha}_1^T \mathbf{\alpha}_2 = 0
$$

我们有两个约束，所以需要两个拉格朗日乘子，$\lambda$ 和 $\phi$。

$$
L(\mathbf{\alpha}_2, \lambda, \phi) = \mathbf{\alpha}_2^T \mathbf{\Sigma} \mathbf{\alpha}_2 - \lambda(\mathbf{\alpha}_2^T \mathbf{\alpha}_2 - 1) - \phi(\mathbf{\alpha}_1^T \mathbf{\alpha}_2)
$$

#### **3. 对 $\mathbf{\alpha}_2$ 求导并令其为0**

$$
\frac{\partial L}{\partial \mathbf{\alpha}_2} = \mathbf{\Sigma}\mathbf{\alpha}_2 - \lambda\mathbf{\alpha}_2 - \phi\mathbf{\alpha}_1 = \mathbf{0}
$$

#### **4. 消去乘子 $\phi$ 

这个方程里有我们不想要的 $\phi$。如何消掉它？我们可以利用向量的正交性。
用 $\mathbf{\alpha}_1^T$ 左乘上式两端：

$$
\mathbf{\alpha}_1^T (\mathbf{\Sigma}\mathbf{\alpha}_2 - \lambda\mathbf{\alpha}_2 - \phi\mathbf{\alpha}_1) = \mathbf{\alpha}_1^T \mathbf{0}
$$

$$
\mathbf{\alpha}_1^T \mathbf{\Sigma}\mathbf{\alpha}_2 - \lambda\mathbf{\alpha}_1^T \mathbf{\alpha}_2 - \phi\mathbf{\alpha}_1^T \mathbf{\alpha}_1 = 0
$$

现在我们来分析这一长串式子的每一项：
*   $\mathbf{\alpha}_1^T \mathbf{\Sigma}\mathbf{\alpha}_2$: 根据我们的约束条件，这一项为 $0$。
*   $\lambda\mathbf{\alpha}_1^T \mathbf{\alpha}_2$: 根据我们推导出的简化约束，$\mathbf{\alpha}_1^T \mathbf{\alpha}_2 = 0$，所以这一项也为 $0$。
*   $\phi\mathbf{\alpha}_1^T \mathbf{\alpha}_1$: 因为 $\mathbf{\alpha}_1$ 是单位向量，所以 $\mathbf{\alpha}_1^T \mathbf{\alpha}_1 = 1$。

代入回去，我们得到：

$$
(0) - \lambda(0) - \phi(1) = 0 \implies -\phi = 0 \implies \phi = 0
$$

这真是一个漂亮的结果！这意味着第二个拉格朗日乘子 $\phi$ 必须是0。

现在把 $\phi=0$ 代回到求导后的方程 $\mathbf{\Sigma}\mathbf{\alpha}_2 - \lambda\mathbf{\alpha}_2 - \phi\mathbf{\alpha}_1 = \mathbf{0}$ 中，得到：

$$
\mathbf{\Sigma}\mathbf{\alpha}_2 - \lambda\mathbf{\alpha}_2 = \mathbf{0} \implies \mathbf{\Sigma}\mathbf{\alpha}_2 = \lambda\mathbf{\alpha}_2
$$

#### **5. 解读结果**

这和我们对第一主成分的推导结果形式完全一样！它表明，$\mathbf{\alpha}_2$ 也必须是 $\mathbf{\Sigma}$ 的一个特征向量。

但是是哪个呢？我们的目标是最大化方差 $\text{var}(y_2) = \mathbf{\alpha}_2^T \mathbf{\Sigma} \mathbf{\alpha}_2 = \lambda$。由于我们已经用掉了最大的特征值 $\lambda_1$ 给了第一主成分，并且 $\mathbf{\alpha}_2$ 必须与 $\mathbf{\alpha}_1$ 正交，所以我们只能在**剩下的特征值**中选择最大的一个。

因为对称矩阵的属于不同特征值的特征向量是天然正交的，所以我们选择的 $\mathbf{\alpha}_2$ 自动满足与 $\mathbf{\alpha}_1$ 正交的条件。因此，我们应该选择**第二大的特征值 $\lambda_2$**。

所以！第二主成分 $y_2 = \mathbf{\alpha}_2^T \mathbf{x}$ 的系数向量 $\mathbf{\alpha}_2$ 是**协方差矩阵 $\mathbf{\Sigma}$ 的第二大特征值 $\lambda_2$ 所对应的单位特征向量**。第二主成分的方差就是 $\lambda_2$。

---

### **第三步：推广到第 k 主成分**

通过数学归纳法，我们可以将这个逻辑推广下去。
假设我们已经求出了前 $k-1$ 个主成分，它们的系数向量是 $\mathbf{\alpha}_1, \dots, \mathbf{\alpha}_{k-1}$，对应 $\mathbf{\Sigma}$ 的前 $k-1$ 大的特征值 $\lambda_1, \dots, \lambda_{k-1}$。

要求第 $k$ 主成分，优化问题是：

$$
\max_{\mathbf{\alpha}_k} \quad \mathbf{\alpha}_k^T \mathbf{\Sigma} \mathbf{\alpha}_k
$$

$$
\text{s.t.} \quad \mathbf{\alpha}_k^T \mathbf{\alpha}_k = 1 \quad \text{and} \quad \mathbf{\alpha}_i^T \mathbf{\alpha}_k = 0, \quad \text{for } i=1, \dots, k-1
$$

用同样的方法构造拉格朗日函数，会发现最优解 $\mathbf{\alpha}_k$ 必须是 $\mathbf{\Sigma}$ 的第 $k$ 大的特征值 $\lambda_k$ 所对应的单位特征向量。

**至此，定理16.1 证毕。**

---

### **推论 16.1 和主要性质的解读**

定理证明之后，剩下的性质就都是这个核心结论的自然推论了。

#### **推论 16.1 与 矩阵形式**

> [!important] 推论 16.1
> 
> $m$ 维随机变量 $\boldsymbol{y} = (y_1, y_2, \cdots, y_m)^T$ 的分量依次是 $\boldsymbol{x}$ 的第一主成分到第 $m$ 主成分的充要条件是：
> 
> 1. $\boldsymbol{y} = \boldsymbol{A}^T \boldsymbol{x}$，$\boldsymbol{A}$ 为正交矩阵：
> 
> $$
> \boldsymbol{A} = \begin{bmatrix}
> \alpha_{11} & \alpha_{12} & \cdots & \alpha_{1m} \\
> \alpha_{21} & \alpha_{22} & \cdots & \alpha_{2m} \\
> \vdots & \vdots & & \vdots \\
> \alpha_{m1} & \alpha_{m2} & \cdots & \alpha_{mm}
> \end{bmatrix}
> $$
> 
> 2. $\boldsymbol{y}$ 的协方差矩阵为对角矩阵：
> 
> $$
> \text{cov}(\boldsymbol{y}) = \text{diag}(\lambda_1, \lambda_2, \cdots, \lambda_m)
> $$
> $$
> \lambda_1 \geq \lambda_2 \geq \cdots \geq \lambda_m
> $$
> 
> 其中，$\lambda_k$ 是 $\boldsymbol{\Sigma}$ 的第 $k$ 个特征值，$\boldsymbol{\alpha}_k$ 是对应的单位特征向量，$k = 1, 2, \cdots, m$。

*   **(1) $y = \mathbf{A}^T \mathbf{x}$**: 这只是把所有 $m$ 个线性变换 $y_k = \mathbf{\alpha}_k^T \mathbf{x}$ 写成了矩阵形式。
    *   $\mathbf{A}$ 是一个 $m \times m$ 的矩阵，它的**每一列**是我们的主成分方向向量 $\mathbf{\alpha}_k$。
    $$
    \mathbf{A} = [\mathbf{\alpha}_1, \mathbf{\alpha}_2, \dots, \mathbf{\alpha}_m]
    $$
    *   因为所有的 $\mathbf{\alpha}_k$ 都是单位向量且相互正交，所以 $\mathbf{A}$ 是一个**正交矩阵**。正交矩阵有一个极好的性质：$\mathbf{A}^T \mathbf{A} = \mathbf{A} \mathbf{A}^T = \mathbf{I}$ (单位矩阵)。

*   **(2) $y$ 的协方差矩阵是对角矩阵**
    我们来推导新变量向量 $\mathbf{y}$ 的协方差矩阵：
    
    $$
    \begin{align*}
    \text{cov}(\mathbf{y}) &= E[(\mathbf{y}-E[\mathbf{y}])(\mathbf{y}-E[\mathbf{y}])^T] \\
    &= E[(\mathbf{A}^T\mathbf{x}-E[\mathbf{A}^T\mathbf{x}])(\mathbf{A}^T\mathbf{x}-E[\mathbf{A}^T\mathbf{x}])^T] \\
    &= E[\mathbf{A}^T(\mathbf{x}-E[\mathbf{x}])(\mathbf{A}^T(\mathbf{x}-E[\mathbf{x}]))^T] \\
    &= E[\mathbf{A}^T(\mathbf{x}-\mathbf{\mu})(\mathbf{x}-\mathbf{\mu})^T\mathbf{A}] \\
    &= \mathbf{A}^T E[(\mathbf{x}-\mathbf{\mu})(\mathbf{x}-\mathbf{\mu})^T] \mathbf{A} && \text{（A是常数矩阵，可提出期望）} \\
    &= \mathbf{A}^T \mathbf{\Sigma} \mathbf{A}
    \end{align*}
    $$
    
    这就是新协方差矩阵的表达式。我们知道 $\mathbf{\Sigma} \mathbf{A} = \mathbf{\Sigma} [\mathbf{\alpha}_1, \dots, \mathbf{\alpha}_m] = [\mathbf{\Sigma}\mathbf{\alpha}_1, \dots, \mathbf{\Sigma}\mathbf{\alpha}_m] = [\lambda_1\mathbf{\alpha}_1, \dots, \lambda_m\mathbf{\alpha}_m] = \mathbf{A} \mathbf{\Lambda}$，其中 $\mathbf{\Lambda}$ 是一个对角线上元素为 $\lambda_1, \dots, \lambda_m$ 的对角矩阵。
    所以：
    
    $$
    \text{cov}(\mathbf{y}) = \mathbf{A}^T \mathbf{\Sigma} \mathbf{A} = \mathbf{A}^T (\mathbf{A} \mathbf{\Lambda}) = (\mathbf{A}^T \mathbf{A}) \mathbf{\Lambda} = \mathbf{I} \mathbf{\Lambda} = \mathbf{\Lambda}
    $$
    
    $$
    \mathbf{\Lambda} = \text{diag}(\lambda_1, \lambda_2, \dots, \lambda_m)
    $$
    
    这完美地证明了，**经过PCA变换后，新变量的协方差矩阵是一个对角矩阵，对角元就是原始协方差矩阵的特征值**。这在数学上证明了新变量之间是线性无关的。

#### **性质 (1) & (2): 总方差不变性**

*   **性质(1)** 就是我们刚才证明的结论：$\text{cov}(\mathbf{y}) = \mathbf{\Lambda}$。
*   **性质(2)** $\sum \lambda_i = \sum \sigma_{ii}$
    这个性质说明，**变换前后的总方差保持不变**。PCA只是将总方差进行了重新分配，将它集中到了前几个主成分上。
    *   **证明**: 利用了矩阵的迹（trace，对角线元素之和）的性质：$\text{tr}(\mathbf{AB}) = \text{tr}(\mathbf{BA})$。
        
        $$
        \begin{align*}
        \sum_{i=1}^m \text{var}(x_i) &= \sum_{i=1}^m \sigma_{ii} = \text{tr}(\mathbf{\Sigma}) && \text{（总方差是协方差矩阵的迹）} \\
        \text{而我们有 } \mathbf{\Sigma} &= \mathbf{A} \mathbf{\Lambda} \mathbf{A}^T \quad (\text{这被称为特征值分解}) \\
        \text{tr}(\mathbf{\Sigma}) &= \text{tr}(\mathbf{A} \mathbf{\Lambda} \mathbf{A}^T) \\
        &= \text{tr}(\mathbf{A}^T \mathbf{A} \mathbf{\Lambda}) && \text{（利用迹的性质 tr(BC) = tr(CB), B=A, C=ΛAᵀ）} \\
        &= \text{tr}(\mathbf{I} \mathbf{\Lambda}) && \text{（因为 A 是正交矩阵）} \\
        &= \text{tr}(\mathbf{\Lambda}) \\
        &= \sum_{i=1}^m \lambda_i = \sum_{i=1}^m \text{var}(y_i)
        \end{align*}
        $$
        
    证明完毕。

#### **性质(3), (4), (5): 因子负荷量 (Factor Loadings)**

这部分是解释主成分的物理意义。因子负荷量 $\rho(y_k, x_i)$ 衡量的是**第k个主成分 $y_k$ 与原始的某个变量 $x_i$ 之间的相关系数**。它的绝对值越大，说明 $y_k$ 主要由这个 $x_i$ 解释。

*   **推导式(16.20)**:
    *   **相关系数定义**: $\rho(A, B) = \frac{\text{cov}(A, B)}{\sqrt{\text{var}(A)\text{var}(B)}}$
    *   **代入**:
        *   $\text{var}(y_k) = \lambda_k$ (我们已证)
        *   $\text{var}(x_i) = \sigma_{ii}$ (定义)
        *   关键是算分子 $\text{cov}(y_k, x_i)$:
            *   $x_i$ 可以表示为 $\mathbf{e}_i^T \mathbf{x}$，其中 $\mathbf{e}_i$ 是一个标准基向量（第i个元素是1，其余是0）。
            *   $\text{cov}(y_k, x_i) = \text{cov}(\mathbf{\alpha}_k^T \mathbf{x}, \mathbf{e}_i^T \mathbf{x}) = \mathbf{\alpha}_k^T \mathbf{\Sigma} \mathbf{e}_i$
            *   因为 $\mathbf{\Sigma} \mathbf{\alpha}_k = \lambda_k \mathbf{\alpha}_k$, 两边取转置得到 $\mathbf{\alpha}_k^T \mathbf{\Sigma} = \lambda_k \mathbf{\alpha}_k^T$.
            *   所以 $\text{cov}(y_k, x_i) = (\lambda_k \mathbf{\alpha}_k^T) \mathbf{e}_i = \lambda_k (\mathbf{\alpha}_k^T \mathbf{e}_i) = \lambda_k \alpha_{ik}$ (其中 $\alpha_{ik}$ 是向量 $\mathbf{\alpha}_k$ 的第 $i$ 个分量)。
    *   **组合起来**:
        $$
        \rho(y_k, x_i) = \frac{\lambda_k \alpha_{ik}}{\sqrt{\lambda_k \sigma_{ii}}} = \frac{\sqrt{\lambda_k} \alpha_{ik}}{\sqrt{\sigma_{ii}}}
        $$

*   **性质(4)**：第k个主成分与所有原始变量的相关系数的平方和，等于其方差（特征值）。这是对主成分“能量”的另一种解释。
    *   **证明**:
        
        $$
        \sum_{i=1}^m \sigma_{ii} \rho^2(y_k, x_i) = \sum_{i=1}^m \sigma_{ii} \left( \frac{\sqrt{\lambda_k} \alpha_{ik}}{\sqrt{\sigma_{ii}}} \right)^2 = \sum_{i=1}^m \sigma_{ii} \frac{\lambda_k \alpha_{ik}^2}{\sigma_{ii}} = \sum_{i=1}^m \lambda_k \alpha_{ik}^2 = \lambda_k \sum_{i=1}^m \alpha_{ik}^2
        $$
        
        因为 $\mathbf{\alpha}_k = (\alpha_{1k}, \dots, \alpha_{mk})^T$ 是单位向量，所以它的各分量平方和 $\sum_i \alpha_{ik}^2 = \mathbf{\alpha}_k^T \mathbf{\alpha}_k = 1$。
        因此，上式等于 $\lambda_k \cdot 1 = \lambda_k$。得证。

*   **性质(5)**：某个原始变量与所有主成分的相关系数的平方和为1（假设该变量已标准化）。这说明所有主成分一起，可以完全解释原始变量的方差。
