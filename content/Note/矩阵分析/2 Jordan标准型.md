---
title: 2 Jordan标准型
date: 2025-12-07
slug: blog-post-slug
tags:
  - 矩阵分析
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
## $\lambda$-矩阵的代数结构与可逆性理论

这一阶段的目标是建立严谨的定义体系，特别是要厘清“$\lambda$-矩阵”与我们之前学过的“数字矩阵”在**可逆性判定**上的根本区别。这是后续所有标准形理论的基石。

### 1. $\lambda$-矩阵的严格定义

在数域 $F$ 上，我们引入变量 $\lambda$。一个矩阵，如果其所有元素均为 $F$ 上关于 $\lambda$ 的多项式，则称该矩阵为 **$\lambda$-矩阵**（或多项式矩阵）。

形式化地，设 $F[\lambda]$ 表示系数在数域 $F$ 上的多项式环，则 $m \times n$ 阶的 $\lambda$-矩阵 $A(\lambda)$ 可以表示为：

$$A(\lambda) \in F[\lambda]^{m \times n}$$

其中矩阵中的每一个元素 $a_{ij}(\lambda)$ 都是 $\lambda$ 的多项式。

为了度量 $\lambda$-矩阵的复杂度，我们需要定义其次数。$A(\lambda)$ 的次数定义为矩阵中所有元素多项式的最高次数。

例如，若

$$A(\lambda) = \begin{bmatrix} \lambda^2 + 1 & \lambda \\ 3 & \lambda^3 - 2 \end{bmatrix}$$

则 $A(\lambda)$ 的次数为 3，因为最高次项 $\lambda^3$ 出现在元素 $a_{22}$ 中。

### 2. 运算律与行列式

$\lambda$-矩阵在代数运算上继承了数字矩阵的性质。其加法、数乘、矩阵乘法以及转置运算的规则与数字矩阵完全相同，且满足结合律与分配律。

对于 $n$ 阶方阵 $A(\lambda)$，其行列式 $\det A(\lambda)$ 的计算方式也与数字矩阵一致。值得注意的是，展开后的 $\det A(\lambda)$ 本身也是 $F$ 上的一个多项式。这与数字矩阵的行列式是一个常数（标量）形成了直接对比。

### 3. 可逆性理论

这是本节最关键的概念，也是最容易产生误解的地方。

定义一个 $n$ 阶 $\lambda$-矩阵 $A(\lambda)$ 是可逆的，当且仅当存在一个同阶的 $\lambda$-矩阵 $B(\lambda)$，满足：

$$A(\lambda)B(\lambda) = B(\lambda)A(\lambda) = E$$

其中 $E$ 是 $n$ 阶单位矩阵。此时 $B(\lambda)$ 称为 $A(\lambda)$ 的逆矩阵，记为 $A^{-1}(\lambda)$ 。

定理：可逆性的判定

一个 $n$ 阶 $\lambda$-矩阵 $A(\lambda)$ 可逆的充分必要条件是：其行列式 $\det A(\lambda)$ 是一个非零常数。

推导与辨析：

在普通数字矩阵中，可逆的充要条件是 $\det A \neq 0$。但在 $\lambda$-矩阵中，仅满足 $\det A(\lambda) \neq 0$ （即行列式是非零多项式）是不够的。

原因如下：根据伴随矩阵公式 $A^{-1}(\lambda) = \frac{1}{\det A(\lambda)} A^*(\lambda)$ ，若要保证逆矩阵 $A^{-1}(\lambda)$ 的每个元素仍然是多项式（而不是分母含有 $\lambda$ 的分式），$\det A(\lambda)$ 必须能整除 $A^*(\lambda)$ 的所有元素。最强的约束来自于行列式的乘积性质：

$$\det(A(\lambda)B(\lambda)) = \det A(\lambda) \cdot \det B(\lambda) = \det E = 1$$

在多项式环中，两个多项式的乘积为常数 1，意味着这两个多项式都必须是非零常数。

### 4. 秩（Rank）的定义

$\lambda$-矩阵的秩的定义采用了“子式”的观点。如果矩阵 $A(\lambda)$ 中存在一个 $r$ 阶子式不为零，且所有 $r+1$ 阶子式（如果存在）全为零，则称 $A(\lambda)$ 的秩为 $r$，记为 $\text{rank} A(\lambda) = r$ 。

