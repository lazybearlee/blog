---
title: "初等因子、不变因子、行列式因子、Smith标准型"
date: "2025-12-13"
slug: blog-post-slug
tags:
  - 标签
categories:
  - 分类
description: 描述
draft: true
state: "0"
---
### **核心概念的逻辑转化体系**

这部分知识点的难点在于概念之间的相互推导。你需要建立如下的“金字塔”结构：

1. 行列式因子 $D_k(\lambda)$：
    这是计算的起点。$D_k(\lambda)$ 是矩阵 $A(\lambda)$ 中所有 $k$ 阶子式的首项系数为1的最大公约式。
    - **性质**：$D_k(\lambda) \mid D_{k+1}(\lambda)$。
    - **常考点**：通常只计算 $D_{n}$（即特征多项式）和 $D_{n-1}$，低阶的通常是1。
2. 不变因子 $d_k(\lambda)$（连接Smith标准型的桥梁）：
    这是Smith标准型对角线上的元素。
    - **公式**：$d_k(\lambda) = \frac{D_k(\lambda)}{D_{k-1}(\lambda)}$ （规定 $D_0=1$）。
    - **Smith标准型**：$S(\lambda) = \text{diag}(d_1(\lambda), d_2(\lambda), \dots, d_r(\lambda), 0, \dots, 0)$。
    - **性质**：$d_1(\lambda) \mid d_2(\lambda) \mid \dots \mid d_r(\lambda)$（整除性是检验答案对错的关键）。
3. 初等因子（构建Jordan标准型的积木）：
    将所有非线性的不变因子 $d_k(\lambda)$ 分解为互不相同的素多项式的幂：$(\lambda - \lambda_1)^{n_1}, (\lambda - \lambda_2)^{n_2}, \dots$。这些幂因子（不包括常数1）就是初等因子。
    - **对应关系**：每一个初等因子 $(\lambda - \lambda_i)^k$ 对应一个 $k$ 阶的 Jordan 块 $J_k(\lambda_i)$。
4. **最小多项式 $m_A(\lambda)$**：
    - **定义**：使得 $m_A(A)=0$ 的最低次首1多项式。
    - **速算结论**：最小多项式等于最后一个（下标最大的）不变因子，即 $m_A(\lambda) = d_n(\lambda)$。
        
---

### **题型一：因子互推与Smith标准型**

这类题型通常出现在填空题或计算题的第一步。题目会给出秩 $r$ 和一组因子，求另一组因子或Smith标准型。

**得分策略与解题算法（列表法）：**

若已知 **初等因子**，求 **不变因子** 和 **Smith标准型**，请严格执行“**归位对齐法**”：

1. **制表**：画一个表格，行数等于矩阵的秩 $r$（通常是 $n$），列数等于不同特征值（$\lambda_i$）的个数。
2. **填入**：将给定的初等因子按照特征值分类。对于同一个特征值，**指数高**的初等因子填在表格的**最下方**（第 $r$ 行），指数低的往上填。如果某一行该特征值没有初等因子，填 $1$。
3. **计算**：
    - **不变因子 $d_k(\lambda)$**：表格中第 $k$ 行所有多项式的乘积。
    - **Smith标准型**：$\text{diag}(d_1, d_2, \dots, d_n)$。
    - **行列式因子 $D_k(\lambda)$**：前 $k$ 个不变因子的乘积（即 $D_k = d_1 d_2 \dots d_k$）。    

**示例演练**：

> 秩为3，初等因子为 $\lambda, \lambda^2, \lambda^3, \lambda+2, (\lambda+2)^2$。
> - 特征值 $\lambda$ 组：$\lambda^3, \lambda^2, \lambda$
> - 特征值 $\lambda+2$ 组：$(\lambda+2)^2, (\lambda+2)$，缺一个补1。
> **排队（自下而上）：**
> 
> - 第3行（$d_3$）：$\lambda^3, (\lambda+2)^2 \Rightarrow d_3(\lambda) = \lambda^3(\lambda+2)^2$
> - 第2行（$d_2$）：$\lambda^2, (\lambda+2) \Rightarrow d_2(\lambda) = \lambda^2(\lambda+2)$
> - 第1行（$d_1$）：$\lambda, 1 \Rightarrow d_1(\lambda) = \lambda$
> 验证：$d_1 | d_2 | d_3$，符合定理。

