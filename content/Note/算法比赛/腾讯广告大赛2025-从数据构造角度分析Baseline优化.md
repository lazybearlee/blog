---
title: 腾讯广告大赛2025-从数据构造角度分析Baseline优化
date: 2025-08-06
slug: blog-post-slug
tags:
  - 推荐系统
  - 广告推荐
  - 腾讯广告大赛2025
  - 生成式推荐
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---
### 序列推荐中的挑战与优化目标回顾

在深入优化之前，我们首先回顾一下当前模型和数据，并明确我们的优化目标。

#### 1.1 当前模型与数据特点

1.  **数据来源：** `seq.jsonl` 包含用户的完整行为序列，这些序列是按时间顺序排列的。每条记录包含 `user_id`, `item_id`, `user_feature`, `item_feature`, `action_type`, `timestamp`。
2.  **关键信息：** `action_type` 明确指出了是 **01曝光点击序列**，这意味着序列中不仅有用户实际点击（或购买）的记录，还有用户仅仅被曝光但未点击的记录。这是一个非常重要的信号。
3.  **特征丰富性：** 模型支持用户和物品的多种特征类型：稀疏、数组、多模态、连续。`item_feat_dict.json` 存储了物品的丰富特征。
4.  **模型架构：** `BaselineModel` 基于Transformer，能够处理序列信息，并通过 `feat2emb` 融合了多种特征。使用了Flash Attention进行性能优化。
5.  **当前损失函数：** 从 `forward` 函数的返回值 `pos_logits, neg_logits` 来看，模型当前似乎采用了**成对（Pairwise）** 的损失形式，即预测正样本得分和负样本得分，通常会结合BCEWithLogitsLoss或BPR Loss。

#### 1.2 现有方法的局限性分析


1.  **特征融合的局限性：**
    *   当前模型在 `feat2emb` 中，简单地将所有 Embedding 后的特征 `torch.cat` 拼接起来，然后通过一个线性层 (`itemdnn`/`userdnn`) 降维。
    *   这种方法虽然能够融合特征，但它**假设了所有特征是独立贡献的，并且其交互关系是线性的**。它未能显式地建模不同特征之间可能存在的复杂、非线性的交叉关系。例如，用户性别特征和某个物品类别特征的组合，可能比它们各自单独的贡献更重要。这确实“忽略了所有输入的重要性”——更准确地说，是忽略了它们之间相互作用的重要性。

2.  **损失函数与优化目标：**
    *   虽然提供了曝光序列信息，但如果模型仍仅使用传统的点击率预测损失（如BCEWithLogitsLoss），它更侧重于预测**单个物品是否会被点击**，而非**整个列表的排序质量**。
    *   **NDCG (Normalized Discounted Cumulative Gain)** 是一种更关注排序位置和相关性分级的评价指标。简单的点击预测损失与NDCG之间存在**目标不一致性 (Mismatch)**。一个高点击率的模型可能在NDCG上表现不佳，因为它可能将所有物品的点击概率都预测得很高，但在区分用户真正偏好的少数物品和大量一般偏好的物品方面能力不足。
    *   曝光序列的引入，正是为了从 **“预测点击”** 迈向 **“优化排序”**。我们不仅要知道用户可能点击什么，还要知道在曝光给用户的多个物品中，哪些物品应该排在更靠前的位置。

#### 1.3 优化方向与核心目标

基于上述分析，我们的优化核心目标是：
1.  **从点预测到排序优化：** 利用曝光序列的丰富信息，将模型从单纯的点击率预测，转向直接优化排序质量，使其能够更好地捕获用户在**一系列被曝光物品中的相对偏好**。
2.  **提升特征交互能力：** 改进特征融合机制，让模型能够自动发现并学习不同特征之间的高阶、非线性交叉关系，从而更精细地建模用户兴趣和物品属性。
3.  **丰富特征表达：** 引入新的、对序列推荐至关重要的特征，进一步提升模型对用户行为模式和兴趣演变的理解。

现在，我们将逐一深入探讨这些优化方向。

---

### 结合曝光序列优化排序 (NDCG) 的策略

#### 2.1 为什么曝光序列很重要？

在推荐系统中，用户行为数据往往是“隐式反馈”：我们主要观察到用户点击了哪些物品，但并不知道他们“没点击”的物品是“不喜欢”还是“没看到”。曝光序列（Impression Sequence）为我们提供了宝贵的信息：用户看到了哪些物品，但在其中又选择了哪些。

考虑以下两种情况：
1.  用户被曝光了物品 $A$ 和 $B$，点击了 $A$。
2.  用户被曝光了物品 $C$ 和 $D$，点击了 $C$。

如果只看点击数据，我们知道用户喜欢 $A$ 和 $C$。但有了曝光数据，我们知道在 $A$ 和 $B$ 中，用户更偏好 $A$；在 $C$ 和 $D$ 中，用户更偏好 $C$。这构成了**相对偏好关系**。

曝光序列使得我们能够构建**负样本的上下文更具信息量**：被曝光但未点击的物品，比从整个海量物品库中随机采样的负样本，更有可能是“困难负样本”。用户看到了它们，但最终没有点击，这表明它们可能与用户的兴趣有所重叠，但不是最匹配的。优化模型去区分这些“被曝光但未点击”的物品，比区分完全不相关的物品更具挑战性，也更能提升模型在细粒度上的辨别能力，从而直接优化排序质量。