秩与可逆性的非等价性：

对于 $n$ 阶 $\lambda$-矩阵，$\text{rank} A(\lambda) = n$ 并不等价于 $A(\lambda)$ 可逆。

例如，考虑矩阵 $A(\lambda) = [\lambda]$。

- 其秩为 1（因为 $\lambda \neq 0$）。
- 其行列式为 $\lambda$（一次多项式，非非零常数）。
- 因此，它满秩但不可逆。
    
---
### 自测

问题：

给定两个 $\lambda$-矩阵：

$$M_1(\lambda) = \begin{bmatrix} 1 & \lambda \\ 0 & 1 \end{bmatrix}, \quad M_2(\lambda) = \begin{bmatrix} \lambda & 1 \\ 1 & 0 \end{bmatrix}$$

1. 请分别计算它们的行列式。
    
2. 根据判定定理，哪一个是可逆的？
    
3. 对于不可逆的那个矩阵，它的逆矩阵若存在于分式域中，形式是什么？这违背了 $\lambda$-矩阵定义的哪一条？
    

## 史密斯标准形的存在性与构造

### 1. $\lambda$-矩阵的初等变换

为了把复杂的矩阵变简单，我们使用初等变换。这与数字矩阵非常相似，但有一点关键不同：**“倍乘”和“倍加”可以使用多项式**。

定义三种初等变换：

1. **对换**：互换两行（或两列）。
2. **倍乘**：用**非零常数** $c$ 乘某一行（或列）。
    - _注意_：这里只能乘常数，**不能**乘含有 $\lambda$ 的多项式（否则行列式会变，且可能不可逆）。
3. **倍加**：将某一行（或列）的 $\varphi(\lambda)$ 倍加到另一行（或列）上。
    - _注意_：这里的 $\varphi(\lambda)$ 可以是任意多项式。
        
如果矩阵 $A(\lambda)$ 经过有限次初等变换变成 $B(\lambda)$，我们称它们**等价**，记为 $A(\lambda) \cong B(\lambda)$ 。

### 2. 史密斯标准形

任何非零的 $\lambda$-矩阵 $A(\lambda)$ 都可以通过上述初等变换，化简成一种极简的对角形式。

定理（存在性） ：

任意 $m \times n$ 阶 $\lambda$-矩阵 $A(\lambda)$ 都等价于如下形式的对角矩阵：

$$\begin{bmatrix} d_1(\lambda) & & & & \\ & d_2(\lambda) & & & \\ & & \ddots & & \\ & & & d_r(\lambda) & \\ & & & & 0_{(m-r) \times (n-r)} \end{bmatrix}$$

这个对角矩阵需满足两个强力条件：

1. **首 1 性**：$d_i(\lambda)$ 是首项系数为 1 的多项式（或 1）。
2. **整除性**：$d_1(\lambda)$ 整除 $d_2(\lambda)$， $d_2(\lambda)$ 整除 $d_3(\lambda)$，以此类推（即 $d_i(\lambda) | d_{i+1}(\lambda)$）。
    
这种形式称为 $A(\lambda)$ 的 **史密斯标准形**。$d_i(\lambda)$ 称为 **不变因子**。

---

### 3. 如何化为标准形？

这通常使用“辗转相除法”的思想。我们通过一个具体例子来演示步骤。

**例题**：将矩阵 $A(\lambda) = \begin{bmatrix} \lambda & 1 \\ 0 & \lambda \end{bmatrix}$ 化为 Smith 标准形。

**步骤演示**：

第一步：处理左上角（元素 $a_{11}$）

我们要找公因式最小的元素放到左上角。这里 $1$（即 $a_{12}$）显然比 $\lambda$ 简单。                                                                                                                                                                                                                                                                                                                                                                                                                                                              

- 操作：交换第 1 列和第 2 列。
    
    $$\begin{bmatrix} \lambda & 1 \\ 0 & \lambda \end{bmatrix} \xrightarrow{C_1 \leftrightarrow C_2} \begin{bmatrix} 1 & \lambda \\ \lambda & 0 \end{bmatrix}$$
    

