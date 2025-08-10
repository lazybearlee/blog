---
title: 腾讯广告大赛2025-基于BPRLoss以及InbatchLoss产生的思考
date: 2025-08-10
slug: blog-post-slug
tags:
  - 推荐系统
  - 腾讯广告大赛2025
  - 生成式推荐
categories:
  - 笔记
description: 描述
draft: true
state: "0"
---


## 当前项目分析

### 1. Baseline模型架构分析
当前的Baseline是一个基于Transformer的推荐系统，主要包含以下组件：
- **特征处理与Embedding层**：处理用户ID、物品ID、稀疏特征、数组特征、多模态特征和连续特征
- **Transformer编码器**：使用Flash Attention优化的多头注意力机制
- **特征融合DNN层**：通过全连接网络融合用户和物品特征
- **预测层**：计算用户序列与正负样本之间的相似度

### 2. 当前损失函数分析
Baseline使用的是简单的BCEWithLogitsLoss，仅针对物品样本计算损失：
```python
indices = np.where(next_token_type == 1)
loss = bce_criterion(pos_logits[indices], pos_labels[indices])
loss += bce_criterion(neg_logits[indices], neg_labels[indices])
```

### 3. 实验结果分析
- Baseline得分：0.03左右
- BPR损失+随机负采样：0.023左右
- Inbatch交叉熵损失（带温度系数和正则化）：0.023左右

这些结果表明您尝试的优化方法反而降低了模型性能，这可能是因为：

1. **负采样策略问题**：当前每个正样本只对应一个负样本，可能不足以提供充分的对比信号
2. **Inbatch交叉熵实现问题**：仅使用batch内样本作为负样本，多样性不足
3. **模型容量限制**：当前模型参数较少（hidden_units=32, num_blocks=1, num_heads=1）
4. **特征利用不充分**：多模态特征可能没有被充分利用

## 优化建议

### 1. 改进负采样策略

当前每个正样本只使用一个负样本，这限制了模型学习区分能力。建议：

```python
# 增加负样本数量
def __getitem__(self, uid):
    # ... 现有代码 ...
    
    # 修改负采样部分，增加负样本数量
    neg_samples = []
    for _ in range(args.num_neg_samples):  # 新增参数，控制负样本数量
        neg_id = self._random_neq(1, self.itemnum + 1, ts)
        neg_samples.append(neg_id)
    
    # 返回多个负样本
    return seq, pos, neg_samples, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat
```

### 2. 混合负采样策略

结合不同类型的负样本，提高负样本的多样性和质量：

```python
def mixed_negative_sampling(self, pos_item, user_history, item_num, num_neg_samples):
    neg_samples = []
    
    # 1. 随机负采样 (50%)
    for _ in range(num_neg_samples // 2):
        neg_id = self._random_neq(1, item_num + 1, user_history)
        neg_samples.append(neg_id)
    
    # 2. 难例负采样 (50%) - 基于相似度的难例
    # 这里需要预先计算或加载物品相似度矩阵
    for _ in range(num_neg_samples - num_neg_samples // 2):
        # 获取与正样本相似但用户未交互的物品
        similar_items = self.get_similar_items(pos_item, k=10)
        for item in similar_items:
            if item not in user_history:
                neg_samples.append(item)
                break
        else:
            # 如果没有找到合适的难例，使用随机负样本
            neg_id = self._random_neq(1, item_num + 1, user_history)
            neg_samples.append(neg_id)
    
    return neg_samples
```

### 3. 改进损失函数

#### 3.1 加权BCE损失
```python
def weighted_bce_loss(pos_logits, neg_logits, pos_weights=None, neg_weights=None):
    """
    加权BCE损失，给难样本更高权重
    """
    if pos_weights is None:
        pos_weights = torch.ones_like(pos_logits)
    if neg_weights is None:
        neg_weights = torch.ones_like(neg_logits)
    
    pos_loss = F.binary_cross_entropy_with_logits(
        pos_logits, torch.ones_like(pos_logits), reduction='none'
    )
    neg_loss = F.binary_cross_entropy_with_logits(
        neg_logits, torch.zeros_like(neg_logits), reduction='none'
    )
    
    # 应用权重
    weighted_pos_loss = (pos_loss * pos_weights).mean()
    weighted_neg_loss = (neg_loss * neg_weights).mean()
    
    return weighted_pos_loss + weighted_neg_loss
```

#### 3.2 结合BCE和Triplet Loss
```python
def combined_loss(pos_logits, neg_logits, user_emb, pos_emb, neg_embs, margin=0.2):
    """
    结合BCE损失和Triplet Loss
    """
    # BCE损失部分
    bce_loss = F.binary_cross_entropy_with_logits(
        pos_logits, torch.ones_like(pos_logits)
    ) + F.binary_cross_entropy_with_logits(
        neg_logits, torch.zeros_like(neg_logits)
    )
    
    # Triplet Loss部分
    triplet_loss = 0
    for neg_emb in neg_embs:
        # 计算距离
        pos_dist = F.pairwise_distance(user_emb, pos_emb, p=2)
        neg_dist = F.pairwise_distance(user_emb, neg_emb, p=2)
        
        # Triplet Loss
        triplet_loss += F.relu(pos_dist - neg_dist + margin)
    
    triplet_loss /= len(neg_embs)
    
    # 组合损失
    total_loss = bce_loss + 0.1 * triplet_loss  # 调整权重
    
    return total_loss
```