#### 2.2 优化NDCG的损失函数策略

NDCG是一个考虑了相关性分级和位置衰减的排序指标。直接优化NDCG通常是困难的，因为它不可导。因此，我们通常通过优化与NDCG高度相关的**代理损失函数 (Surrogate Loss Functions)**。

当前的 `BaselineModel` 的 `forward` 方法返回 `pos_logits, neg_logits`，这暗示它使用了成对损失。我们可以在此基础上进行优化。

**2.2.1 引入曝光上下文的成对损失**

最直接的优化方式是利用曝光数据来构造更有效的正负样本对。

假设对于一个用户 $u$，在一个序列位置 $t$ 上，他点击了物品 $i$（正样本），同时被曝光了物品 $j_1, j_2, \dots, j_k$ 但未点击。那么，我们可以构建如下类型的对比关系：

1.  **点击 vs. 曝光未点击：** 目标是让用户 $u$ 对点击物品 $i$ 的偏好得分，高于对曝光未点击物品 $j_x$ 的偏好得分。
2. $$
    \mathcal{L}_{\text{ExposurePair}} = \sum_{(u,i) \in D_{click}} \sum_{j \in D_{exposure\_neg}(u,i)} -\log \sigma(\text{score}(u,i) - \text{score}(u,j))
    $$
    
    其中 $D_{click}$ 是点击样本集合，$D_{exposure\_neg}(u,i)$ 是在与 $(u,i)$ 相同曝光上下文下，用户 $u$ 被曝光但未点击的物品集合。

在 `BaselineModel` 中，`pos_seqs` 和 `neg_seqs` 分别代表正样本ID和负样本ID。`next_action_type` 可以用来区分哪些是点击，哪些是曝光。
*   如果 `next_action_type` 为1（点击），那么 `pos_seqs` 中的物品就是真实点击的正样本。
*   如果 `next_action_type` 为0（曝光未点击），那么 `neg_seqs` 中的物品就是**基于曝光上下文的困难负样本**。

**如何修改：**
在 `forward` 函数中，我们可以根据 `next_action_type` 来选择不同的损失计算方式。
当前的 `pos_logits` 对应用户序列和正样本的相似度，`neg_logits` 对应用户序列和负样本的相似度。
如果 `neg_seqs` 能够被精心地构建为“曝光但未点击”的物品，那么现有的 `pos_logits - neg_logits` 结构就能直接用于优化这种偏好关系。

**2.2.2 列表式排序损失 (List-wise Ranking Loss)**

成对损失虽然有效，但它只考虑了每对样本的相对顺序，没有考虑整个排序列表的全局最优性。列表式排序损失直接操作整个列表，旨在优化整个列表的排序质量。

一个典型的列表式损失是**SoftRank**或基于Softmax的排序损失。
如果我们能为每个用户在每个序列位置上，构建一个由一个正样本和 $K$ 个负样本组成的列表，那么我们可以使用类似InfoNCE的结构：

对于用户 $u$ 在序列位置 $t$ 上的行为，其点击了物品 $i$，同时在同一曝光上下文下有 $K$ 个曝光未点击的物品 $j_1, \dots, j_K$。我们希望 $i$ 的得分最高。

假设 `log_feats` 是用户序列在位置 $t$ 的Embedding $\mathbf{h}_{u,t}$。
正样本 $i$ 的Embedding是 $\mathbf{e}_i$。
负样本 $j_k$ 的Embedding是 $\mathbf{e}_{j_k}$。

那么，可以构建一个Softmax概率分布，表示在所有曝光物品中，点击物品 $i$ 的概率：

$$
P(i | \mathbf{h}_{u,t}, \{j_1, \dots, j_K\}) = \frac{\exp(\text{score}(\mathbf{h}_{u,t}, \mathbf{e}_i) / \tau)}{\sum_{m=1}^K \exp(\text{score}(\mathbf{h}_{u,t}, \mathbf{e}_{j_m}) / \tau) + \exp(\text{score}(\mathbf{h}_{u,t}, \mathbf{e}_i) / \tau)}
$$

然后最小化 $-\log P(i | \mathbf{h}_{u,t}, \{j_1, \dots, j_K\})$, 这就是我们之前在[推荐系统中的对比损失](Blog/Deeplearning/推荐系统中的对比损失.md)讨论的InfoNCE损失。

**在 `forward` 函数中的体现：**
目前的 `forward` 函数返回 `pos_logits` 和 `neg_logits`，这与InfoNCE的 `all_logits` (包含了正负样本的相似度) 结构非常相似。我们可以将 `pos_logits` 视为正样本的得分，`neg_logits` 视为负样本的得分，然后将它们拼接起来，喂给 `F.cross_entropy` 函数，并指定正样本的类别索引为0。

**修改思路（伪代码）:**