**第二步：利用左上角的“1”消去第一行和第一列的其他元素**

- 操作：第 2 行减去第 1 行的 $\lambda$ 倍（$R_2 - \lambda R_1$）。
    
    $$\begin{bmatrix} 1 & \lambda \\ \lambda & 0 \end{bmatrix} \xrightarrow{R_2 - \lambda R_1} \begin{bmatrix} 1 & \lambda \\ 0 & -\lambda^2 \end{bmatrix}$$
    
- 操作：第 2 列减去第 1 列的 $\lambda$ 倍（$C_2 - \lambda C_1$）。
    
    $$\begin{bmatrix} 1 & \lambda \\ 0 & -\lambda^2 \end{bmatrix} \xrightarrow{C_2 - \lambda C_1} \begin{bmatrix} 1 & 0 \\ 0 & -\lambda^2 \end{bmatrix}$$
    

第三步：调整为首 1 多项式

现在的对角元是 $1$ 和 $-\lambda^2$。$1 | (-\lambda^2)$ 满足整除条件，但 $-\lambda^2$ 首项系数不是 1。

- 操作：第 2 行乘以 $-1$（非零常数倍乘）。
    
    $$\begin{bmatrix} 1 & 0 \\ 0 & \lambda^2 \end{bmatrix}$$
    

结果：

Smith 标准形为 $\text{diag}(1, \lambda^2)$。不变因子为 $d_1(\lambda)=1, d_2(\lambda)=\lambda^2$。

---

### 自测

请按照类似步骤，将下面这个简单的 $\lambda$-矩阵化为 Smith 标准形。

题目：

$$A(\lambda) = \begin{bmatrix} \lambda & \lambda \\ \lambda & \lambda + 1 \end{bmatrix}$$

**提示**：想办法先在左上角制造出一个“1”。

回顾原矩阵 $A(\lambda) = \begin{bmatrix} \lambda & \lambda \\ \lambda & \lambda + 1 \end{bmatrix}$。

观察矩阵结构，我们要利用元素间的差异制造常数。注意到第二行第二列的元素 $\lambda+1$ 与第一行对应元素 $\lambda$ 仅差 1，且第一列元素完全相同。

执行行变换 $R_2 - R_1$。将第一行的 $-1$ 倍加到第二行上，矩阵变为 $\begin{bmatrix} \lambda & \lambda \\ 0 & 1 \end{bmatrix}$。这一步至关重要，因为我们在矩阵中直接制造出了一个单位元素 **1**，且无需引入分式。

接下来利用这个“1”清理所在的行和列。首先，为了符合史密斯标准形 $d_1(\lambda) | d_2(\lambda)$ 的排序要求（通常“小”的元素在左上角），我们需要将这个 1 移至 $(1,1)$ 位置。执行行交换 $R_1 \leftrightarrow R_2$，矩阵变为 $\begin{bmatrix} 0 & 1 \\ \lambda & \lambda \end{bmatrix}$。接着执行列交换 $C_1 \leftrightarrow C_2$，矩阵变为 $\begin{bmatrix} 1 & 0 \\ \lambda & \lambda \end{bmatrix}$。此时，左上角已就位。

最后消去非对角线元素。执行行变换 $R_2 - \lambda R_1$，即用第一行的 $\lambda$ 倍减去第二行，矩阵变为 $\begin{bmatrix} 1 & 0 \\ 0 & \lambda \end{bmatrix}$。此时矩阵已是对角阵，且 $1$ 整除 $\lambda$，首项系数均为 1。这就是最终的 Smith 标准形。

---

## 行列式因子与标准形的唯一性

刚才我们通过繁琐的行变换求出了标准形。但在考试或复杂计算中，行变换极易出错。这就引入了本阶段的核心工具：**行列式因子**。这是一套基于不变量的理论，允许我们不通过变换，直接算出 Smith 标准形。

1. 行列式因子 $D_k(\lambda)$ 的定义

对于 $n$ 阶 $\lambda$-矩阵 $A(\lambda)$，其 $k$ 阶行列式因子 $D_k(\lambda)$ 定义为 $A(\lambda)$ 中所有 $k$ 阶子式的首项系数为 1 的最大公因式 (GCD)。