---

### **题型二：求Jordan标准型与过渡矩阵**

这是最经典的必考大题。通常要求 $J$ 和 $P$ 使得 $P^{-1}AP=J$。

**得分策略与解题算法：**

1. **求特征多项式**：计算 $|\lambda I - A|$，分解因式得到特征值及其代数重数。
2. **确定Jordan块的数量**：对于每个特征值 $\lambda_i$，计算矩阵 $B = A - \lambda_i I$ 的秩。
    - **几何重数 $q_i$**（即属于 $\lambda_i$ 的Jordan块总数）= $n - \text{rank}(A - \lambda_i I)$。
    - 若几何重数等于代数重数，则是对角阵；否则存在高阶Jordan块。
3. **确定Jordan块的阶数（点积法/秩阶梯法）**：
    - 计算 $(A - \lambda_i I)^k$ 的秩序列，利用公式 $N_k = \text{rank}(B^{k-1}) - 2\text{rank}(B^k) + \text{rank}(B^{k+1})$ 来确定 $k$ 阶Jordan块的个数。
    - **实战捷径**：对于 $3 \times 3$ 矩阵，若 $\lambda$ 是3重根且秩为2（零度为1），则只有一个块 $J_3(\lambda)$；若秩为1（零度为2），则有一个 $J_2(\lambda)$ 和一个 $J_1(\lambda)$。
4. **求过渡矩阵 $P$（这是难点）**：
    - 对于 $k$ 阶Jordan块 $J_k(\lambda_i)$，需要寻找**广义特征向量链** $\xi_1, \xi_2, \dots, \xi_k$。
    - 链的定义：$(A-\lambda_i I)\xi_k = \xi_{k-1}, \dots, (A-\lambda_i I)\xi_2 = \xi_1, (A-\lambda_i I)\xi_1 = 0$。
    - **求解顺序**：通常先解方程组 $(A-\lambda_i I)^k x = 0$ 找到 $\xi_k$，然后依次左乘 $(A-\lambda_i I)$ 得到其余向量。    

---

### **题型三：矩阵函数 $f(A)$ 的计算**

题目通常要求计算 $e^{tA}, \sin A, A^{100}$ 等。

**得分策略与解题算法：**

不要死记硬背所有方法，根据题目要求选择最优路径：

路径 A：若题目已要求求 Jordan 标准型 $J$ 和 $P$

直接使用定义法：

$$f(A) = P f(J) P^{-1}$$

其中 $f(J) = \text{diag}(f(J_1), f(J_2), \dots)$。

对于 $k$ 阶 Jordan 块 $J_k(\lambda)$，

$$f(J_k(\lambda)) = \begin{pmatrix} f(\lambda) & f'(\lambda) & \frac{f''(\lambda)}{2!} & \dots \\ 0 & f(\lambda) & f'(\lambda) & \dots \\ \vdots & \vdots & \ddots & \vdots \\ 0 & 0 & \dots & f(\lambda) \end{pmatrix}$$

注意：这种方法计算量大（涉及矩阵求逆和乘法），仅在已求出 $P$ 时推荐。

路径 B：若题目未要求求 $P$（推荐 Lagrange-Sylvester 插值法）

这是考研和期末考最快的方法，也是历年题中展示的“或”下面的方法。

1. **求最小多项式 $m_A(\lambda)$**：通常等于特征多项式（除非有重根且几何重数大）。设 $m_A(\lambda) = (\lambda - \lambda_1)^{k_1} \dots (\lambda - \lambda_s)^{k_s}$。
2. **设定插值多项式 $r(\lambda)$**：设 $r(\lambda)$ 次数为 $\deg(m_A) - 1$。
3. 列方程组：$$r^{(j)}(\lambda_i) = f^{(j)}(\lambda_i), \quad i=1\dots s, \ j=0\dots k_i-1$$
    即函数值相等，直至 $k_i-1$ 阶导数也相等。
