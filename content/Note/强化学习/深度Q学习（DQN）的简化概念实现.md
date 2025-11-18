---
title: 深度Q学习（DQN）的简化概念实现
date: 2025-11-17
slug: blog-post-slug
tags:
  - 强化学习
  - DQN
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
![](Note/强化学习/assets/Pasted%20image%2020251117202133.png)
### 深度Q学习（DQN）的简化概念实现

假定我们已经有了两个神经网络（主网络和目标网络）以及一个回放缓存。

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim
import random
from collections import deque

# 假设的环境和网络定义
STATE_DIM = 4
ACTION_DIM = 2
REPLAY_BUFFER_SIZE = 10000
MINIBATCH_SIZE = 64
GAMMA = 0.99
C_ITERATIONS = 10  # 目标网络更新频率

# 1. 网络架构 (简化为单层感知机)
class QNetwork(nn.Module):
    def __init__(self, state_dim, action_dim):
        super(QNetwork, self).__init__()
        # Q-网络接收状态 S，输出每个动作 A 的 Q 值
        self.fc = nn.Sequential(
            nn.Linear(state_dim, 64),
            nn.ReLU(),
            nn.Linear(64, action_dim)
        )

    def forward(self, x):
        return self.fc(x)

# 2. 回放缓存（Experience Replay Buffer）
class ReplayBuffer:
    def __init__(self, capacity):
        # 使用 deque 实现 FIFO 队列
        self.buffer = deque(maxlen=capacity)

    def push(self, state, action, reward, next_state):
        # 存储转移四元组 (s, a, r, s')
        self.buffer.append((state, action, reward, next_state))

    def sample(self, batch_size):
        # 从缓存中均匀随机抽取 mini-batch
        minibatch = random.sample(self.buffer, batch_size)
        
        # 将样本转换为 PyTorch Tensor (这里使用 numpy 辅助转换)
        states = torch.FloatTensor(np.array([s for s, a, r, s_next in minibatch]))
        actions = torch.LongTensor([a for s, a, r, s_next in minibatch]).unsqueeze(-1)
        rewards = torch.FloatTensor([r for s, a, r, s_next in minibatch]).unsqueeze(-1)
        next_states = torch.FloatTensor(np.array([s_next for s, a, r, s_next in minibatch]))
        
        return states, actions, rewards, next_states

    def __len__(self):
        return len(self.buffer)

# 3. DQN 训练循环的核心逻辑
def dqn_update(main_net, target_net, optimizer, replay_buffer, iterations_count):
    
    if len(replay_buffer) < MINIBATCH_SIZE:
        return # 缓存不足，跳过更新

    # 1. 均匀抽取 mini-batch (Uniformly draw a mini-batch of samples from B)
    states, actions, rewards, next_states = replay_buffer.sample(MINIBATCH_SIZE)

    # 2. 计算目标值 y_T (Calculate the target value y_T)
    # 2a. 使用 target_net 预测下一状态 S' 的所有动作 Q 值
    with torch.no_grad():
        # max_a' Q_hat(S', a', w_T)
        next_q_values = target_net(next_states).max(1)[0].unsqueeze(-1) 
        # y_T = r + gamma * max_a' Q_hat(S', a', w_T)
        y_T = rewards + GAMMA * next_q_values 

    # 3. 计算当前 Q 值 (Q_hat(S, A, w))
    # 3a. 使用 main_net 预测当前状态 S 的所有动作 Q 值
    current_q_values = main_net(states).gather(1, actions) # 仅取出实际执行动作 A 的 Q 值

    # 4. 更新主网络 (Update the main network to minimize (y_T - Q_hat(s, a, w))^2)
    # 损失函数 J = (y_T - Q_hat(s, a, w))^2 
    loss = nn.MSELoss()(current_q_values, y_T)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # 5. 目标网络更新 (Set w_T = w every C iterations)
    if iterations_count % C_ITERATIONS == 0:
        target_net.load_state_dict(main_net.state_dict())
        
    return loss.item()