- $D_1(\lambda)$：所有元素（1 阶子式）的最大公因式。
- $D_2(\lambda)$：所有 2 阶子式（即 $2 \times 2$ 小行列式）的最大公因式。
- ...
- $D_n(\lambda)$：整个矩阵的行列式（归一化为首 1）。
    
2. 核心定理：唯一性与计算公式

定理表明，等价的矩阵拥有相同的行列式因子。这意味着原矩阵 $A(\lambda)$ 和它的 Smith 标准形 $B(\lambda) = \text{diag}(d_1, \dots, d_n)$ 拥有完全一样的 $D_k(\lambda)$。

由于标准形是对角阵，其 $k$ 阶子式非常容易计算，由此导出 不变因子 $d_k(\lambda)$ 的直接计算公式：

$$d_k(\lambda) = \frac{D_k(\lambda)}{D_{k-1}(\lambda)}, \quad (k=1, \dots, n; \text{ 规定 } D_0(\lambda)=1)$$

这不仅证明了 Smith 标准形的**唯一性** ，更为我们提供了一种无需行变换的计算路径。

3. 验证刚才的例题

让我们用这个新工具重新计算 $A(\lambda) = \begin{bmatrix} \lambda & \lambda \\ \lambda & \lambda + 1 \end{bmatrix}$ 的标准形。

首先计算 $D_1(\lambda)$。矩阵有四个元素 $\lambda, \lambda, \lambda, \lambda+1$。它们的最大公因式是什么？显然，$\lambda$ 和 $\lambda+1$ 互质（即没有公因式），所以它们的 GCD 为 1。

$$D_1(\lambda) = \text{GCD}(\lambda, \lambda, \lambda, \lambda+1) = 1$$

其次计算 $D_2(\lambda)$。对于 $2 \times 2$ 矩阵，唯一的 2 阶子式就是行列式本身。

$$\det A(\lambda) = \lambda(\lambda+1) - \lambda(\lambda) = \lambda^2 + \lambda - \lambda^2 = \lambda$$

因此 $D_2(\lambda) = \lambda$（已是首 1）。

最后利用公式计算不变因子：

- $d_1(\lambda) = D_1(\lambda) / D_0(\lambda) = 1 / 1 = 1$
- $d_2(\lambda) = D_2(\lambda) / D_1(\lambda) = \lambda / 1 = \lambda$
    
结果为 $\text{diag}(1, \lambda)$。这与我们用行变换得到的结果完全一致，但过程要快得多且不易出错。

---

## 初等因子 

如果说“不变因子”是将矩阵拆解为“层级结构”（$d_1 | d_2 | \dots$），那么“初等因子”就是将其进一步拆解为“原子结构”。这是处理矩阵相似问题（尤其是 Jordan 标准形）的关键一步。

### 1. 初等因子的定义

将 $\lambda$-矩阵 $A(\lambda)$ 的每一个不变因子 $d_i(\lambda)$ 在复数域上彻底分解，分解为一次因式幂的乘积：

$$d_i(\lambda) = (\lambda - \lambda_1)^{k_1} (\lambda - \lambda_2)^{k_2} \dots$$

**定义**：所有这些分解出来的、指数大于 0 的**一次因式幂**（例如 $(\lambda-2)^3$）的**全体集合**，称为矩阵 $A(\lambda)$ 的 **初等因子**。

**注意**：

1. **集合性**：初等因子是一个列表（Multiset），相同的因子要重复列出。
2. **排除常数**：分解出的常数因子（如 1）不计入初等因子。
    
### 2. 实例解析

假设某 $4 \times 4$ 矩阵的 Smith 标准形的主对角线元素（不变因子）为：

- $d_1(\lambda) = 1$
- $d_2(\lambda) = 1$
- $d_3(\lambda) = (\lambda - 1)(\lambda - 2)$
- $d_4(\lambda) = (\lambda - 1)^2 (\lambda - 2)$
    
**提取步骤**：

1. 忽略 $d_1, d_2$（因为它们是 1）。
2. 分解 $d_3$：得到 $(\lambda - 1)$ 和 $(\lambda - 2)$。
3. 分解 $d_4$：得到 $(\lambda - 1)^2$ 和 $(\lambda - 2)$。
    
结果：