4. **解出系数**：得到 $r(\lambda)$。
5. **代入矩阵**：$f(A) = r(A)$。


## 例题


---

### **第一战：初等因子与不变因子的转换（2021-2022真题）**

题目来源：2021-2022 第一学期（二次修订）

题目描述：如果 $\lambda$-矩阵 $A(\lambda)$ 的秩为 3，其初等因子为 $\lambda, \lambda^2, \lambda^3, \lambda+2, (\lambda+2)^2$。

求：$A(\lambda)$ 的不变因子和行列式因子。

#### **深度解析与步骤**

这道题考察的是我之前提到的“归位对齐法”。

第一步：分组与排序

将初等因子按“特征值”分组，并在每组内按“次数从高到低”排列：

- **$\lambda$ 组**：$\lambda^3, \lambda^2, \lambda$
- **$\lambda+2$ 组**：$(\lambda+2)^2, (\lambda+2)$
    
第二步：列表归位（关键步骤）

画一个表格，行数等于矩阵的秩（这里是 $r=3$）。

规则：将分好组的因子沉底填入。即每列最下方填次数最高的，不够的上面补 1。

|**行（不变因子）**|**λ 的因子**|**λ+2 的因子**|
|---|---|---|
|**第 1 行 ($d_1$)**|$\lambda$|$1$ (补位)|
|**第 2 行 ($d_2$)**|$\lambda^2$|$\lambda+2$|
|**第 3 行 ($d_3$)**|$\lambda^3$|$(\lambda+2)^2$|

**第三步：横向相乘得出结果**

- **不变因子**（每一行的乘积）：
    
    - $d_1(\lambda) = \lambda \cdot 1 = \mathbf{\lambda}$
        
    - $d_2(\lambda) = \lambda^2 \cdot (\lambda+2) = \mathbf{\lambda^3 + 2\lambda^2}$
        
    - $d_3(\lambda) = \lambda^3 \cdot (\lambda+2)^2 = \mathbf{\lambda^3(\lambda+2)^2}$
        
    - _自检_：显然 $d_1 | d_2 | d_3$，答案有效。
        
- **行列式因子**（前 $k$ 个不变因子的乘积）：
    
    - $D_1(\lambda) = d_1 = \mathbf{\lambda}$
        
    - $D_2(\lambda) = d_1 d_2 = \lambda \cdot \lambda^2(\lambda+2) = \mathbf{\lambda^3(\lambda+2)}$
        
    - $D_3(\lambda) = d_1 d_2 d_3 = \mathbf{\lambda^6(\lambda+2)^3}$
        

---

### **第二战：求Jordan标准型与过渡矩阵 P（2015-2016真题）**

题目来源：2015-2016 第一学期

题目描述：已知矩阵 $A = \begin{pmatrix} -3 & 3 & -2 \\ -7 & 6 & -3 \\ 1 & -1 & 2 \end{pmatrix}$。

(1) 求 $A$ 的 Jordan 标准型，并求变换矩阵 $P$
(2) 求线性微分方程组 $\frac{d\vec{X}}{dt} = A\vec{X}$ 的解

#### **深度解析与步骤**

这是计算量最大的一类题。求 $J$ 容易，求 $P$ 极易出错。我们严格按步骤执行。

第一步：求特征值

计算特征多项式 $| \lambda I - A |$：

$$\begin{vmatrix} \lambda+3 & -3 & 2 \\ 7 & \lambda-6 & 3 \\ -1 & 1 & \lambda-2 \end{vmatrix} = (\lambda-1)(\lambda-2)^2$$

- 特征值：$\lambda_1 = 1$（单重），$\lambda_2 = 2$（二重）。
    

**第二步：确定 Jordan 标准型 $J$**

- 对于 $\lambda_1=1$：一定是 1 阶块 $[1]$。
    
