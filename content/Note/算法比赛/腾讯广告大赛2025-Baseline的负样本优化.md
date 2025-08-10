---
title: 腾讯广告大赛2025-Baseline的负样本优化
date: 2025-08-09
slug: blog-post-slug
tags:
  - 推荐系统
  - 腾讯广告大赛2025
  - 生成式推荐
  - 负样本
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
在我们基于[腾讯广告大赛2025-Baseline的损失函数优化分析](Note/算法比赛/腾讯广告大赛2025-Baseline的损失函数优化分析.md)引入BPR Loss之后，结果没有提升，甚至有所下降，这在模型优化过程中是非常常见且有价值的现象。它告诉我们，仅仅改变损失函数这个单一变量，并没有直接命中当前模型的性能瓶颈。BPR没有奏效，可能的原因有以下几点：

1.  **负样本质量过低**：BPR Loss的核心是拉开正负样本的差距。但我们目前采用的是**随机负采样**。对于一个活跃用户，随机采样的广告大概率是用户本来就不感兴趣的“简单负样本”。模型能够轻易地将`pos_logit`与`neg_logit`的差值推到一个很大的正数，导致`log_sigmoid`的值迅速饱和接近0，梯度消失，模型很快就“学不动了”。它在简单任务上过拟合了，却没有学到区分那些“有点像但用户最终没点”的困难负样本的能力。
2.  **L2正则化问题**：回顾`main.py`的代码，有一个细节值得注意：
    ```python
    # 添加L2正则化项
    for param in model.item_emb.parameters():
        loss += args.l2_emb * torch.norm(param)
    ```
    这个L2正则化项只对`item_emb`的参数进行了惩罚，而`user_emb`、`pos_emb`以及Transformer部分都没有被正则化。更重要的是，它加在BPR/BCE损失**之后**。BPR Loss的值域通常比BCE Loss小，一个固定的`l2_emb`系数可能对BPR Loss产生了过大的影响，主导了梯度，使得模型更倾向于让embedding的模长变小，而不是优化排序。

**结论**：我们当前最可能遇到的瓶颈是**负样本的质量和数量不足**，导致无论是BCE还是BPR，模型都在做一个过于简化的任务，无法学会应对推理时海量候选集的挑战。

---

### **下一步优化规划：引入高质量负样本**

我们的应该转向：**在训练时为模型创造一个更接近真实推理环境的“小规模排序”任务**。

#### **任务一：实现In-Batch Negatives（Batch内负采样）**

这是目前业界主流且效果显著的负采样策略，也是从Pairwise向Listwise思想演进的第一步，性价比极高。

**是什么？**
对于一个batch内的一个用户（我们称之为“锚点用户”），**其他用户的正样本**，对于这个锚点用户来说，都是非常高质量的负样本。因为这些广告是“有人点击的”，说明它们本身是高质量、有吸引力的广告，只是“恰好不符合当前用户的口味”。模型如果能学会区分“我喜欢的”和“别人喜欢的”，其推荐能力将大大增强。

**如何实现？**

这需要对`main.py`中的训练逻辑进行一次重构。

1.  **获取Batch内的所有正样本Embedding**：在一个训练step中，我们会得到一个batch的正样本序列`pos`（Shape: `[B, L]`）。我们可以用`model.feat2emb()`一次性计算出所有这些正样本的Embedding，得到`pos_embs`（Shape: `[B, L, D]`）。
2.  **构建相似度矩阵**：对于序列中的每一个位置 `l`，我们需要计算锚点用户的序列表示 `log_feats[:, l, :]`（Shape: `[B, D]`）与**整个batch内所有正样本** `pos_embs[:, l, :]`（Shape: `[B, D]`）的点积相似度。
    *   这可以通过一个矩阵乘法高效完成：`logits_matrix = torch.matmul(log_feats[:, l, :], pos_embs[:, l, :].T)`。结果`logits_matrix`的Shape是`[B, B]`。
    *   `logits_matrix[i, j]` 代表了第 `i` 个用户的序列表示与第 `j` 个用户的正样本之间的相似度。