该矩阵的初等因子为：$(\lambda - 1), (\lambda - 2), (\lambda - 1)^2, (\lambda - 2)$。

（共 4 个初等因子）。

### 3. 为什么需要初等因子？

定理：

两个 $m \times n$ 阶 $\lambda$-矩阵等价的充要条件是：它们具有相同的秩和相同的初等因子。

这个定理在处理 分块对角矩阵 时威力巨大。

**定理**：如果矩阵 $A(\lambda)$ 是准对角矩阵（由子块 $A_1, A_2, \dots$ 组成），那么 $A(\lambda)$ 的初等因子全体就是各子块初等因子的**并集**。

这意味着我们不需要把大矩阵化为 Smith 标准形，只需要分别求出小块的初等因子，然后把它们倒在一个篮子里即可。

---

### 例题

最难的考点往往是反过来的：已知初等因子，求不变因子。

这需要遵循 $d_i(\lambda) | d_{i+1}(\lambda)$ 的规则进行“组装”。

**规则**：

1. 将初等因子按“底数”分类（如所有底数为 $\lambda-1$ 的放在一组）。
2. 在每一组内，将幂次**从大到小**排列。
3. **填坑**：最大的幂次给 $d_n$，次大的给 $d_{n-1}$，依此类推。如果不够分，前面的 $d_i$ 补 1。

例题演示：

设一个 4 阶矩阵，秩为 4。其初等因子为：$(\lambda - 1)^2, (\lambda - 1), (\lambda + 2)^3$。求其不变因子。

**组装过程**：

- **第 1 步：分类与排序**
    - $\lambda - 1$ 组：$(\lambda - 1)^2, (\lambda - 1)$
    - $\lambda + 2$ 组：$(\lambda + 2)^3$
- **第 2 步：分配给 $d_4, d_3, d_2, d_1$** (从最后一个往前分，取每组最大的)
    - $d_4$（最大容器）：拿走 $\lambda-1$ 组最大的 $(\lambda-1)^2$ 和 $\lambda+2$ 组最大的 $(\lambda+2)^3$。
        $\Rightarrow d_4(\lambda) = (\lambda-1)^2 (\lambda+2)^3$
    - $d_3$（次大容器）：拿走 $\lambda-1$ 组剩下的 $(\lambda-1)$。$\lambda+2$ 组已空（视为 1）。
        $\Rightarrow d_3(\lambda) = (\lambda-1)$
    - $d_2$：两组都空了。
        $\Rightarrow d_2(\lambda) = 1$
    - $d_1$：
        $\Rightarrow d_1(\lambda) = 1$ 

结果：

不变因子为 $1, 1, \lambda-1, (\lambda-1)^2(\lambda+2)^3$。

Smith 标准形即为 $\text{diag}(1, 1, \lambda-1, (\lambda-1)^2(\lambda+2)^3)$。

---

### 自测

这是一个经典的逆向重构题。

题目：

已知一个 5 阶 $\lambda$-矩阵的秩为 5。其初等因子如下：

$$(\lambda - 3), \quad (\lambda - 3)^2, \quad (\lambda - 3), \quad (\lambda + 1)^2, \quad (\lambda + 1)$$

请推导它的 5 个不变因子 $d_1(\lambda)$ 到 $d_5(\lambda)$ 分别是什么？

（提示：先将初等因子分组，然后优先把每组幂次最高的塞给 $d_5$，次高的给 $d_4$，以此类推。）

## 数字矩阵的相似性判定准则

在本科线性代数中，判断两个矩阵是否相似（即是否存在可逆矩阵 $P$ 使得 $P^{-1}AP=B$）通常很难，往往只能通过“特征值相同”和“迹相同”来做必要条件排除，或者求特征向量。

但有了 $\lambda$-矩阵理论，我们拥有了判定相似的充要条件。

### 1. 特征矩阵

对于数域 $F$ 上的 $n$ 阶数字矩阵 $A$，我们引入变量 $\lambda$，构造其特征矩阵：

$$\lambda I - A$$

这是一个 $n$ 阶 $\lambda$-矩阵。

#### 2. 相似 $\iff$ 等价

这是本章最著名的定理，连接了“相似”与“等价”两个世界：