- 对于 $\lambda_2=2$：代数重数为 2。需看几何重数（即 $A-2I$ 的零空间维数）。
    
    计算 $A - 2I = \begin{pmatrix} -5 & 3 & -2 \\ -7 & 4 & -3 \\ 1 & -1 & 0 \end{pmatrix}$。
    
    易见第 1 行与第 3 行线性无关，秩 $r=2$。
    
    几何重数 $k = n - r = 3 - 2 = 1$。
    
    结论：只有一个特征向量，因此 $\lambda=2$ 只有一个 Jordan 块。
    
    $$J = \begin{pmatrix} 1 & 0 & 0 \\ 0 & 2 & 1 \\ 0 & 0 & 2 \end{pmatrix}$$
    
    _(注：这里将 $\lambda=2$ 的块写成 $\begin{pmatrix} 2 & 1 \\ 0 & 2 \end{pmatrix}$)_
    

第三步：求过渡矩阵 $P$（核心难点）

我们需要构造 $P = [\xi_1, \eta_1, \eta_2]$，满足 $P^{-1}AP = J$，即 $AP = PJ$。

对应的列关系为：

1. $A\xi_1 = 1\cdot\xi_1$ （$\xi_1$ 是 $\lambda=1$ 的特征向量）
2. $A\eta_1 = 2\cdot\eta_1$ （$\eta_1$ 是 $\lambda=2$ 的特征向量）
3. $A\eta_2 = 1\cdot\eta_1 + 2\cdot\eta_2 \Rightarrow (A-2I)\eta_2 = \eta_1$ （$\eta_2$ 是 $\lambda=2$ 的**广义特征向量**）

**计算过程**：

1. 求 $\xi_1$ ($\lambda=1$)：
    解 $(A-I)x = 0$。
    
    $\begin{pmatrix} -4 & 3 & -2 \\ -7 & 5 & -3 \\ 1 & -1 & 1 \end{pmatrix} \xrightarrow{rref} \begin{pmatrix} 1 & 0 & -1 \\ 0 & 1 & -2 \\ 0 & 0 & 0 \end{pmatrix} \Rightarrow \xi_1 = \begin{pmatrix} 1 \\ 2 \\ 1 \end{pmatrix}$
    
1. **求 $\eta_1$ 和 $\eta_2$ ($\lambda=2$) —— 链式求解法**：
    - 先求特征向量 $\eta_1$：
        解 $(A-2I)x = 0$。
        
        $\begin{pmatrix} -5 & 3 & -2 \\ -7 & 4 & -3 \\ 1 & -1 & 0 \end{pmatrix} \xrightarrow{r_1 \leftrightarrow r_3} \begin{pmatrix} 1 & -1 & 0 \\ 0 & 1 & 1 \\ 0 & 0 & 0 \end{pmatrix} \Rightarrow \begin{cases} x_1 = x_2 \\ x_2 = -x_3 \end{cases}$
        
        取 $x_3 = -1$，得 $\eta_1 = \begin{pmatrix} 1 \\ 1 \\ -1 \end{pmatrix}$。
        (注意：这里必须保留这个结果，它是生成广义特征向量的“种子”)
        
    - 再求广义特征向量 $\eta_2$：
        解方程 $(A-2I)\eta_2 = \eta_1$。
        $$\begin{pmatrix} -5 & 3 & -2 \\ -7 & 4 & -3 \\ 1 & -1 & 0 \end{pmatrix} \begin{pmatrix} x \\ y \\ z \end{pmatrix} = \begin{pmatrix} 1 \\ 1 \\ -1 \end{pmatrix}$$
        
        由第三行得：$x - y = -1 \Rightarrow y = x+1$。
        
        代入第一行：$-5x + 3(x+1) - 2z = 1 \Rightarrow -2x - 2z = -2 \Rightarrow z = 1-x$。
        
        令自由变量 $x=0$（最简取法），则 $y=1, z=1$。
        
        得 $\eta_2 = \begin{pmatrix} 0 \\ 1 \\ 1 \end{pmatrix}$。
        
