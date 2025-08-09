---
title: 腾讯广告大赛2025-Baseline的损失函数优化分析
date: 2025-08-09
slug: blog-post-slug
tags:
  - 腾讯广告大赛2025
  - 生成式推荐
  - 推荐系统
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
### **1. 当前损失函数 (`BCEWithLogitsLoss`) 是否合适？**

当前的损失函数是`torch.nn.BCEWithLogitsLoss`，即带有Logits的二元交叉熵损失。

在`main.py`的训练循环中，它的用法是：

```python
# ... inside training loop ...
indices = np.where(next_token_type == 1) # 只对目标是item的位置计算loss
# 对正样本，希望模型的输出(pos_logits)接近1
loss = bce_criterion(pos_logits[indices], pos_labels[indices]) 
# 对负样本，希望模型的输出(neg_logits)接近0
loss += bce_criterion(neg_logits[indices], neg_labels[indices])
```

这种方法被称为**Pointwise Learning to Rank (点对学习排序)**。它的核心思想是，将排序问题转化为一个二分类问题：

*   对于一个（用户序列，正样本广告）对，它是一个**正类别**（label=1）。
*   对于一个（用户序列，负样本广告）对，它是一个**负类别**（label=0）。

模型的目标是独立地、准确地判断每个样本的类别。

**结论：** `BCEWithLogitsLoss` 在这个场景下是**合理但非最优**的。
*   **合理性**：它确实能驱动模型去给正样本打高分，给负样本打低分，这是排序的基础。
*   **非最优性**：比赛的评估指标是`HitRate@10`和`NDCG@10`，这些是**排序指标 (Ranking Metrics)**。它们不关心一个广告的绝对得分是多少，只关心**正样本广告相对于所有负样本广告的排名**。而Pointwise方法并不直接优化这个相对顺序，它只关心每个样本的“绝对正确性”。这是一个核心的矛盾点。

---

### **2. 损失函数与预测目标 (`infer.py`) 是否一致？**

让我们回顾`infer.py`的流程：
1.  `model.predict()`为每个用户生成一个最终的`user_embedding`。
2.  `model.save_item_emb()`为候选库里的**所有广告**生成`item_embedding`。
3.  Faiss对一个`user_embedding`和**整个候选库**的`item_embedding`计算相似度（点积），然后返回相似度最高的Top-10。

**差距分析：**

*   **预测（Inference）时**：模型需要解决的是一个“**百里挑一**”甚至“**万里挑一**”的问题。它需要保证用户真实点击的那个广告，其得分要**高于候选库里成千上万个其他广告**。这是一个**Listwise（列表）** 的排序问题。
*   **训练（Training）时**：模型解决的是一个“**二选一**”的简化问题。它在每一时刻只看到了**一个正样本**和**一个负样本**。它只需要保证正样本的得分高于这一个随机挑选的负样本即可，模型并没有被激励去拉开与**其他潜在负样本**的差距。

**后果：**
由于训练和预测目标的不一致，模型在训练时可能“用力过猛”于区分一个简单的负样本，而没有学会如何在一个大规模的候选池中将正样本排到最前面。这很可能是导致分数仅有0.03的重要原因。模型学会了“好瓜比坏瓜甜”，但没学会在“一堆好瓜里挑出最甜的那个”。

---

### **3. 我们可以如何优化损失函数？**

优化的核心思想是：**让训练时的目标更接近预测时的排序目标**。我们可以从易到难尝试以下几种方案：

#### **方案一：增加负样本数量（改进Pointwise）**

这是最简单的改进。既然一个负样本不够，那我们就用多个。

*   **思路**：对于每一个正样本，我们采样 N 个负样本（例如N=4或更多）。
*   **修改**：
    1.  修改`dataset.py`中的`__getitem__`，使其为每个正样本返回一个包含N个负样本ID和特征的列表。
    2.  修改`model.py`的`forward`，使其能处理多个负样本。
    3.  修改`main.py`的损失计算，将N个负样本的损失累加起来。
