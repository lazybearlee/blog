---
title: PPO
date: 2025-11-23
slug: blog-post-slug
tags:
  - 强化学习
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
### 马尔可夫决策过程与策略梯度范式

近端策略优化（PPO）算法的数学构建始于对强化学习问题的标准形式化，即马尔可夫决策过程（MDP）。定义元组 $\mathcal{M} = \langle \mathcal{S}, \mathcal{A}, \mathcal{P}, \mathcal{R}, \gamma \rangle$，其中 $\mathcal{S} \subseteq \mathbb{R}^n$ 为状态空间，$\mathcal{A} \subseteq \mathbb{R}^m$ 为动作空间。策略 $\pi_\theta: \mathcal{S} \to \Delta(\mathcal{A})$ 被参数化为由向量 $\theta \in \mathbb{R}^d$ 控制的概率分布函数，通常由神经网络逼近。我们的核心优化目标是最大化期望累积回报 $J(\theta)$：

$$J(\theta) = \mathbb{E}_{\tau \sim \pi_\theta} \left[ \sum_{t=0}^{\infty} \gamma^t r(s_t, a_t) \right]$$

此处 $\tau = (s_0, a_0, s_1, a_1, \dots)$ 表示由策略 $\pi_\theta$ 诱导出的轨迹。基于策略梯度定理，目标函数关于参数 $\theta$ 的梯度 $\nabla_\theta J(\theta)$ 可表示为：

$$\nabla_\theta J(\theta) = \mathbb{E}_{t} \left[ \nabla_\theta \log \pi_\theta(a_t | s_t) A_t \right]$$

其中 $A_t$ 为优势函数（Advantage Function），量化了在状态 $s_t$ 下执行动作 $a_t$ 相对于平均策略表现的优劣程度。在标准的策略梯度方法中，参数更新遵循 $\theta_{k+1} = \theta_k + \alpha \nabla_\theta J(\theta_k)$。然而，这种一阶梯度更新存在步长 $\alpha$ 敏感性问题：若步长过大，策略参数在参数空间中的剧烈变动可能导致策略性能的崩溃式下降，且由于数据分布的非平稳性，这种下降往往不可逆。

### 重要性采样与替代目标函数

为了解决参数更新的稳定性问题，引入信赖域（Trust Region）概念，旨在限制新策略 $\pi_\theta$ 与旧策略 $\pi_{\theta_{old}}$ 之间的差异。鉴于直接求解带有 KL 散度约束的优化问题（如 TRPO 所做）计算复杂度高（涉及海森矩阵的逆），我们转而寻求一阶近似方法。利用重要性采样（Importance Sampling）技术，可以在旧策略 $\pi_{\theta_{old}}$ 采样的样本上评估新策略的目标函数。定义概率比率 $r_t(\theta)$ 为新旧策略在动作 $a_t$ 上的概率密度之比：

$$r_t(\theta) = \frac{\pi_\theta(a_t | s_t)}{\pi_{\theta_{old}}(a_t | s_t)}$$

基于此定义，显然当 $\theta = \theta_{old}$ 时，$r_t(\theta) = 1$。由此构建替代目标函数（Surrogate Objective）$L^{CPI}(\theta)$（Conservative Policy Iteration）：

$$L^{CPI}(\theta) = \mathbb{E}_t \left[ r_t(\theta) \hat{A}_t \right]$$

该函数是一个局部近似。然而，若无约束地最大化 $L^{CPI}$，会导致 $r_t(\theta)$ 偏离 1 过远，从而破坏重要性采样的数值稳定性，并违背局部近似的前提。

### 截断机制与下界优化

PPO 的核心创新在于通过引入截断操作（Clipping）直接在目标函数中施加约束，从而避免复杂的约束优化求解。定义超参数 $\epsilon$（通常取 $0.1$ 或 $0.2$）作为允许的策略变动范围。PPO 的目标函数 $L^{CLIP}(\theta)$ 定义为未截断目标与截断目标的逐点最小值（Pessimistic Lower Bound）：

$$L^{CLIP}(\theta) = \mathbb{E}_t \left[ \min \left( r_t(\theta) \hat{A}_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon) \hat{A}_t \right) \right]$$

在此公式中，$\text{clip}(x, l, h)$ 函数将 $x$ 限制在区间 $[l, h]$ 内。对该目标函数的梯度流动力学进行分情况讨论，可以揭示其“悲观更新”的本质：

考察优势函数 $\hat{A}_t > 0$ 的情形（即当前动作优于平均水平）：

此时我们需要增大该动作的概率，即提升 $r_t(\theta)$。

若 $r_t(\theta) < 1+\epsilon$，则目标函数简化为 $r_t(\theta) \hat{A}_t$，梯度推动 $r_t$ 增大。

若 $r_t(\theta) \ge 1+\epsilon$，则目标函数被截断为 $(1+\epsilon)\hat{A}_t$，关于 $\theta$ 的梯度为零。这实际上由 $\min$ 操作符强制设置了一个概率提升的上限，防止策略过度自信地更新。

考察优势函数 $\hat{A}_t < 0$ 的情形（即当前动作劣于平均水平）：

此时我们需要减小该动作的概率，即降低 $r_t(\theta)$。

若 $r_t(\theta) > 1-\epsilon$，梯度推动 $r_t$ 减小。

若 $r_t(\theta) \le 1-\epsilon$，目标函数被截断，梯度消失。这防止了将某一动作的概率降得过低（这可能破坏探索性）。

通过取最小值操作，PPO 构建了一个被截断目标函数所包络的下界函数。最大化这个下界，保证了我们在提升策略性能的同时，不会因概率比率 $r_t(\theta)$ 的剧烈震荡而导致策略坍塌。

### 算法实现

将上述数学推导映射到计算实现中，需特别注意张量维度的对齐与梯度传播的控制。在 PyTorch 环境下，假设我们通过广义优势估计（GAE）计算得到了优势张量 `advantages` $\in \mathbb{R}^{B}$（$B$ 为批量大小）以及对应的对数概率张量。

```Python
import torch
import torch.nn as nn

def ppo_loss(new_log_probs, old_log_probs, advantages, epsilon=0.2):
    """
    计算 PPO-Clip 损失函数。
    
    参数:
    new_log_probs: \pi_\theta(a_t|s_t) 的对数概率, shape [Batch_Size]
    old_log_probs: \pi_{\theta_{old}}(a_t|s_t) 的对数概率, shape [Batch_Size]
    advantages: 估计的优势函数 \hat{A}_t, shape [Batch_Size]
    epsilon: 截断参数 \epsilon
    """
    
    # 1. 计算概率比率 r_t(\theta)
    # 使用 exp(log_a - log_b) = a/b 以保证数值稳定性
    ratio = torch.exp(new_log_probs - old_log_probs)
    
    # 2. 计算未截断部分: r_t(\theta) * A_t
    surr1 = ratio * advantages
    
    # 3. 计算截断部分: clip(r_t(\theta), 1-\epsilon, 1+\epsilon) * A_t
    surr2 = torch.clamp(ratio, 1.0 - epsilon, 1.0 + epsilon) * advantages
    
    # 4. 取最小值 (注意 PyTorch 默认为最小化，故取负号以进行梯度下降最大化)
    # L^{CLIP} = E[min(surr1, surr2)]
    loss = -torch.min(surr1, surr2).mean()
    
    return loss
```

通过自动微分机制，`loss.backward()` 将仅在未被截断的样本区域内回传非零梯度，从而在参数更新过程中动态地实施信赖域约束。