```python
# 假设 log_feats 已经计算好
# pos_embs 和 neg_embs 也已经计算好

# 这里需要确保 pos_embs 是 (batch_size, seq_len, 1, hidden_units) 
# neg_embs 是 (batch_size, seq_len, num_negative_samples, hidden_units)
# 这样才能在最后一个维度上cat，并在之后的操作中正确广播。
# 如果 pos_embs 是 (batch_size, seq_len, hidden_units)，需要 .unsqueeze(2)

# 确保 pos_embs 和 neg_embs 的维度是可拼接的
# 假设 pos_embs 形状 (B, L, D), neg_embs 形状 (B, L, N_neg, D)
# 我们需要把 pos_embs 扩展到 (B, L, 1, D)
pos_embs_expanded = pos_embs.unsqueeze(2) # Shape: (batch_size, seq_len, 1, hidden_units)

# 将正负样本Embedding拼接在一起
# Shape: (batch_size, seq_len, 1 + num_negative_samples, hidden_units)
all_candidate_embs = torch.cat([pos_embs_expanded, neg_embs], dim=2)

# 计算用户序列与所有候选（正+负）物品的相似度
# log_feats: (batch_size, seq_len, hidden_units)
# all_candidate_embs: (batch_size, seq_len, 1 + num_negative_samples, hidden_units)
# 期望结果: (batch_size, seq_len, 1 + num_negative_samples)
# 可以使用 einsum 或逐元素乘法后求和
# 例如: scores = torch.einsum('bld,blnd->bln', log_feats, all_candidate_embs)
# 或者: scores = (log_feats.unsqueeze(2) * all_candidate_embs).sum(dim=-1)
# 我们使用更直接的矩阵乘法，需要reshape
batch_size, seq_len, _ = log_feats.shape
num_candidates = 1 + neg_embs.shape[2] # neg_embs.shape[2] 是 num_negative_samples

# Reshape log_feats for matrix multiplication: (B*L, D)
log_feats_flat = log_feats.view(-1, log_feats.shape[-1]) 
# Reshape all_candidate_embs for matrix multiplication: (B*L, N_cand, D) -> (B*L, D, N_cand) for transpose
all_candidate_embs_flat = all_candidate_embs.view(-1, num_candidates, all_candidate_embs.shape[-1])

# Calculate scores: (B*L, D) @ (B*L, D, N_cand) -> (B*L, N_cand)
scores_flat = torch.bmm(log_feats_flat.unsqueeze(1), all_candidate_embs_flat.transpose(1, 2)).squeeze(1)
# Reshape back to (B, L, N_cand)
scores = scores_flat.view(batch_size, seq_len, num_candidates)


# 应用温度系数
# temperature = some_float_value (hyperparameter)
scaled_scores = scores / temperature

# 构建目标标签：正样本始终是第一个 (索引为 0)
# targets 形状为 (batch_size, seq_len)
# 注意：只有 next_mask == 1 (物品token) 的位置才计算损失
targets = torch.zeros_like(next_mask, dtype=torch.long) # 初始化为0
targets = targets[next_mask == 1] # 只保留物品位置的target

# 筛选出需要计算损失的 scores
# scores_for_loss 形状为 (num_valid_items, 1 + num_negative_samples)
scores_for_loss = scaled_scores[next_mask == 1]

# 计算InfoNCE损失 (交叉熵损失)
loss = F.cross_entropy(scores_for_loss, targets)

# 返回 loss，而不是 pos_logits, neg_logits
# 或者如果需要保持兼容性，可以返回 pos_logits, neg_logits
# 但它们现在是 infoNCE loss 计算的中间产物，不再直接用于二分类
return loss
```
**注意：** 在 `neg_seqs` 的构造上，需要确保它包含的是**被曝光但未点击的物品**。如果你的数据预处理已经能够提供这种类型的负样本，那么这个改变是水到渠成的。否则，你需要修改数据加载部分，使其能够根据 `action_type=0` 的曝光记录来构建负样本。

通过这种列表式的InfoNCE损失，模型被明确地训练去最大化正样本在所有曝光物品中的相对概率，从而直接优化排序目标。

#### 2.3 利用曝光序列进行自监督学习

除了直接修改排序损失，我们还可以将曝光序列作为自监督信号，利用对比学习来增强用户/物品Embedding。

**思路：** 如果用户 $u$ 在一个会话中连续曝光了物品 $i$ 和 $j$，即使 $j$ 没有被点击，但因为它在用户的关注视野中出现，某种程度上它与用户 $u$ 或物品 $i$ 存在一定的关联。我们可以利用这种“上下文相似性”来构建自监督任务。

**例子：曝光序列内的对比**
考虑一个用户 $u$ 的行为序列 $S = [i_1, i_2, \dots, i_L]$，其中每个 $i_t$ 都是一个物品（无论是点击还是曝光未点击）。
我们可以定义：
*   **锚点：** 某个物品 $i_t$ 的Embedding $\mathbf{e}_{i_t}$。
*   **正样本：** 序列中与 $i_t$ 相邻的物品 $i_{t+1}$ 或 $i_{t-1}$ 的Embedding，或者在同一会话中出现的其他物品。
*   **负样本：** 随机采样的其他物品，或来自同一批次中其他序列的物品。

**损失函数：** 同样是InfoNCE损失，目标是让序列内相邻或共现的物品Embedding相互靠近，而与其他不相关物品的Embedding相互远离。