### 4. 改进Inbatch交叉熵损失

```python
def inbatch_cross_entropy_loss(user_emb, pos_emb, neg_embs, temperature=0.05):
    """
    改进的Inbatch交叉熵损失
    结合batch内负样本和全局负样本
    """
    batch_size = user_emb.size(0)
    
    # 1. 计算正样本相似度
    pos_sim = torch.cosine_similarity(user_emb, pos_emb, dim=1)
    
    # 2. 计算batch内负样本相似度
    batch_neg_sim = torch.matmul(user_emb, pos_emb.t())  # [batch_size, batch_size]
    # 排除自身（对角线）
    mask = torch.eye(batch_size, device=user_emb.device).bool()
    batch_neg_sim = batch_neg_sim.masked_fill(mask, -1e9)
    
    # 3. 计算全局负样本相似度
    global_neg_sim = []
    for i in range(batch_size):
        # 计算与全局负样本的相似度
        neg_sim = torch.matmul(user_emb[i:i+1], neg_embs[i].t())  # [1, num_neg]
        global_neg_sim.append(neg_sim)
    global_neg_sim = torch.cat(global_neg_sim, dim=0)  # [batch_size, num_neg]
    
    # 4. 合并所有负样本相似度
    all_neg_sim = torch.cat([batch_neg_sim, global_neg_sim], dim=1)  # [batch_size, batch_size-1+num_neg]
    
    # 5. 应用温度系数
    pos_sim = pos_sim / temperature
    all_neg_sim = all_neg_sim / temperature
    
    # 6. 计算交叉熵损失
    logits = torch.cat([pos_sim.unsqueeze(1), all_neg_sim], dim=1)  # [batch_size, 1+batch_size-1+num_neg]
    labels = torch.zeros(batch_size, dtype=torch.long, device=user_emb.device)
    
    loss = F.cross_entropy(logits, labels)
    
    return loss
```

### 5. 模型架构优化

#### 5.1 增加模型容量
```python
# 在get_args函数中增加模型容量
parser.add_argument('--hidden_units', default=128, type=int, help='隐藏层维度，影响模型容量')
parser.add_argument('--num_blocks', default=3, type=int, help='Transformer块数量，影响模型深度')
parser.add_argument('--num_heads', default=4, type=int, help='注意力头数量，影响模型表达能力')
```

#### 5.2 添加特征交叉层
```python
class FeatureCrossLayer(torch.nn.Module):
    """
    特征交叉层，用于捕捉特征之间的交互
    """
    def __init__(self, hidden_units):
        super(FeatureCrossLayer, self).__init__()
        self.cross_layer = torch.nn.Linear(hidden_units, hidden_units)
        self.activation = torch.nn.ReLU()
        
    def forward(self, x):
        # x shape: [batch_size, seq_len, hidden_units]
        batch_size, seq_len, hidden_units = x.size()
        
        # 特征交叉
        cross_features = []
        for i in range(seq_len):
            for j in range(i+1, seq_len):
                # 计算特征交叉
                cross = x[:, i] * x[:, j]  # [batch_size, hidden_units]
                cross_features.append(cross)
        
        # 聚合交叉特征
        cross_features = torch.stack(cross_features, dim=1)  # [batch_size, num_crosses, hidden_units]
        cross_features = self.cross_layer(cross_features)
        cross_features = self.activation(cross_features)
        
        # 平均池化
        cross_features = torch.mean(cross_features, dim=1)  # [batch_size, hidden_units]
        
        return cross_features
```

### 6. 针对评估指标的优化

由于评估指标是0.31HitRate@10 + 0.69NDCG@10，我们可以直接优化这两个指标：

```python
def listwise_loss(user_emb, pos_emb, neg_embs, k=10):
    """
    针对排序指标的Listwise损失函数
    直接优化HitRate@K和NDCG@K
    """
    batch_size = user_emb.size(0)
    
    # 计算所有候选的得分
    all_scores = []
    all_labels = []
    
    for i in range(batch_size):
        # 正样本得分
        pos_score = torch.cosine_similarity(user_emb[i:i+1], pos_emb[i:i+1])
        
        # 负样本得分
        neg_scores = torch.cosine_similarity(user_emb[i:i+1], neg_embs[i])
        
        # 合并所有得分
        scores = torch.cat([pos_score, neg_scores])
        labels = torch.cat([torch.ones(1), torch.zeros(len(neg_scores))])
        
        all_scores.append(scores)
        all_labels.append(labels)
    
    # 计算ListNet损失
    total_loss = 0
    for scores, labels in zip(all_scores, all_labels):
        # 计算Top-K概率分布
        topk_scores, topk_indices = torch.topk(scores, k)
        topk_labels = labels[topk_indices]
        
        # 计算预测概率和真实概率
        pred_probs = F.softmax(topk_scores, dim=0)
        true_probs = topk_labels / topk_labels.sum()
        
        # 计算KL散度
        loss = F.kl_div(torch.log(pred_probs), true_probs, reduction='sum')
        total_loss += loss
    
    return total_loss / batch_size
```
