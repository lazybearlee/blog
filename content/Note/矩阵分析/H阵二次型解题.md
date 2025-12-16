---
title: "H阵二次型解题"
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
![](Note/矩阵分析/assets/Pasted%20image%2020251213231939.png)
这道题有一个**巨大的“福利”**：矩阵是**分块对角**的。如果你能一眼看出这个结构，计算量直接减少一半。

下面我带你用“无脑操作流程 (SOP)”拿下这 10 分。

---

### **解题核心逻辑 (SOP)**

1.  **观察结构**：矩阵右下角的 $0, 0, 4$ 把矩阵分割成了两部分。可以直接把 $4$ 提出来作为一个特征值，只需要算左上角的 $2 \times 2$ 矩阵。
2.  **求特征值**：算出 $\lambda_1, \lambda_2, \lambda_3$。
3.  **求特征向量**：算出对应的向量，并**单位化**（除以模长）。
4.  **组装 $U$**：把单位特征向量拼成矩阵。
5.  **写结果**：标准形就是 $\sum \lambda_i |y_i|^2$，定性看 $\lambda$ 的符号。

---

### **详细步骤演示**

#### **Step 1: 拆解矩阵求特征值（起手式）**

观察矩阵 $A$：
$$A = \begin{pmatrix} 3 & 2+i & 0 \\ 2-i & 7 & 0 \\ 0 & 0 & 4 \end{pmatrix}$$

这显然是一个分块矩阵：
$$A = \begin{pmatrix} A_{11} & 0 \\ 0 & 4 \end{pmatrix}$$
其中 $A_{11} = \begin{pmatrix} 3 & 2+i \\ 2-i & 7 \end{pmatrix}$。

**直接得出第一个特征值**：
* **$\lambda_3 = 4$**（对应特征向量在 $z$ 轴方向）。

**计算 $A_{11}$ 的特征值**：
特征多项式 $|\lambda I - A_{11}| = 0$：
$$
\begin{vmatrix} \lambda-3 & -(2+i) \\ -(2-i) & \lambda-7 \end{vmatrix} = (\lambda-3)(\lambda-7) - (2+i)(2-i)
$$
* **复数运算小贴士**：$(a+bi)(a-bi) = a^2 + b^2$。这里 $(2+i)(2-i) = 2^2 + 1^2 = 5$。

方程变为：
$$(\lambda^2 - 10\lambda + 21) - 5 = \lambda^2 - 10\lambda + 16 = 0$$
因式分解：
$$(\lambda - 2)(\lambda - 8) = 0$$

**得到另外两个特征值**：
* **$\lambda_1 = 2$**
* **$\lambda_2 = 8$**

**汇总特征值**：$\lambda_1 = 2, \lambda_2 = 8, \lambda_3 = 4$。

---

#### **Step 2: 求特征向量并单位化（核心计算）**

我们需要解 $(A - \lambda I)x = 0$。

**1. 对于 $\lambda_1 = 2$（代入 $2 \times 2$ 块）**
$$
\begin{pmatrix} 2-3 & -(2+i) \\ -(2-i) & 2-7 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = \begin{pmatrix} -1 & -(2+i) \\ -(2-i) & -5 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = 0
$$
取第一行方程：$-x_1 - (2+i)x_2 = 0 \Rightarrow x_1 = -(2+i)x_2$。
令 $x_2 = 1$，则 $x_1 = -(2+i)$。
得到向量 $\xi_1 = \begin{pmatrix} -(2+i) \\ 1 \\ 0 \end{pmatrix}$。*(补上第三维 0)*

**单位化**：
模长平方 $||\xi_1||^2 = |-(2+i)|^2 + |1|^2 = (2^2+1^2) + 1 = 5+1 = 6$。
模长 $= \sqrt{6}$。
单位向量 **$u_1 = \frac{1}{\sqrt{6}} \begin{pmatrix} -(2+i) \\ 1 \\ 0 \end{pmatrix}$**。

**2. 对于 $\lambda_2 = 8$（代入 $2 \times 2$ 块）**
$$
\begin{pmatrix} 8-3 & -(2+i) \\ -(2-i) & 8-7 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = \begin{pmatrix} 5 & -(2+i) \\ -(2-i) & 1 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = 0
$$
取第二行方程（比较简单）：$-(2-i)x_1 + x_2 = 0 \Rightarrow x_2 = (2-i)x_1$。
令 $x_1 = 1$，则 $x_2 = 2-i$。
得到向量 $\xi_2 = \begin{pmatrix} 1 \\ 2-i \\ 0 \end{pmatrix}$。

**单位化**：
模长平方 $||\xi_2||^2 = |1|^2 + |2-i|^2 = 1 + 5 = 6$。
模长 $= \sqrt{6}$。
单位向量 **$u_2 = \frac{1}{\sqrt{6}} \begin{pmatrix} 1 \\ 2-i \\ 0 \end{pmatrix}$**。

**3. 对于 $\lambda_3 = 4$**
观察原矩阵，$\lambda=4$ 对应的是独立的第三行/列。
特征向量直接就是 $z$ 轴单位向量：
单位向量 **$u_3 = \begin{pmatrix} 0 \\ 0 \\ 1 \end{pmatrix}$**。

---

#### **Step 3: 写出结果（拿分时刻）**

**(1) 写出标准形和酉变换**

* **标准形**：
    直接把特征值填进去。
    $$f(y) = 2|y_1|^2 + 8|y_2|^2 + 4|y_3|^2$$
    *(注意：一定要写模的平方 $|y|^2$，因为是复数域)*

* **酉变换 $U$**：
    把 $u_1, u_2, u_3$ 拼起来（顺序要和特征值对应）。
    $$
    U = [u_1, u_2, u_3] = \begin{pmatrix} \frac{-(2+i)}{\sqrt{6}} & \frac{1}{\sqrt{6}} & 0 \\ \frac{1}{\sqrt{6}} & \frac{2-i}{\sqrt{6}} & 0 \\ 0 & 0 & 1 \end{pmatrix}
    $$
    变换公式为 $X = UY$。

**(2) 判断定性**

* **判据**：看特征值的符号。
* **分析**：特征值分别为 $2, 8, 4$。
* **结论**：
    因为所有特征值 $\lambda_i > 0$，所以该 Hermitian 二次型是 **正定** 的。

---

### **总结：避坑指南**

1.  **复数模长**：算向量长度时，是 $|x|^2 = \bar{x}x$。比如 $|2+i|^2 = 2^2+1^2=5$，千万别算成 $(2+i)^2 = 3+4i$。
2.  **分块处理**：看到题目里那一排 0，第一时间反应“这是分块矩阵”，千万别傻乎乎去算 3 阶行列式展开，容易算错。
3.  **标准形写法**：Hermite 二次型的标准形系数一定是实数，变量要加绝对值符号。
4.  **$U$ 矩阵**：列向量必须单位化！如果不单位化，$U$ 就不是酉矩阵，就扣分了。