2. **组装矩阵 $P$**：
    
    $$P = [\xi_1, \eta_1, \eta_2] = \begin{pmatrix} 1 & 1 & 0 \\ 2 & 1 & 1 \\ 1 & -1 & 1 \end{pmatrix}$$
    

第四步：微分方程组的解（送分题）

公式：$\vec{X}(t) = e^{tA}\vec{X}(0) = P e^{tJ} P^{-1} C$。

更直接的写法是利用基本解矩阵：

对应 $J$ 的解基为 $e^t, e^{2t}, te^{2t}$。

由于 $\lambda=2$ 处有一个 2 阶块，解的形式为：

$$\vec{X} = c_1 e^t \xi_1 + c_2 e^{2t} \eta_1 + c_3 e^{2t} (t\eta_1 + \eta_2)$$

将之前求出的向量代入即可：

$$\vec{X} = c_1 e^t \begin{pmatrix} 1 \\ 2 \\ 1 \end{pmatrix} + e^{2t} \left[ c_2 \begin{pmatrix} 1 \\ 1 \\ -1 \end{pmatrix} + c_3 \left( t \begin{pmatrix} 1 \\ 1 \\ -1 \end{pmatrix} + \begin{pmatrix} 0 \\ 1 \\ 1 \end{pmatrix} \right) \right]$$


#### **为什么微分方程组的解是那种形式？**

对于常系数线性微分方程组 $\frac{d\vec{X}}{dt} = A\vec{X}$，其解的本质是 $\vec{X}(t) = e^{tA}\vec{X}(0)$。当矩阵 $A$ 不能对角化（存在Jordan块）时，我们需要借助相似变换来“解耦”。

**推导过程：**

1. 坐标变换（解耦）：
    
    令 $\vec{X} = P\vec{Y}$，其中 $P$ 是使 $P^{-1}AP = J$ 的过渡矩阵。
    
    代入原方程：
    
    $$P\frac{d\vec{Y}}{dt} = AP\vec{Y} \implies \frac{d\vec{Y}}{dt} = P^{-1}AP\vec{Y} = J\vec{Y}$$
    
    现在方程变成了 $\dot{\vec{Y}} = J\vec{Y}$。由于 $J$ 是准对角阵，问题被分解为独立的 Jordan 块子系统。
    
2. 解 Jordan 块对应的子系统：
    
    考虑我们在上一题中遇到的 2 阶 Jordan 块 $J_2(\lambda) = \begin{pmatrix} \lambda & 1 \\ 0 & \lambda \end{pmatrix}$。
    
    对应的微分方程子系统是：
    
    $$\begin{cases} y_1'(t) = \lambda y_1(t) + y_2(t) \\ y_2'(t) = 0 \cdot y_1(t) + \lambda y_2(t) \end{cases}$$
    
    - 先解第二个方程（简单的指数增长）：
        
        $$y_2(t) = c_2 e^{\lambda t}$$
        
    - 代入第一个方程：
        
        $$y_1'(t) - \lambda y_1(t) = c_2 e^{\lambda t}$$
        
        这是一个一阶线性非齐次微分方程。使用常数变易法或积分因子 $e^{-\lambda t}$，解得：
        
        $$y_1(t) = c_1 e^{\lambda t} + c_2 \cdot t \cdot e^{\lambda t}$$
        
    - 写成向量形式：
        
        $$\vec{Y}_{block}(t) = \begin{pmatrix} c_1 e^{\lambda t} + c_2 t e^{\lambda t} \\ c_2 e^{\lambda t} \end{pmatrix} = c_1 e^{\lambda t} \begin{pmatrix} 1 \\ 0 \end{pmatrix} + c_2 e^{\lambda t} \begin{pmatrix} t \\ 1 \end{pmatrix}$$
        
