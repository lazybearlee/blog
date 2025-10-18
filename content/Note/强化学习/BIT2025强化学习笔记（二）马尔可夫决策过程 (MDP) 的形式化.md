---
title: BIT2025强化学习笔记（二）马尔可夫决策过程 (MDP) 的形式化
date: 2025-10-17
slug: blog-post-slug
tags:
  - 强化学习
categories:
  - 笔记
description: 描述
draft: false
state: "0"
---

## 学习模块 2：马尔可夫决策过程 (MDP) 的形式化

**学习目标：**

1. 严格定义 MDP 的五元组结构及其数学组件。
2. 形式化理解**策略**（Policy）的概念及其在控制系统中的作用。
3. 深入分析**折扣因子** $\gamma$ 在数学上如何保证无限序列回报的收敛性。
4. 严格推导并理解**策略评估**（Policy Evaluation）和**最优策略**（Optimal Policy）的数学定义。

---

## MDP 的形式化定义 (对应 Slides 2, 6)

马尔可夫决策过程 (MDP) 是对马尔可夫过程 (MP) 的扩展，增加了**动作（Actions）** 和**奖励（Rewards）**，以实现决策和优化的目标。

**定义 1.1：马尔可夫决策过程 (MDP)**

一个有限马尔可夫决策过程由一个五元组 $(\mathcal{S}, \mathcal{A}, \mathcal{P}, \mathcal{R}, \gamma)$ 严格定义：