**如何在 `BaselineModel` 中实现：**
这需要在 `forward` 函数之外，或者在 `log2feats` 的某个阶段，引入一个新的损失项。
1.  **构造正样本对：** 对 `log_feats` 中的每个位置 $t$，可以考虑 $i_t$ 和 $i_{t+1}$ 构成正样本对，或者 $i_t$ 和 $i_{\text{sampled from same sequence}}$。
2.  **构造负样本：** 可以使用批次内其他序列的 `log_feats` 作为负样本。
3.  **计算损失：** 调用我们之前实现的 `info_nce_loss` 函数。

**伪代码示例：**
```python
# 在 BaselineModel 的 forward 函数中，在计算 pos_logits, neg_logits 之后
# 或者在一个单独的自监督损失函数中

# 1. 编码用户序列，得到 log_feats: (batch_size, seq_len, hidden_units)
# log_feats = self.log2feats(user_item, mask, seq_feature)

# 2. 构造自监督的正样本对和负样本
# 简单示例：每个位置的item作为query，其下一个item作为positive key
# 假设 log_seqs 是原始ID，mask == 1 的位置是物品
# self_sup_queries = log_feats[:, :-1, :] # 序列中除了最后一个位置的embedding作为query
# self_sup_pos_keys = log_feats[:, 1:, :]  # 序列中除了第一个位置的embedding作为positive key

# 排除padding位置
valid_mask_query = (next_mask[:, :-1] == 1) # 只有item token才能作为query
valid_mask_pos_key = (next_mask[:, 1:] == 1) # 只有item token才能作为positive key
combined_valid_mask = valid_mask_query & valid_mask_pos_key

# 筛选出有效的 query 和 positive_key
# 维度会变为 (num_valid_transitions, hidden_units)
valid_queries = self_sup_queries[combined_valid_mask]
valid_pos_keys = self_sup_pos_keys[combined_valid_mask]

if valid_queries.numel() == 0: # 避免空批次导致错误
    self_supervised_loss = torch.tensor(0.0, device=self.dev)
else:
    # 构造负样本：可以使用当前batch中所有其他非正样本的embedding
    # 也就是整个 batch 的 log_feats (除了query和positive key自身) 作为负样本池
    # 或者简单从整个 item embedding space 随机采样
    
    # 这里的负样本构造可以非常多样，例如：
    # 1. Batch内的其他随机item作为负样本
    # 2. 从全局item space随机采样负样本
    # 3. 困难负样本挖掘：选择与query相近但不是正样本的item
    
    # 简化的负样本构造：使用批次内所有非当前query或positive key的item
    # 这是一个比较通用的做法，不需要额外的负采样机制
    batch_negative_keys = log_feats.view(-1, log_feats.shape[-1]) # Flatten all log_feats in batch
    
    # 确保 query 和 pos_key 不会被作为负样本
    # 实际操作中，F.scaled_dot_product_attention 或 F.cross_entropy 
    # 在计算时，会天然地将 positive_key 排除在负样本之外（因为它是“目标”），
    # 只要负样本池中没有重复的 positive_key 即可。
    
    # 构建 InfoNCE 损失
    # 这里需要确保 valid_queries 和 batch_negative_keys 的形状适配
    # 我们的 info_nce_loss 函数期望 negative_key_embeddings 形状是 (batch_size, num_negative_samples, embedding_dim)
    # 对于自监督任务，可以针对每个 query，将 batch_negative_keys 复制 N_neg 次
    # 或者，将 batch_negative_keys 视为 batch 中所有其他 query 的负样本。
    
    # 我们可以通过调整 info_nce_loss 的输入来适应。
    # 假设我们为每个 valid_query 随机选择 num_negative_samples 个负样本。
    
    # Option A: Simple random sampling from a global pool (less informative)
    # random_neg_items = torch.randint(1, self.item_num + 1, (valid_queries.shape[0], num_negative_samples), device=self.dev)
    # random_neg_embs = self.item_emb(random_neg_items)
    # self_supervised_loss = info_nce_loss(valid_queries, valid_pos_keys, random_neg_embs, temperature=self.temperature_ss)

    # Option B: Batch-in-batch negatives (more common in self-supervision)
    # For each query in valid_queries, its negative keys are all other items in the batch
    # except its corresponding positive key. This is handled naturally by InfoNCE if you
    # concatenate all items in the batch and tell cross_entropy which one is positive.
    
    # Let's align with the info_nce_loss signature
    # For each valid_query, we need to construct a batch of negative samples.
    # A common approach is to use other samples in the current batch as negatives.
    # For simplicity, we can flatten all log_feats and use them as negatives.
    
    # Let's consider a simpler InfoNCE implementation for self-supervised contrastive loss in-batch
    # For each query_i, the positive is pos_i, and negatives are all other (query_j, pos_j) pairs in the batch
    # We take valid_queries as queries and valid_pos_keys as positive_keys
    
    # The InfoNCE needs (query, positive, negative_batch).
    # Here, for each query `q_i`, the positive is `p_i`.
    # The negative keys are `q_j` (j != i) and `p_j` (j != i).
    # A common way is to construct a large 'key' matrix for the batch.
    
    # Let's simplify and use the common "in-batch negative" approach
    # For each (q, p) pair, other (q', p') in the batch are considered negatives.
    # The actual InfoNCE function often takes all `keys` in a batch, then
    # for each query, it knows which key is positive, and all others are negative.
    
    # Let's assume valid_queries are the "anchors" (Q) and valid_pos_keys are "positive_keys" (K+)
    # The "negative_keys" (K-) can be other positive_keys from the same batch (or other queries)
    
    # Re-using the info_nce_loss function
    # It expects: query_embeddings (batch_size, D)
    #            positive_key_embeddings (batch_size, D)
    #            negative_key_embeddings (batch_size, num_negative_samples, D)
    
    # To use in-batch negatives:
    # For each valid_query[i], its positive is valid_pos_keys[i].
    # Its negatives are valid_pos_keys[j] for all j != i.
    
    # Constructing `negative_key_embeddings` for InfoNCE:
    # Need to iterate through `valid_queries` and for each query, pick other valid_pos_keys as negatives.
    
    num_queries = valid_queries.shape[0]
    if num_queries > 1: # 需要至少两个样本才能形成对比
        # 复制 valid_pos_keys 来构造负样本批次
        # 对于每个 query，负样本是除了它自己 positive_key 之外的所有 valid_pos_keys
        # Shape: (num_queries, num_queries - 1, hidden_units)
        temp_neg_keys = []
        for i in range(num_queries):
            # 将除第 i 个 positive_key 外的所有 positive_keys 作为负样本
            other_pos_keys = torch.cat([valid_pos_keys[:i], valid_pos_keys[i+1:]], dim=0)
            temp_neg_keys.append(other_pos_keys.unsqueeze(0))
        
        batch_neg_keys_for_self_sup = torch.cat(temp_neg_keys, dim=0) # Shape: (num_queries, num_queries - 1, hidden_units)
        
        # Ensure embeddings are normalized (if not already by log2feats)
        valid_queries = F.normalize(valid_queries, p=2, dim=-1)
        valid_pos_keys = F.normalize(valid_pos_keys, p=2, dim=-1)
        batch_neg_keys_for_self_sup = F.normalize(batch_neg_keys_for_self_sup, p=2, dim=-1)
        
        self_supervised_loss = info_nce_loss(valid_queries, valid_pos_keys, batch_neg_keys_for_self_sup, temperature=0.1)
    else:
        self_supervised_loss = torch.tensor(0.0, device=self.dev)

# 最终的总损失可以是在推荐任务损失和自监督损失之间的加权和
# total_loss = recommendation_task_loss + alpha * self_supervised_loss
```
通过这种方式，即使没有显式的点击标签，模型也能从行为序列的结构中学习到有用的语义信息，增强Embedding的鲁棒性和判别力，这有助于NDCG的提升。