*   **优点**：实现简单，能有效让模型见到更多样的负例，学习更鲁棒的决策边界。

#### **方案二：采用Pairwise Loss（成对学习排序）**

Pairwise方法不关心每个样本的绝对得分，而是直接优化“正样本得分 > 负样本得分”这个相对关系。这比Pointwise更接近排序的本质。

*   **典型代表：BPR Loss (Bayesian Personalized Ranking)**
    *   **思路**：最大化`pos_logits`和`neg_logits`之间的差值。
    *   **公式**：`Loss = -log(sigmoid(pos_logits - neg_logits))`
    *   **修改 (`main.py`)**:
        ```python
        # 损失函数定义不需要改变，我们直接在计算时实现BPR
        # from torch.nn.functional import logsigmoid
        
        # ... 在训练循环中 ...
        indices = np.where(next_token_type == 1)[0] # 获取需要计算loss的索引
        
        # 只选择有效的logits
        valid_pos_logits = pos_logits.flatten()[indices]
        valid_neg_logits = neg_logits.flatten()[indices]

        # 计算BPR Loss
        loss = -torch.log_sigmoid(valid_pos_logits - valid_neg_logits).mean()

        # 后续的L2正则化、反向传播等不变
        ```
*   **优点**：目标非常明确，就是让正例的排名高于负例。在很多场景下，BPR比BCE Loss效果更好。实现起来也非常简单，只需要改变一行损失计算代码。

#### **方案三：采用Listwise Loss（列表学习排序）**

这是最接近最终评估指标的方案。它一次性考虑一个包含“1个正样本和N个负样本”的列表，并直接优化这个列表的排序。

*   **思路**：将排序问题看作一个多分类问题。在 `[正样本, 负样本1, 负样本2, ..., 负样本N]` 这个列表中，我们希望模型能以最高的概率将“正样本”分到第一位。
*   **实现**：
    1.  首先需要像方案一那样，为每个正样本采样N个负样本。
    2.  将这 `N+1` 个样本的logits拼接起来，形成一个logits向量。
    3.  使用`softmax`将其转换为概率分布。
    4.  我们的目标（label）是这个列表的第一个元素（正样本）概率为1，其余为0。即 `target = [1, 0, 0, ..., 0]`。
    5.  使用交叉熵损失来计算模型输出的概率分布与目标分布之间的差距。
*   **修改 (`main.py`)**:
    ```python
    # 假设模型forward返回 pos_logits 和一个包含N个负样本logits的列表 neg_logits_list
    # pos_logits: [B, L], neg_logits_list: list of N tensors, each [B, L]

    indices = np.where(next_token_type == 1)[0]
    
    # 将所有logits拼接成一个列表
    # 假设 neg_logits_list 是一个 [B, L, N] 的张量
    all_logits = torch.cat([pos_logits.unsqueeze(2), neg_logits_list], dim=2) # Shape: [B, L, 1+N]
    
    # 只选择有效位置的logits
    valid_logits = all_logits.view(-1, 1 + N)[indices] # Shape: [num_valid, 1+N]

    # 目标label是第一位，即索引为0
    target_labels = torch.zeros(valid_logits.shape[0], dtype=torch.long, device=args.device)

    # 使用交叉熵损失
    loss = torch.nn.CrossEntropyLoss()(valid_logits, target_labels)
    ```

### **行动建议**

考虑到迭代优化的原则，我建议按以下顺序尝试：

1.  **首选尝试：Pairwise BPR Loss**。
    *   **原因**：这是从Pointwise到Pairwise的理念转变，非常契合排序任务。而且代码改动极小，几乎没有引入新的复杂性，是性价比最高的尝试。
2.  **如果BPR效果提升有限，再尝试：增加负样本数量 + Listwise Loss**。
    *   **原因**：这个方案在理念上最接近NDCG等排序指标，潜力最大。但它需要修改数据加载部分来提供N个负样本，实现起来会稍微复杂一些。

我们先从最直接、最核心的**Pairwise BPR Loss**开始，看看它能否为我们带来初步但显著的性能提升。