3. 还原回 $\vec{X}$ 空间：
    
    $\vec{X} = P\vec{Y}$。注意 $P$ 的列向量就是广义特征向量链 $[\xi_1, \eta_1, \eta_2]$。
    
    对应到上面的 2 阶块，$\eta_1$ 对应 $y_1$（链头），$\eta_2$ 对应 $y_2$（链尾，但在 $J$ 中通常高阶向量排在后面，这里的对应关系取决于 $P$ 的排列顺序）。
    
    若 $P$ 排列为 $[\text{特征向量}, \text{广义特征向量}]$，即 $[\eta_1, \eta_2]$，则：
    
    $$\vec{X}(t) = [ \eta_1, \eta_2 ] \begin{pmatrix} y_1 \\ y_2 \end{pmatrix} = y_1 \eta_1 + y_2 \eta_2$$
    
    代入 $y_1, y_2$ 的解：
    
    $$\vec{X}(t) = (c_1 e^{\lambda t} + c_2 t e^{\lambda t})\eta_1 + (c_2 e^{\lambda t})\eta_2 = c_1 e^{\lambda t}\eta_1 + c_2 e^{\lambda t}(t\eta_1 + \eta_2)$$
    
    这就是上一题解中 $e^{2t}(t\eta_1 + \eta_2)$ 这一项的来源。**那个 $t$ 系数正是因为 Jordan 块右上角的 1 导致的耦合效应。**
    

---

### **最小多项式与矩阵函数**

题目来源： 2018-2019 第二学期

题目描述：已知矩阵 $A = \begin{pmatrix} 2 & -2 & 0 \\ 2 & 6 & 0 \\ 1 & 1 & 4 \end{pmatrix}$。

(1) 求矩阵的 Jordan 标准形和最小多项式；

(2) 求矩阵函数 $\sin A, e^{tA}$。

此题是考查**最小多项式**和**矩阵函数计算**的绝佳范例。

#### **步骤 1：求特征值与 Jordan 标准形**

1. 特征多项式：
    观察矩阵结构，是一个分块下三角矩阵。左上角是 $\begin{pmatrix} 2 & -2 \\ 2 & 6 \end{pmatrix}$，右下角是 $4$。
    $$|\lambda I - A| = (\lambda - 4) \begin{vmatrix} \lambda-2 & 2 \\ -2 & \lambda-6 \end{vmatrix}$$
    $$= (\lambda - 4) [ (\lambda-2)(\lambda-6) + 4 ] = (\lambda-4)(\lambda^2 - 8\lambda + 16) = (\lambda-4)^3$$
    
    - **特征值**：$\lambda = 4$（代数重数为 3）。
2. 确定 Jordan 块结构：
    计算 $B = A - 4I$ 的秩：
    $$A - 4I = \begin{pmatrix} -2 & -2 & 0 \\ 2 & 2 & 0 \\ 1 & 1 & 0 \end{pmatrix}$$
    显然第 1、2 列成比例，第 3 列为 0。矩阵秩 $r=1$。
    - **几何重数（Jordan 块个数）**：$k = n - r = 3 - 1 = 2$。
    - 结论：共有 2 个 Jordan 块。由于总阶数为 3，唯一的组合是一个 2 阶块和一个 1 阶块。
        $$J = \begin{pmatrix} 4 & 1 & 0 \\ 0 & 4 & 0 \\ 0 & 0 & 4 \end{pmatrix}$$
        

#### **步骤 2：确定最小多项式 $m_A(\lambda)$**

这是本题的关键得分点。

- **定义**：最小多项式的根与特征多项式相同，但其重数由**对应特征值的最大 Jordan 块的阶数**决定。
- **判断**：$\lambda=4$ 最大的 Jordan 块是 2 阶（即 $J_2(4)$）。
- 结果：$$m_A(\lambda) = (\lambda - 4)^2$$
    (对比：特征多项式是 $(\lambda-4)^3$。如果 $A$ 可对角化，最大块为 1 阶，最小多项式就是 $\lambda-4$。)
    

#### **步骤 3：计算矩阵函数 $\sin A$ 和 $e^{tA}$**