#### 2.4 温度系数 $\tau$ 的重要性

在InfoNCE损失中，温度系数 $\tau$ 是一个至关重要的超参数。
*   **小 $\tau$：** 使得 Softmax 分布更“尖锐”，高相似度的样本概率被进一步放大，低相似度的样本概率被进一步缩小。这会使得模型更专注于区分**困难负样本**，因为它对错误分类的惩罚更强。
*   **大 $\tau$：** 使得 Softmax 分布更“平滑”，所有样本的概率差异缩小。这会使得模型更倾向于学习一个广泛的相似性度量，对困难负样本的关注度降低。

调节 $\tau$ 的值通常需要进行实验。较小的 $\tau$ 值（如0.07-0.1）在许多对比学习任务中被证明是有效的，因为它鼓励模型学习更具判别力的特征。

---

### 优化输入特征融合机制


特征交互是深度学习推荐系统中一个关键的研究方向，旨在显式地建模不同特征之间的高阶、非线性交叉关系。以下是一些可以考虑的优化方向：

#### 3.1 引入特征交互网络 (Feature Interaction Networks)

不再是简单的拼接后线性变换，而是引入专门的模块来学习特征之间的交互。

**3.1.1 Factorization Machines (FM) 或 Field-aware Factorization Machines (FFM) 思想**

虽然FM/FFM通常用于浅层模型，但其**二阶交叉**的思想可以启发我们。它假设每个特征有一个隐向量，并通过隐向量的内积来捕捉特征间的二阶交互。

**如何在现有模型中应用其思想：**
在 `feat2emb` 中，在 `torch.cat` 之前或之后，可以添加一个模块来显式计算特征交叉。

例如，在 `feat2emb` 中的 `all_item_emb` (和 `all_user_emb`) 在经过 `torch.cat` 之后，但在 `itemdnn` (和 `userdnn`) 之前，可以引入一个或多个交互层。

假设拼接后的 `all_item_emb` 形状为 `[batch_size, seq_len, total_feature_dim]`。我们可以将其拆解回原始的各个特征Embedding，然后计算它们两两之间的内积或外积。

**示例：二阶点积交互层**
假设我们有 $N$ 个不同类型的特征 Embedding $e_1, e_2, \dots, e_N$。
我们可以计算所有特征对的内积：

$$
\text{Interaction}_{ij} = \mathbf{e}_i^\top \mathbf{e}_j
$$

然后将这些交互项与原始特征一起输入到 `itemdnn`。