# 4. 初始化和主循环示例 (省略了环境交互部分，仅展示更新)
# main_q_net = QNetwork(STATE_DIM, ACTION_DIM)
# target_q_net = QNetwork(STATE_DIM, ACTION_DIM)
# target_q_net.load_state_dict(main_q_net.state_dict())
# optimizer = optim.Adam(main_q_net.parameters(), lr=0.0005)
# replay_buffer = ReplayBuffer(REPLAY_BUFFER_SIZE)
#
# # 假设已经向 buffer 中填充了足够的样本...
#
# for iteration in range(1, 1000):
#     loss = dqn_update(main_q_net, target_q_net, optimizer, replay_buffer, iteration)
#     # ... 环境交互和 buffer 填充
```

---

#### 问题一：为什么没有策略更新？（Why no policy update?）

DQN 算法的理论基础是**动作价值函数（Action-Value Function）** $Q(S, A)$ 的学习和逼近，属于**价值学习（Value-Based Learning）** 的范畴。

1.  **策略的隐式定义：** 在DQN中，**策略 $\pi$ 并非通过一个独立的参数化模型显式定义和优化**。相反，最优策略 $\pi^*(A|S)$ 是**隐式地**从最优动作价值函数 $Q^*(S, A)$ 中导出的，即通过**贪婪地选择**具有最大 Q 值的动作：
    $$\pi^*(S) = \arg \max_{A \in \mathcal{A}(S)} Q^*(S, A)$$
2.  **更新机制：** DQN 的更新过程仅集中于最小化贝尔曼最优方程的误差（即最小化 $\left(y_T - \hat{q}(s, a, w)\right)^2$），本质上是学习 $Q$ 函数的参数 $w$。
    $$\text{梯度更新: } w \leftarrow w - \alpha \nabla_w \left(y_T - \hat{q}(s, a, w)\right)^2$$
    **当 $Q$ 函数 $\hat{q}(S, A, w)$ 被优化到收敛时，其参数 $w$ 已经间接地定义了最优策略 $\pi^*$。** 因此，DQN 算法通过**价值函数的优化**来**间接实现策略的优化**，而无需显式执行一个单独的“策略更新”步骤。

#### 问题二：为什么不使用我们推导出的策略更新方程？（Why not using the policy update equation that we derived?）

此问题暗示了在策略梯度（Policy Gradient）方法中存在的**显式策略更新方程**。不使用它们的原因再次归结于DQN的**价值学习**性质及其**off-policy**特性。

1.  **价值学习 vs. 策略学习：**
    *   **策略学习（如REINFORCE, A2C/A3C）：** 其目标是最大化预期的累积奖励 $\mathbb{E}[G_0]$，通过对策略参数 $\theta$ 求梯度 $\nabla_{\theta} J(\theta)$ 来直接更新策略 $\pi_{\theta}(A|S)$。这是**显式的策略更新**。
    *   **价值学习（DQN）：** 其目标是学习最优价值函数 $Q^*(S, A)$，通过最小化贝尔曼误差的损失函数 $J(w)$ 来更新价值函数参数 $w$。这是一种**价值驱动的更新**。
2.  **off-policy的特性：** DQN 是一种**off-policy**算法，它使用由**行为策略** $\pi_b$（通常是 $\epsilon$-贪婪探索策略）采集的数据来训练**目标策略** $\pi$（即由 $\arg \max \hat{q}$ 导出的贪婪策略）。
    *   **Q-学习（Q-learning）** 的理论基础是其更新规则本身就能够直接收敛到最优 $Q$ 函数 $Q^*$，**与采集数据的策略无关**，只要采集策略 $\pi_b$ 能够充分探索（即**充分覆盖**状态-动作空间）。
    *   如果使用策略梯度（Policy Gradient）方法，则需要引入**重要性采样（Importance Sampling）** 来校正行为策略 $d^{\pi_b}$ 与目标策略 $d^{\pi}$ 之间的分布差异，以维持无偏性。DQN 避免了复杂的**重要性采样**机制，而是直接通过贝尔曼最优性算子（Bellman Optimality Operator）的收敛性来保证其有效性。

因此，DQN 不使用策略更新方程，是因为它从根本上选择了**价值迭代**的路径，而非**策略梯度**的路径，其收敛性由贝尔曼最优性算子的不动点性质所保证。