这里演示两种方法。考试时，如果第一问已经求出了 $J$，**方法一**通常更直观；如果没有求 $J$ 或 $J$ 很复杂，**方法二（Lagrange-Sylvester插值）** 更快。

**方法一：利用 Jordan 标准形公式**

公式：$f(A) = P f(J) P^{-1}$。但本题未要求求 $P$，直接算 $P$ 及其逆太耗时。我们需要观察题目——通常这类题如果只求 $f(A)$ 而不要求 $P$，往往暗示可以使用 Lagrange-Sylvester 插值法，或者 $P$ 非常好求。

鉴于第一问只问了 $J$ 没问 $P$，强行算 $P$ 是下策。我们采用方法二。

**方法二：Lagrange-Sylvester 插值法（利用最小多项式）**

这是处理矩阵函数的神技，必须掌握。

1. **依据**：$f(A) = r(A)$，其中 $r(\lambda)$ 是满足插值条件的多项式，且 $\deg(r) < \deg(m_A)$。
2. 设定：$m_A(\lambda) = (\lambda - 4)^2$，次数为 2。
    设 $r(\lambda) = a\lambda + b$（一次多项式）。
3. 插值条件：
    函数值相等：$r(4) = f(4)$
    导数值相等（因为 4 是二重根）：$r'(4) = f'(4)$
    

计算 (a)：求 $\sin A$

令 $f(\lambda) = \sin \lambda$。

- $r(4) = 4a + b = \sin 4$
- $r'(4) = a = \cos 4$
    解得：$a = \cos 4$， $b = \sin 4 - 4\cos 4$。
    代入 $A$：$$\sin A = (\cos 4) A + (\sin 4 - 4\cos 4) I$$$$= \cos 4 (A - 4I) + \sin 4 I$$
    
    将 $A-4I = \begin{pmatrix} -2 & -2 & 0 \\ 2 & 2 & 0 \\ 1 & 1 & 0 \end{pmatrix}$ 代入即可得到具体矩阵。
    

计算 (b)：求 $e^{tA}$

注意这里变量是 $t$，我们在对 $\lambda$ 求导时把 $t$ 看作常数。

令 $f(\lambda) = e^{t\lambda}$。

- $r(4) = 4a + b = e^{4t}$
    
- $r'(4) = a = t e^{4t}$ （注意 $f'(\lambda) = t e^{t\lambda}$）
    
    解得：$a = t e^{4t}$， $b = e^{4t} - 4t e^{4t} = e^{4t}(1 - 4t)$。
    

于是：

$$e^{tA} = aA + bI = t e^{4t} A + e^{4t}(1 - 4t) I$$

整理得更优雅的形式：

$$e^{tA} = e^{4t} [ t A + (1 - 4t) I ] = e^{4t} [ I + t(A - 4I) ]$$

**最终结果代入**：

$$e^{tA} = e^{4t} \left( \begin{pmatrix} 1 & 0 & 0 \\ 0 & 1 & 0 \\ 0 & 0 & 1 \end{pmatrix} + t \begin{pmatrix} -2 & -2 & 0 \\ 2 & 2 & 0 \\ 1 & 1 & 0 \end{pmatrix} \right) = e^{4t} \begin{pmatrix} 1-2t & -2t & 0 \\ 2t & 1+2t & 0 \\ t & t & 1 \end{pmatrix}$$

---

### **解题套路 SOP**

1. **看到微分方程组求解** $\rightarrow$ 本质是求 $e^{tA}$ $\rightarrow$ 利用 Jordan 块解耦，出现 $t e^{\lambda t}$ 是因为有非对角元素 。
2. **看到求最小多项式** $\rightarrow$ 先求特征值 $\rightarrow$ 看 $A-\lambda I$ 的秩定 Jordan 块大小 $\rightarrow$ 最大块阶数即为因子次数。
3. **看到求矩阵函数** $\rightarrow$ 若已知 $J$ 和 $P$，用 $P f(J) P^{-1}$；若无 $P$，务必使用 **Lagrange-Sylvester 插值法**（即 $r(A)$ 法），利用最小多项式的根和重数列方程组。