这需要在 `feat2emb` 内部进行更精细的控制，因为它现在直接拼接了。
**修改思路：**
1.  **保存独立的特征Embedding：** 在 `feat2emb` 中，不要立即 `cat` 所有 `item_feat_list` 和 `user_feat_list`。
2.  **构建特征交互层：**
    ```python
    # 在feat2emb的末尾，item_feat_list 和 user_feat_list 已经包含了各种特征的Embedding
    # 假设 item_feat_list = [item_embedding, sparse1_emb, array1_emb, ...]
    
    # 转换为一个 (num_features, batch_size, seq_len, hidden_units) 的张量列表
    # (或者更简单地，在拼接前，对每个特征embedding进行 reshape 以便后续操作)
    
    # 以下为概念伪代码，实际实现需要根据特征数量和维度进行调整
    all_item_embeddings_list = item_feat_list # 假设每个元素都是 (B, L, D)
    
    interactive_features = []
    # 遍历所有特征对，计算点积交互
    for i in range(len(all_item_embeddings_list)):
        for j in range(i + 1, len(all_item_embeddings_list)):
            # 逐元素相乘后求和得到点积
            # 形状 (B, L, D) * (B, L, D) -> (B, L, D) -> sum(dim=-1) -> (B, L)
            # 或者更复杂的方式，比如外积再卷积
            interaction = (all_item_embeddings_list[i] * all_item_embeddings_list[j]).sum(dim=-1, keepdim=True)
            interactive_features.append(interaction) # 形状 (B, L, 1)
            
    # 拼接原始特征和交互特征
    # 首先将原始特征展平，或直接使用原始cat的结果
    original_concat_emb = torch.cat(item_feat_list, dim=-1) # (B, L, D_total)
    
    # 拼接交互特征 (需要先调整维度，使其与 original_concat_emb 在 seq_len 维度对齐)
    if interactive_features:
        concat_interactive = torch.cat(interactive_features, dim=-1) # (B, L, num_interactions)
        final_fusion_input = torch.cat([original_concat_emb, concat_interactive], dim=-1)
    else:
        final_fusion_input = original_concat_emb
        
    # 然后再送入 self.itemdnn(final_fusion_input)
    ```
这种方法会增加特征维度，并引入更多的非线性。

**3.1.2 Deep & Cross Network (DCN) 或 FiBiNET 等**

更复杂的特征交互网络，如DCN通过残差连接和交叉网络层显式地学习高阶交叉特征。FiBiNET则通过SENet（Squeeze-and-Excitation Network）的思想为特征动态加权，并进行双线性特征交互。

这些模型通常是在Embedding层之上，Transformer编码器之前，或者在Transformer编码器内部的FFN层中集成。

**修改思路：**
在 `BaselineModel` 的 `__init__` 中定义一个 `FeatureInteraction` 模块，然后在 `log2feats` 的 `seqs = self.feat2emb(...)` 之后，在进入Transformer层之前，将 `seqs` 传入这个特征交互模块。
```python
class FeatureInteractionLayer(nn.Module):
    def __init__(self, hidden_units, num_features):
        super().__init__()
        # 简化版，例如一个简单的多层感知机来学习交互
        # 实际可以是DCN, FiBiNET等复杂结构
        self.interaction_mlp = nn.Sequential(
            nn.Linear(num_features * hidden_units, hidden_units * 2),
            nn.ReLU(),
            nn.Linear(hidden_units * 2, hidden_units)
        )
        # 或者更精细的二阶交叉
        self.cross_net = ... # 交叉网络层，如DCN中的CrossNet

    def forward(self, embeddings_list): # embeddings_list 是一个列表，包含每个特征的embedding
        # embeddings_list: [e_user_id, e_item_id, e_sparse1, e_array1, ...]
        # 假设每个 e_i 形状是 (B, L, D)
        
        # 简单拼接后输入MLP (忽略了不同特征的交互，但能学到非线性)
        # combined_emb = torch.cat(embeddings_list, dim=-1) # (B, L, sum(D_i))
        # return self.interaction_mlp(combined_emb) # (B, L, hidden_units)

        # 或者显式特征交互 (例如外积后卷积，或DCN的cross_net)
        # 需要更复杂的实现，可能需要传入原始的 item_feat_list 来进行更细粒度的交互
        
        # 针对BaselineModel的feat2emb输出
        # feat2emb 已经做了初步拼接和itemdnn/userdnn融合
        # 如果要在这里加交互，需要将feat2emb内部的itemdnn/userdnn去掉
        # 然后将原始的 item_feat_list 和 user_feat_list 传递出来
        # 在 log2feats 中，接收这些列表，然后在这个 FeatureInteractionLayer 中进行融合
        pass
```
这需要对 `feat2emb` 进行较大改动，使其不再直接 `cat` 并降维，而是返回一个包含所有独立特征Embedding的列表，然后在 `log2feats` 或单独的 `FeatureFusionLayer` 中进行特征交互。

#### 3.2 门控机制 (Gating Mechanism)

门控机制可以学习不同特征的重要性，并动态地调整它们对最终表示的贡献。例如，在用户年轻时，年龄特征可能不那么重要，但随着年龄增长，其重要性可能上升。

**思路：** 为每个特征Embedding分配一个学习到的门控权重。