**定理**：两个同阶数字矩阵 $A$ 与 $B$ 相似，当且仅当它们的特征矩阵 $\lambda I - A$ 与 $\lambda I - B$ **等价**。

由于我们已经知道判定 $\lambda$-矩阵等价的工具是“不变因子”和“初等因子”，因此该定理迅速衍生出两个可操作的推论：

- **推论 1（不变因子法）**：$A \sim B$ $\iff$ 它们的特征矩阵拥有完全相同的**不变因子** 。
- **推论 2（初等因子法）**：$A \sim B$ $\iff$ 它们的特征矩阵拥有完全相同的**初等因子** 。
    
这意味着，初等因子就是矩阵相似变换下的“指纹”。如果指纹不同，矩阵绝不相似。

---

### 3. 如何判定相似？

案例：

给定两个 2 阶矩阵：

$$A = \begin{bmatrix} 1 & 1 \\ 0 & 1 \end{bmatrix}, \quad B = \begin{bmatrix} 1 & 0 \\ 0 & 1 \end{bmatrix}$$

判断它们是否相似？

解题思路：

我们不能只看特征值（它们特征值都是 1，迹都是 2，行列式都是 1，看起来很像）。我们需要计算它们的特征矩阵的指纹。

步骤 1：分析矩阵 $A$

特征矩阵为 $\lambda I - A = \begin{bmatrix} \lambda-1 & -1 \\ 0 & \lambda-1 \end{bmatrix}$。

- $D_1(\lambda)$（所有元素 GCD）：包含 $-1$（常数），所以 $D_1 = 1$。
- $D_2(\lambda)$（行列式）：$(\lambda-1)^2$。
    
- **不变因子**：
    - $d_1 = D_1/1 = 1$
    - $d_2 = D_2/D_1 = (\lambda-1)^2$
- **初等因子**：$(\lambda-1)^2$（因为 1 忽略）。

步骤 2：分析矩阵 $B$

特征矩阵为 $\lambda I - B = \begin{bmatrix} \lambda-1 & 0 \\ 0 & \lambda-1 \end{bmatrix}$。

这已经是对角阵了！

- **不变因子**：直接读出对角线元素 $\lambda-1, \lambda-1$。
    - 注意：不变因子要求整除。这里 $\lambda-1 \mid \lambda-1$，符合条件。
    - 所以 $d_1 = \lambda-1, d_2 = \lambda-1$。
- **初等因子**：$\lambda-1, \lambda-1$。
    

**步骤 3：比对**

- $A$ 的初等因子集合：$\{ (\lambda-1)^2 \}$
- $B$ 的初等因子集合：$\{ \lambda-1, \lambda-1 \}$

结论：

因为初等因子集合不同（$\{ (\lambda-1)^2 \} \neq \{ \lambda-1, \lambda-1 \}$），所以 $A$ 与 $B$ 不相似。

---

### 自测

请运用上述逻辑解决这个问题：

题目：

已知两个 3 阶复数矩阵 $A$ 和 $B$。

- 矩阵 $A$ 的特征矩阵 $\lambda I - A$ 的**不变因子**为：$1, 1, (\lambda-2)^3$。
- 矩阵 $B$ 的特征矩阵 $\lambda I - B$ 的**初等因子**为：$\lambda-2, (\lambda-2)^2$。

请问：矩阵 $A$ 和 $B$ 是否相似？

## Jordan 标准形的代数构造

这一阶段非常直观。如果说求初等因子是“拆解”过程，那么写出 Jordan 标准形就是“组装”过程。我们将把抽象的多项式（初等因子）翻译成具体的矩阵块。

### 1. Jordan 块 (Jordan Block)

Jordan 标准形是由一个个独立的子矩阵拼成的对角块矩阵。这些子矩阵被称为 **Jordan 块**。

定义：

对于复数 $\lambda_0$ 和正整数 $k$，一个 $k$ 阶 Jordan 块 $J_k(\lambda_0)$ 定义为如下形式的矩阵：

$$J_k(\lambda_0) = \begin{bmatrix} \lambda_0 & 1 & 0 & \cdots & 0 \\ 0 & \lambda_0 & 1 & \cdots & 0 \\ \vdots & \vdots & \ddots & \ddots & \vdots \\ 0 & 0 & \cdots & \lambda_0 & 1 \\ 0 & 0 & \cdots & 0 & \lambda_0 \end{bmatrix}_{k \times k}$$