3.  **构建损失函数**：
    *   对于第 `i` 个用户，它的正样本是 `pos_embs[i, l, :]`，对应的logit是对角线元素 `logits_matrix[i, i]`。
    *   它的负样本是所有其他用户的正样本，对应的logits是第 `i` 行的非对角线元素 `logits_matrix[i, j]` (where `j != i`)。
    *   我们可以把这个问题看作一个B维的分类问题。目标是让模型在`logits_matrix`的每一行中，将对角线元素（正样本）的概率预测为最大。
    *   这可以直接使用**交叉熵损失** (`CrossEntropyLoss`)。对于一个大小为B的batch，我们的logits是`logits_matrix` (`[B, B]`)，而我们的标签是一个从0到B-1的序列`[0, 1, 2, ..., B-1]`。

**修改要点 (`main.py`)**:

```python
# ... 在训练循环中 ...
# 不再需要 neg 数据
seq, pos, _, token_type, next_token_type, next_action_type, seq_feat, pos_feat, _ = batch

# 1. 获取用户序列的最终表示
log_feats = model.log2feats(seq, token_type, seq_feat) # Shape: [B, L, D]

# 2. 获取所有正样本的Embedding
pos_embs = model.feat2emb(pos, pos_feat, include_user=False) # Shape: [B, L, D]

# 3. 筛选需要计算loss的位置
indices = np.where(next_token_type.flatten() == 1)[0]
# 如果没有需要计算的位置，则跳过
if len(indices) == 0:
    continue

# 将 [B, L, D] -> [B*L, D]
log_feats_flat = log_feats.view(-1, args.hidden_units)
pos_embs_flat = pos_embs.view(-1, args.hidden_units)

# 只选择有效位置的向量
valid_log_feats = log_feats_flat[indices] # Shape: [N, D], N是有效位置数量
valid_pos_embs = pos_embs_flat[indices] # Shape: [N, D]

# 4. 计算相似度矩阵
# [N, D] @ [D, N] -> [N, N]
logits_matrix = torch.matmul(valid_log_feats, valid_pos_embs.T)

# 5. 构建交叉熵损失
# 标签是_对角线_上的元素，即 0, 1, 2, ...
labels = torch.arange(logits_matrix.shape[0], device=args.device)
loss = torch.nn.CrossEntropyLoss()(logits_matrix, labels)

# （可选）可以增加一个温度系数来调节softmax的平滑度
# logits_matrix /= temperature

# 后续步骤：L2正则化、反向传播等
# ...
```

#### **任务二：优化L2正则化**

**修改要点 (`main.py`)**:

1.  找到优化器的定义：
    ```python
    optimizer = torch.optim.Adam(model.parameters(), lr=args.lr, betas=(0.9, 0.98))
    ```
2.  添加`weight_decay`参数（它等效于L2正则化）：
    ```python
    optimizer = torch.optim.Adam(model.parameters(), lr=args.lr, betas=(0.9, 0.98), weight_decay=args.l2_emb)
    ```
3.  **移除**训练循环中手写的L2正则化代码：
    ```python
    # (删除这几行)
    # for param in model.item_emb.parameters():
    #     loss += args.l2_emb * torch.norm(param)
    ```

### 损失分析

在实现inbatch时我们能观察到一个现象——In-batch loss收敛在5.0左右，而BPR loss在0.01左右——是**完全正常且符合预期的**。这非但不是一个问题，反而是一个积极的信号，表明模型正在学习一个更困难、更有意义的任务。
### **1. 两种Loss的数学本质差异**

#### **BPR Loss / BCE Loss (Pairwise / Pointwise)**