$$
\mathbf{z} = \sigma(\mathbf{W}_g \cdot [\mathbf{e}_1; \mathbf{e}_2; \dots; \mathbf{e}_N] + \mathbf{b}_g) \\
\mathbf{E}_{\text{fused}} = \sum_{k=1}^N \mathbf{z}_k \odot \mathbf{e}_k
$$

其中 $\sigma$ 是Sigmoid激活函数，$\odot$ 是元素级乘法。

**修改思路：**
可以在 `feat2emb` 的 `torch.cat` 之后，`itemdnn`/`userdnn` 之前，添加一个门控网络。

```python
# 在 feat2emb 中，假设 all_item_emb 是 cat 拼接后的结果 (B, L, total_feature_dim)
# all_item_emb = torch.cat(item_feat_list, dim=2)

# 门控网络
# input_dim = total_feature_dim
# gate_mlp = torch.nn.Sequential(
#     torch.nn.Linear(input_dim, input_dim),
#     torch.nn.Sigmoid()
# )
# gate_weights = gate_mlp(all_item_emb) # Shape: (B, L, total_feature_dim)

# 逐元素相乘，然后可以再通过 itemdnn
# gated_item_emb = all_item_emb * gate_weights
# all_item_emb = torch.relu(self.itemdnn(gated_item_emb))
```
这种方式允许模型学习到每个特征在不同上下文下的动态重要性。

#### 3.3 利用Transformer的自注意力进行特征融合

`BaselineModel` 的 `log2feats` 已经使用了Transformer来处理序列。我们也可以将不同的特征（例如，物品ID Embedding、物品稀疏特征Embedding、物品多模态特征Embedding等）视为序列中的“token”，然后让Transformer的自注意力层来学习它们之间的交互。

**思路：**
1.  对于每个序列位置的物品，将它的各种特征Embedding（例如，物品ID Embedding `e_id`、类别Embedding `e_cat`、品牌Embedding `e_brand`、图片Embedding `e_img`）拼接起来，形成一个“特征序列”。
2.  将这个“特征序列”输入到一个额外的、轻量级的Transformer层（或多头注意力层）。让其自注意力机制学习不同特征之间的交互。
3.  输出这个特征Transformer的 `[CLS]` token 或者所有特征的池化结果，作为该物品的最终Embedding。

**修改思路：**
在 `feat2emb` 的 `item_feat_list` 和 `user_feat_list` 内部：
```python
class FeatureTransformer(nn.Module):
    def __init__(self, hidden_units, num_heads, dropout_rate):
        super().__init__()
        self.attn = FlashMultiHeadAttention(hidden_units, num_heads, dropout_rate)
        self.ffn = PointWiseFeedForward(hidden_units, dropout_rate)
        self.norm1 = nn.LayerNorm(hidden_units)
        self.norm2 = nn.LayerNorm(hidden_units)
        # 假设所有特征Embedding已经对齐到 hidden_units 维度

    def forward(self, features_list_for_one_item_in_seq):
        # features_list_for_one_item_in_seq: [tensor_item_id, tensor_sparse1, tensor_array1, tensor_emb1]
        # 形状都是 (B, 1, D)
        # 拼接成 (B, num_item_features, D)
        features_seq = torch.cat(features_list_for_one_item_in_seq, dim=1) # dim=1 是特征数量维度
        
        # Transformer Layer
        x = self.norm1(features_seq)
        attn_out, _ = self.attn(x, x, x)
        x = x + attn_out

        x = self.norm2(x)
        x = x + self.ffn(x)

        # 可以取第一个 token (e.g., item ID embedding) 作为代表，或进行 pooling
        return x[:, 0, :] # 返回第一个特征的表示，或 torch.mean(x, dim=1)
```
然后，在 `feat2emb` 中，替换掉 `torch.cat(item_feat_list, dim=2)` 和 `itemdnn`，将其替换为这个 `FeatureTransformer`。这会显著增加模型的复杂度，但也能捕获更复杂的特征交互。

---

### 序列推荐场景下的重要新特征


在现有模型的基础上，我们可以加入更多对序列推荐有价值的特征。`BaselineModel` 已经处理了用户/物品ID、位置编码和多种辅助特征，但仍有一些对用户兴趣漂移、上下文偏好等关键信号有帮助的特征未被显式建模。

重要的特征通常来源于对用户行为的深入理解和对推荐场景的洞察。

#### 4.1 时序特征

用户行为序列具有强烈的时间属性，目前模型只用了简单的位置编码。更精细的时间特征可以捕捉用户兴趣的动态变化和周期性。

1.  **交互时间间隔 (Time Interval):**
    *   **特征：** 用户两次连续交互之间的时间间隔（例如，秒、小时、天）。
    *   **重要性：** 短时间间隔可能表示用户当前对某个话题兴趣浓厚；长时间间隔可能表示兴趣发生漂移。对于“曝光未点击”的序列，两次曝光之间的时间间隔也很有意义。
    *   **构造：** 在数据预处理阶段，计算 `timestamp` 字段的差值。
    *   **融入：** 作为连续特征，在 `feat2emb` 中像 `ITEM_CONTINUAL_FEAT` 一样处理，然后与其他Embedding拼接。