**特征**：

- **主对角线**：全部是特征值 $\lambda_0$。
- **次对角线（上方）**：全部是 **1**。
- **其余位置**：全部是 0。
    
**特例**：

- 1 阶 Jordan 块：$[\lambda_0]$。
- 2 阶 Jordan 块：$\begin{bmatrix} \lambda_0 & 1 \\ 0 & \lambda_0 \end{bmatrix}$。

---

### 2. 从初等因子到 Jordan 标准形

这是本章最重要的构造定理，它告诉我们初等因子与 Jordan 块是一一对应的关系。

定理 2：

若 $n$ 阶矩阵 $A$ 的全部初等因子为：

$$(\lambda - \lambda_1)^{n_1}, \quad (\lambda - \lambda_2)^{n_2}, \quad \dots, \quad (\lambda - \lambda_s)^{n_s}$$

（注意：这里 $\lambda_i$ 可能相同，必须列出所有重复项）

则 $A$ 的 Jordan 标准形 $J$ 是一个准对角矩阵，由对应上述初等因子的 Jordan 块组成：

$$J = \text{diag}(J_{n_1}(\lambda_1), \ J_{n_2}(\lambda_2), \ \dots, \ J_{n_s}(\lambda_s))$$

**简单口诀**：

- **每一个**初等因子对应**一个** Jordan 块。
- 因子的**底数** $\lambda_i$ 决定块的**对角线元素**。
- 因子的**指数** $n_i$ 决定块的**阶数 (Size)**。

唯一性：

如果不考虑 Jordan 块在对角线上的排列顺序，矩阵 $A$ 的 Jordan 标准形是唯一的。

---

### 3. 实例

让我们通过一个具体例子来演练这种转换。

**已知**：某 5 阶矩阵 $A$ 的初等因子如下：

1. $(\lambda - 2)^3$
2. $(\lambda - 2)$
3. $(\lambda + 5)$

**分析构造**：
- **第 1 个块**：对应 $(\lambda - 2)^3$。
    - 特征值 $\lambda = 2$，阶数 $3$。
    - 块结构：$\begin{bmatrix} 2 & 1 & 0 \\ 0 & 2 & 1 \\ 0 & 0 & 2 \end{bmatrix}$
- **第 2 个块**：对应 $(\lambda - 2)$。
    - 特征值 $\lambda = 2$，阶数 $1$。
    - 块结构：$[2]$
- **第 3 个块**：对应 $(\lambda + 5)$。
    - 特征值 $\lambda = -5$，阶数 $1$。
    - 块结构：$[-5]$    

最终结果 $J$：

将它们拼成对角块（排列顺序随意，通常按特征值归类）：

$$J = \begin{bmatrix} \mathbf{2} & \mathbf{1} & \mathbf{0} & 0 & 0 \\ \mathbf{0} & \mathbf{2} & \mathbf{1} & 0 & 0 \\ \mathbf{0} & \mathbf{0} & \mathbf{2} & 0 & 0 \\ 0 & 0 & 0 & \mathbf{2} & 0 \\ 0 & 0 & 0 & 0 & \mathbf{-5} \end{bmatrix}$$

(注：粗体数字代表非零元素区域，其余为0)

---

### 4. 对角化判定

利用这一定理，我们可以瞬间判定矩阵是否可以对角化（即是否相似于纯对角矩阵）。

推论 4：

$n$ 阶矩阵 $A$ 可对角化的充分必要条件是：$A$ 的所有初等因子都是一次因式（即形式均为 $(\lambda - \lambda_i)^1$）。

- **理解**：如果所有初等因子的指数都是 1，那么所有对应的 Jordan 块都是 $1 \times 1$ 的。$1 \times 1$ 的块拼起来，自然就是普通的对角矩阵。
- **反之**：只要有一个初等因子的指数 $\ge 2$，Jordan 标准形中就会出现次对角线上的“1”，矩阵就无法对角化。
    

---

### 自测

请根据以下描述写出对应的 Jordan 标准形。

题目：

已知一个 4 阶矩阵 $A$，其特征矩阵 $\lambda I - A$ 的不变因子为：