*   **BPR Loss公式**: `Loss = -log(sigmoid(pos_logit - neg_logit))`
*   **核心目标**: 让 `pos_logit` 比 `neg_logit` 大。
*   **值域分析**:
    *   当模型学习得非常好时，`pos_logit - neg_logit`会是一个很大的正数（比如 `+10`）。
    *   `sigmoid(+10)` 的值无限接近于 `1`。
    *   `log(1)` 的值等于 `0`。
    *   因此，BPR Loss的**理论下限是0**。当模型面对简单的随机负样本时，它可以非常轻易地将loss推向0.01甚至更低。这看起来很美好，但可能是一种“虚假的繁荣”，因为它只表明模型学会了区分“苹果”和“石头”，而不是区分“红富士”和“花牛”。

#### **In-Batch Cross-Entropy Loss (Listwise)**

*   **代码中的实现**: `Loss = CrossEntropyLoss(logits_matrix, labels)`
*   **核心目标**: 在一个大小为N的logits向量（`logits_matrix`的一行）中，让目标位置（对角线）的logit值成为最大，并与其他的logit值拉开差距。
*   **值域分析**:
    *   `CrossEntropyLoss(logits, target)` 在数学上等价于 ` -log(softmax(logits)[target]) `。
    *   在我们的场景中，对于`logits_matrix`的第一行 `[logit_11, logit_12, ..., logit_1N]`，它的目标是`0`号位置（即`logit_11`）。
    *   所以，它对应的loss是 ` -log( softmax([logit_11, ..., logit_1N])[0] ) `，也就是 ` -log( exp(logit_11) / (exp(logit_11) + exp(logit_12) + ... + exp(logit_1N)) ) `。
    *   让我们来推导一下loss为`5.0`意味着什么：
        *   `Loss = 5.0`
        *   `log(softmax(logits)[target]) = -5.0`
        *   `softmax(logits)[target] = exp(-5.0)`
        *   `exp(-5.0)` 约等于 `0.0067`。

**这意味着什么？**

当loss收敛在5.0左右时，模型对于一个给定的用户，能以大约**0.67%的概率**，在整个batch的N个高质量候选广告中，准确地将属于他自己的那个正样本排在第一位。

### **2. 为什么这个结果是正常的？**

1.  **任务难度急剧增加**：
    *   **BPR**: 任务是“二选一”。理想情况下，随机猜对的概率是50%。
    *   **In-Batch**: 任务是“N选一”，N等于我们的batch size里有效样本的数量。假设`batch_size`为128，N约等于128。随机猜对的概率是 `1/128 ≈ 0.78%`。
    *   我们的模型达到了`0.67%`的准确率，这说明它几乎还处于**随机猜测**的水平！这恰恰证明了In-batch负样本的**有效性**——它们对于模型来说太难了，以至于模型在初始阶段和随机乱猜表现差不多。这正是我们想要的起点！如果loss一上来就很低，反而说明负样本太简单了。
2.  **Loss的绝对值没有可比性**：
    *   BPR Loss的“优秀值”在0附近。
    *   Cross-Entropy Loss的“随机值”是 `-log(1/N)`。如果N=128，那么随机猜测的loss应该是 `-log(1/128) = log(128) ≈ 4.85`。
    *   我们观察到的loss收敛在**5.0左右**，这与理论上的随机猜测值`4.85`惊人地吻合！这证明了我们的代码实现是正确的，并且模型确实在面对一个大小约为128的分类任务。
3.  **收敛是一个积极信号**：
    *   Loss从一个更高的初始值（可能会是6或7）下降并**收敛**在5.0附近，说明训练过程是稳定的。梯度没有爆炸，模型正在努力学习，但由于任务太难，暂时只能达到接近随机猜测的水平。
    *   相比之下，BPR loss迅速下降到0.01，更像是一种“假性收敛”或“过拟合于简单任务”，模型并没有真正学到精细的排序能力。

### **总结与展望**

在对比不同类型的损失函数时，它们的绝对值没有任何可比性，我们真正需要关心的是：

1.  **Loss是否稳定下降并收敛？** （是的，我们的观察证实了这一点）
2.  **Loss的量级是否符合其数学定义和任务难度？** （是的，5.0左右完美符合N≈128的交叉熵损失）
3.  **最终的线下/线上评估指标（HitRate, NDCG）是否有提升？** （这是我们最终的衡量标准）