1.  **状态空间 $\mathcal{S}$：** 有限的状态集合。
2.  **动作空间 $\mathcal{A}$：** 有限的动作集合。
3.  **转移概率函数 $\mathcal{P}$：** MDP 的核心动态模型。
    这是一个四元函数：$\mathcal{P}: \mathcal{S} \times \mathcal{A} \times \mathcal{S} \to [0, 1]$。
    我们定义 $P(s' | s, a)$ 为在状态 $s$ 采取动作 $a$ 后，转移到下一个状态 $s'$ 的概率：
    $$P(s' | s, a) = P(S_{t+1}=s' | S_t=s, A_t=a)$$
    （注意：这个函数继承了马尔可夫性质和平稳性。）

4.  **奖励函数 $\mathcal{R}$：** 描述即时回报。
    幻灯片 4 中提及的奖励函数 $R(s_t, a_t)$ 是一种形式。更严谨的定义通常是期望奖励或基于转移的奖励：
    $$\mathcal{R}(s, a, s') = \mathbb{E}[R_{t+1} | S_t=s, A_t=a, S_{t+1}=s']$$
    或者，简化为基于状态-动作对的期望即时奖励：
    $$R(s, a) = \mathbb{E}[R_{t+1} | S_t=s, A_t=a] = \sum_{s' \in \mathcal{S}} P(s'|s, a) \cdot \mathcal{R}(s, a, s')$$
    （我们通常默认奖励函数也是**平稳的**，即 $R(s, a)$ 不随时间 $t$ 变化。）

5.  **折扣因子 $\gamma$：** $\gamma \in [0, 1]$。

---

## 奖励最大化与折扣因子 $\gamma$ 的数学意义 (对应 Slides 4, 5)

强化学习的根本目标是最大化**预期回报**（Expected Return）。

### 2.1 回报 (Return) 的定义

Agent 试图最大化的是从时间 $t$ 开始的奖励总和。我们用 $G_t$ 表示从时间 $t$ 开始的**回报**（Return）。

对于一个**有限时域**（Finite Horizon, $h$ 步）：
$$G_t = R_{t+1} + R_{t+2} + \dots + R_{t+h} = \sum_{k=1}^h R_{t+k}$$

对于一个**无限时域**（Infinite Horizon）：
$$G_t = R_{t+1} + R_{t+2} + R_{t+3} + \dots = \sum_{k=1}^\infty R_{t+k}$$

### 2.2 引入折扣因子 ($\gamma$) 解决收敛性问题

对于无限时域（Infinite Horizon）问题，除非所有奖励 $R_{t+k}$ 都为零，否则简单求和 $G_t$ 将趋于无穷大（$\sum R_{t+k} \to \infty$），这使得不同策略的回报无法比较。

**解决方案：折扣回报 (Discounted Return)** (Slide 5)

引入折扣因子 $\gamma \in [0, 1)$，重新定义无限时域下的回报 $G_t$:

$$G_t = R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \dots = \sum_{k=1}^\infty \gamma^{k-1} R_{t+k}$$

**数学推导：保证收敛性**

假设即时奖励的绝对值有一个上界 $R_{\max} = \sup_t |R_{t+k}| < \infty$。
则回报 $G_t$ 的绝对值 $|G_t|$ 有界：

$$|G_t| = \left| \sum_{k=1}^\infty \gamma^{k-1} R_{t+k} \right| \leq \sum_{k=1}^\infty \gamma^{k-1} |R_{t+k}| \leq R_{\max} \sum_{k=0}^\infty \gamma^k$$

这是一个几何级数求和。由于我们假设 $\gamma < 1$，该几何级数收敛：
$$\sum_{k=0}^\infty \gamma^k = \frac{1}{1 - \gamma}$$

因此，回报 $G_t$ 是有界的：
$$|G_t| \leq R_{\max} \cdot \frac{1}{1 - \gamma}$$

**结论：** 引入 $\gamma < 1$ 在数学上保证了无限时域回报序列的**收敛性**，使得最大化期望回报的目标成为一个定义良好的优化问题。

**重要性质：递归关系**
折扣回报 $G_t$ 具有一个关键的递归结构，这是贝尔曼方程的基础：
$$G_t = R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \dots$$
$$G_t = R_{t+1} + \gamma (R_{t+2} + \gamma R_{t+3} + \dots)$$
$$G_t = R_{t+1} + \gamma G_{t+1}$$

---

## 策略 (Policy) 的形式化 (对应 Slide 8)

Agent 的决策机制被称为**策略**。

**定义 3.1：策略 $\pi$ (Policy)**

策略 $\pi$ 是一个函数，它定义了在给定状态下选择某个动作的概率。

$$\pi: \mathcal{S} \times \mathcal{A} \to [0, 1]$$

其中 $\pi(a|s)$ 是在状态 $s$ 时选择动作 $a$ 的概率，且必须满足 $\sum_{a \in \mathcal{A}} \pi(a|s) = 1$。

**确定性策略 (Deterministic Policy)：**

幻灯片 8 提到 $\pi(s_t) = a_t$，这是一种**确定性策略**。它是一个函数 $\pi: \mathcal{S} \to \mathcal{A}$，直接将状态映射到动作。
在这种情况下，对于任何状态 $s$，只有一个动作 $a$ 满足 $\pi(a|s)=1$，其余动作概率为 0。

**MDP 与策略的结合**

一旦确定了一个策略 $\pi$，Agent 在状态 $s$ 选择了动作 $a = \pi(s)$，环境就会按照转移概率 $P(s'|s, a)$ 转移。

**关键点：** 给定一个固定的策略 $\pi$，MDP 会退化成一个马尔可夫过程 (MP)，我们称之为 **MP($\pi$)**。

在这个 MP($\pi$) 中：

1.  **状态空间** 仍是 $\mathcal{S}$。
2.  **状态转移矩阵** $P^\pi$ 的元素 $P^\pi(s'|s)$ 可以计算为：
    $$P^\pi(s'|s) = \sum_{a \in \mathcal{A}} \pi(a|s) \cdot P(s'|s, a)$$
    （即在状态 $s$ 下，我们根据 $\pi$ 选择 $a$，再根据 $P$ 转移到 $s'$。）
3.  **期望即时奖励** $R^\pi(s)$ 可以计算为：
    $$R^\pi(s) = \sum_{a \in \mathcal{A}} \pi(a|s) \cdot R(s, a)$$

---

## 值函数：策略评估的形式化 (对应 Slide 9)

**值函数**（Value Function）是衡量一个策略 $\pi$ 在特定状态 $s$ 下好坏的数学工具。它是期望回报。

### 4.1 状态值函数 $V^\pi(s)$

**定义 4.1.1：状态值函数 (State-Value Function)**

状态值函数 $V^\pi(s)$ 定义为从状态 $s$ 开始，遵循策略 $\pi$ 所能获得的**期望折扣回报**：

$$V^\pi(s) = \mathbb{E}_\pi [G_t | S_t = s]$$

其中 $\mathbb{E}_\pi[\cdot]$ 表示在策略 $\pi$ 下对所有随机变量（动作 $A_t$ 和后续状态 $S_{t+1}, S_{t+2}, \dots$）求期望。

**策略评估：** 计算 $V^\pi(s)$ 的过程称为策略评估（Policy Evaluation）。

### 4.2 贝尔曼期望方程 (Bellman Expectation Equation)

利用 $G_t$ 的递归性质 $G_t = R_{t+1} + \gamma G_{t+1}$，我们可以对值函数进行分解。

$$V^\pi(s) = \mathbb{E}_\pi [R_{t+1} + \gamma G_{t+1} | S_t = s]$$

根据期望的线性性质 $\mathbb{E}[X+Y] = \mathbb{E}[X] + \mathbb{E}[Y]$：

$$V^\pi(s) = \mathbb{E}_\pi [R_{t+1} | S_t = s] + \gamma \mathbb{E}_\pi [G_{t+1} | S_t = s]$$

我们来分解右侧的两个期望项：

**第一项：期望即时奖励** $\mathbb{E}_\pi [R_{t+1} | S_t = s]$

$$\mathbb{E}_\pi [R_{t+1} | S_t = s] = \sum_{a \in \mathcal{A}} \pi(a|s) \cdot R(s, a) = R^\pi(s)$$

**第二项：期望未来折扣回报** $\mathbb{E}_\pi [G_{t+1} | S_t = s]$

根据全期望公式和马尔可夫性质，我们必须对所有可能的下一步状态 $s'$ 求和：
$$\mathbb{E}_\pi [G_{t+1} | S_t = s] = \sum_{s' \in \mathcal{S}} P(S_{t+1}=s' | S_t=s) \cdot \mathbb{E}_\pi [G_{t+1} | S_{t+1}=s']$$

注意到 $\mathbb{E}_\pi [G_{t+1} | S_{t+1}=s']$ 定义为 $V^\pi(s')$。
同时，利用我们前面定义的 $P^\pi(s'|s)$，我们得到：
$$\mathbb{E}_\pi [G_{t+1} | S_t = s] = \sum_{s' \in \mathcal{S}} P^\pi(s'|s) \cdot V^\pi(s')$$

**最终的贝尔曼期望方程：**

将两项代回原式，得到著名的贝尔曼期望方程（用于策略评估）：

$$V^\pi(s) = R^\pi(s) + \gamma \sum_{s' \in \mathcal{S}} P^\pi(s'|s) \cdot V^\pi(s')$$

### 4.3 动作值函数 $Q^\pi(s, a)$

为了方便决策，我们需要评估在状态 $s$ 采取特定动作 $a$ 的价值。

**定义 4.3.1：动作值函数 (Action-Value Function)**

动作值函数 $Q^\pi(s, a)$ 定义为在状态 $s$ 采取动作 $a$，然后从下一步开始遵循策略 $\pi$ 所能获得的**期望折扣回报**：
$$Q^\pi(s, a) = \mathbb{E}_\pi [G_t | S_t = s, A_t = a]$$

$Q^\pi(s, a)$ 的贝尔曼方程推导与 $V^\pi(s)$ 类似：
$$Q^\pi(s, a) = R(s, a) + \gamma \sum_{s' \in \mathcal{S}} P(s'|s, a) \cdot V^\pi(s')$$

**两者关系：** $V^\pi(s)$ 是 $Q^\pi(s, a)$ 在策略 $\pi$ 下对所有可能动作 $a$ 的期望：
$$V^\pi(s) = \sum_{a \in \mathcal{A}} \pi(a|s) \cdot Q^\pi(s, a)$$

---

## 步骤 5：最优策略与贝尔曼最优方程 (对应 Slide 9, 12)

MDP 的**目标**（Goal）是找到一个**最优策略** $\pi^*$，使得在所有状态下，其值函数最大。

### 5.1 最优值函数

**定义 5.1.1：最优状态值函数 $V^*(s)$**

$$V^*(s) = \max_{\pi} V^\pi(s)$$

**定义 5.1.2：最优动作值函数 $Q^*(s, a)$**

$$Q^*(s, a) = \max_{\pi} Q^\pi(s, a)$$

### 5.2 贝尔曼最优方程 (Bellman Optimality Equation)

最优策略 $\pi^*$ 必须是**贪婪的**（Greedy）——在每一步都选择能带来最高 $Q$ 值的动作。

因此，最优值函数 $V^*$ 满足一个特殊的递归关系，即**贝尔曼最优方程**：

$$V^*(s) = \max_{a \in \mathcal{A}} Q^*(s, a)$$

将 $Q^*(s, a)$ 展开，我们得到 $V^*(s)$ 的贝尔曼最优方程：

$$V^*(s) = \max_{a} \left\{ R(s, a) + \gamma \sum_{s' \in \mathcal{S}} P(s'|s, a) \cdot V^*(s') \right\}$$

**最优策略的提取：**

找到 $V^*(s)$ 后，最优策略 $\pi^*$ 可以通过在每个状态下选择使贝尔曼最优方程最大化的动作来确定 (Slide 12):

$$\pi^*(s) = \arg\max_{a} \left\{ R(s, a) + \gamma \sum_{s' \in \mathcal{S}} P(s'|s, a) \cdot V^*(s') \right\}$$

这个公式是**值迭代**（Value Iteration）和大部分基于值的方法的理论基础。