$$d_1(\lambda) = 1, \quad d_2(\lambda) = 1, \quad d_3(\lambda) = \lambda - 3, \quad d_4(\lambda) = (\lambda - 3)(\lambda + 1)^2$$

**任务**：

1. 先写出 $A$ 的**初等因子**。
2. 写出 $A$ 的 Jordan 标准形矩阵 $J$（请直接写出矩阵形式）。

请尝试解答。


## 几何性质与秩阶梯算法


想象一下，如果给你一个 $10 \times 10$ 的数字矩阵，让你求 Jordan 标准形。

- **方法 A（$\lambda$-矩阵法）**：计算 $\lambda I - A$，求 10 阶行列式因子...（计算量是天文数字，完全不可行）。
- **方法 B（秩阶梯算法）**：通过计算几个数字矩阵的秩，直接“数”出 Jordan 块的个数和大小。这是解决高阶矩阵问题的唯一实战路径。
    
### 1. 代数重数 vs 几何重数

对于特征值 $\lambda_i$，我们有两个极其重要的指标：

1. **代数重复度**：
    - 它是特征多项式 $\det(\lambda I - A)$ 中 $(\lambda - \lambda_i)$ 的幂次。
    - **几何意义**：所有属于 $\lambda_i$ 的 Jordan 块的 **阶数之和**（也就是总面积）。
2. **几何重复度**：
    - 它是特征值 $\lambda_i$ 对应的线性无关特征向量的个数，计算公式为 $n - \text{rank}(A - \lambda_i I)$。
    - **几何意义**：属于 $\lambda_i$ 的 Jordan 块的 **个数**（也就是块的数量）。
    
**例**：若特征值 $\lambda=2$ 的代数重数为 4，几何重数为 2。
- 这意味着：关于 2 的所有块加起来占 4 行 4 列。
- 但这 4 行分成了 **2 个块**。
- 可能的组合：$3+1$ 或 $2+2$。仅凭几何重数还不够，我们需要更精细的工具。
    
### 2. 秩阶梯算法 (Weyr Characteristic)

为了区分是 "$3+1$" 还是 "$2+2$"，我们观察矩阵幂次的秩的下降速度。

对于特定的特征值 $\lambda_i$，定义 $r_k$ 为矩阵幂次的秩：

$$r_k = \text{rank}((A - \lambda_i I)^k), \quad r_0 = n$$

核心公式（块数公式） ：

对应于特征值 $\lambda_i$ 的 $k$ 阶 Jordan 块 的个数 $N_k$ 为：

$$N_k = r_{k-1} - 2r_k + r_{k+1}$$

这个公式看起来很吓人，但我们可以用一个直观的 **"二次差分表格"** 来操作：

|**幂次 k**|**秩 rk​**|**一阶差 Δk​=rk−1​−rk​**|**二阶差 (结果 Nk​) Δk​−Δk+1​**|
|---|---|---|---|
|0|$n$|-|-|
|1|$r_1$|$\Delta_1$ (总块数)|**$N_1$ (1阶块个数)**|
|2|$r_2$|$\Delta_2$|**$N_2$ (2阶块个数)**|
|3|$r_3$|$\Delta_3$|**$N_3$ (3阶块个数)**|

**规律**：

- **一阶差** $\Delta_k$ 代表：阶数 $\ge k$ 的 Jordan 块的总数。
- **二阶差**（上一层减下一层）代表：阶数 **恰好等于 $k$** 的 Jordan 块个数。

---

### 3. 例题



---

### 你的实战挑战

题目：

已知 7 阶矩阵 $A$，特征值 $\lambda=2$ 是唯一的特征值（代数重数 7）。

经计算，矩阵 $B = A - 2I$ 的幂次秩如下：

- $r_0 = 7$
    
- $r_1 = \text{rank}(B) = 4$
    
- $r_2 = \text{rank}(B^2) = 2$
    
- $r_3 = \text{rank}(B^3) = 1$
    
- $r_4 = \text{rank}(B^4) = 1$ (稳定)
    

**请按照上述“差分法”分析：**

1. 共有几个 Jordan 块？
    
2. 1 阶块、2 阶块、3 阶块各有多少个？
    
3. 写出最终的 Jordan 标准形。