2.  **绝对时间特征 (Absolute Time Features):**
    *   **特征：** 交互发生的具体时间（如星期几、一天中的小时、月份、季节）。
    *   **重要性：** 捕捉用户行为的周期性。例如，用户可能在工作日晚上偏好新闻，周末白天偏好娱乐。
    *   **构造：** 从 `timestamp` 字段中提取。
    *   **融入：** 作为稀疏特征（如星期几、小时）或连续特征（如小时的sin/cos变换，用于处理周期性）。

#### 4.2 用户侧动态特征 (User Dynamic Features)

除了静态用户画像，用户兴趣是动态变化的。

1.  **用户兴趣漂移特征：**
    *   **特征：** 基于用户最近 $K$ 次交互的物品类别、标签或品牌分布，与用户历史整体兴趣分布的差异。
    *   **重要性：** 捕捉用户短期兴趣的变化。
    *   **构造：** 需要在数据处理时，动态计算每个用户在当前时间点前的短期兴趣分布。
    *   **融入：** 可以是Embedding（例如，将兴趣分布编码为一个向量），或作为额外的一个连续特征向量（例如，短时兴趣Embedding与长时兴趣Embedding的距离）。

2.  **用户活跃度特征：**
    *   **特征：** 用户最近 $N$ 天的交互次数、登录频率等。
    *   **重要性：** 活跃用户可能更容易被推荐成功，他们的兴趣表达更强烈。
    *   **构造：** 基于用户历史行为统计。
    *   **融入：** 作为连续特征。

#### 4.3 物品侧动态特征 (Item Dynamic Features)

物品的属性也可能随时间变化，或者具有一定的时效性。

1.  **物品流行度 / 新鲜度 (Popularity / Freshness):**
    *   **特征：** 物品的总点击/曝光次数、最近 $N$ 天的点击/曝光次数、物品上架时间距今的时间。
    *   **重要性：** 新鲜的物品或热门的物品更容易被关注。但也要注意长尾效应。
    *   **构造：** 实时或离线统计。
    *   **融入：** 作为连续特征。

2.  **物品生命周期阶段：**
    *   **特征：** 物品是否处于新品期、成熟期、衰退期等。
    *   **重要性：** 不同阶段的物品推荐策略可能不同。
    *   **构造：** 基于物品上架时间、销量变化趋势等。
    *   **融入：** 作为稀疏特征。

#### 4.4 交叉特征 (Interaction Features)

高阶交叉特征可以捕捉更复杂的模式。

1.  **用户-物品交叉：**
    *   **特征：** 用户对某个物品类别是否偏好、用户是否是某个品牌的忠实用户、用户是否喜欢某个特定标签的物品。
    *   **重要性：** 更个性化的偏好。
    *   **构造：** 可以通过预先计算，或者让模型自动学习（如第三章提到的特征交互网络）。
    *   **融入：** 如果是预计算的，可以是稀疏特征（例如，`User_Loves_Category_X` 为 True/False）。

2.  **物品-物品交叉：**
    *   **特征：** 两个物品是否经常被同一批用户交互、两个物品是否属于同一品类下的子类别。
    *   **重要性：** 协同过滤的体现，有助于发现关联商品。
    *   **构造：** 可以在 `log_feats` 序列中，利用自注意力机制，自然地学习到序列中物品间的隐式交叉。但也可以显式地构建。

#### 4.5 会话级特征 (Session-level Features)

如果用户的行为可以被划分为独立的会话。

1.  **会话长度：**
    *   **特征：** 当前用户会话中包含的物品数量。
    *   **重要性：** 长会话可能表示用户探索性更强，短会话可能表示目标性更强。
    *   **构造：** 在会话划分后计算。
    *   **融入：** 作为连续特征。

2.  **会话多样性：**
    *   **特征：** 会话中物品类别、品牌的丰富度。
    *   **重要性：** 捕捉用户兴趣的广度。
    *   **构造：** 基于会话内物品的特征统计。
    *   **融入：** 作为连续特征。

#### 4.6 融入新特征的步骤

将这些新特征融入 `BaselineModel` 的通用流程是：

1.  **数据预处理：** 在 `Dataset` 或 `Dataloader` 层面，根据原始数据（`seq.jsonl`，`item_feat_dict.json`）计算和提取这些新的特征，并将它们添加到 `seq_feature` 或 `item_feature` / `user_feature` 的字典中。
2.  **更新特征统计：** 如果是新的类别特征，需要更新 `feat_statistics` 来获取其基数。
3.  **更新特征类型映射：** 在 `_init_feat_info` 方法中，更新 `feat_types` 字典，将新特征 ID 归类到 `USER_CONTINUAL_FEAT`, `ITEM_SPARSE_FEAT` 等现有或新增的特征类型列表中。
4.  **模型初始化：** 在 `BaselineModel.__init__` 中，根据新特征的类型，在 `self.sparse_emb`, `self.emb_transform` 等模块中初始化对应的Embedding层或线性变换层。
5.  **特征嵌入：** 在 `feat2emb` 方法中，添加对新特征的处理逻辑。根据其类型（稀疏、数组、连续），调用对应的Embedding或直接拼接。
6.  **维度调整：** 由于新增特征会增加 `userdim` 和 `itemdim`，`self.userdnn` 和 `self.itemdnn` 的输入维度需要相应调整。
