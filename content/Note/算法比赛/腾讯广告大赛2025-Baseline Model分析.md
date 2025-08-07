---
title: 腾讯广告大赛2025-Baseline Model分析
date: 2025-08-06
slug: blog-post-slug
tags:
  - 推荐系统
  - 广告推荐
  - 生成式推荐
  - 腾讯广告大赛2025
categories:
  - 分类
description: 描述
draft: true
state: "0"
---

### 模型整体架构

官方提供的 `BaselineModel` 类是一个基于Transformer架构的推荐系统基线模型。它的设计目标是处理用户在广告或商品上的行为序列，并结合多种丰富的辅助特征（如多模态特征、稀疏特征、连续特征等），最终为用户提供个性化推荐。

#### 1.1 核心定位

该模型试图解决以下推荐系统中的常见挑战：
*   **序列建模：** 用户行为（如点击、曝光序列）具有时间依赖性，Transformer能够有效捕获序列中的长距离依赖关系。
*   **多源特征融合：** 真实世界的推荐系统需要处理用户和物品的多种类型特征，模型需要一个统一的框架来整合这些异构信息。
*   **高效训练与推理：** 利用PyTorch 2.0+的Flash Attention等优化技术，提升模型在长序列和大规模数据集上的计算效率。

#### 1.2 整体架构概览

`BaselineModel` 可以被看作是一个复杂的编码器，它接收用户行为序列和相关特征作为输入，输出高质量的用户和物品表征（Embedding）。其核心组件可以分为以下几个功能模块：

1.  **特征处理与Embedding层 (`_init_feat_info`, `feat2tensor`, `feat2emb`)：**
    *   负责解析和管理模型所需的各种特征（用户ID、物品ID、稀疏特征、数组特征、多模态特征、连续特征）。
    *   将这些不同类型、不同格式的原始特征转换为统一的、固定维度的稠密向量（Embedding），并进行初步的特征融合。

2.  **Transformer编码器 (`FlashMultiHeadAttention`, `PointWiseFeedForward`, `log2feats`)：**
    *   这是模型的核心序列建模部分。它由多层Transformer块组成，每个块包含一个多头注意力层和一个逐点前馈网络。
    *   通过自注意力机制，捕获用户行为序列中物品之间的复杂依赖关系和用户兴趣的演变。
    *   特别地，它采用了**Flash Attention**这一优化技术，以提高计算效率和内存利用率。

3.  **特征融合DNN层 (`userdnn`, `itemdnn`)：**
    *   在将所有特征Embedding化之后，模型使用两层简单的全连接网络（DNN）来进一步融合用户和物品的各种特征表示，将它们压缩到 `hidden_units` 维度。

4.  **预测层 (`forward`)：**
    *   在训练阶段，利用Transformer编码器输出的序列特征和正负样本的特征，计算它们之间的相似度（Logits）。这些Logits将作为损失函数的输入。

5.  **推理/服务层 (`predict`, `save_item_emb`)：**
    *   在推理阶段，模型能够提取用户序列的最终表征，用于实时推荐或离线候选生成。
    *   还可以批量生成所有物品的Embedding，用于构建向量检索系统，实现高效的Top-K推荐。

#### 1.3 数据流向与维度变化

一个典型的数据流向大致如下：
*   **原始数据输入：** `user_item` (用户行为序列ID)、`seq_feature` (序列中每个行为对应的丰富特征，如物品类别、标签等)、`pos_seqs` (正样本ID)、`neg_seqs` (负样本ID) 等。
*   **特征处理 (`feat2tensor` -> `feat2emb`)：**
    *   原始 ID 或特征值 -> 各种Embedding表或线性变换层 -> `[batch_size, seq_len, hidden_units]` (对于序列特征) 或 `[batch_size, hidden_units]` (对于单个用户/物品特征)。
    *   多种特征的Embedding会被拼接起来 (`torch.cat`)。
*   **Transformer编码器 (`log2feats`)：**
    *   拼接后的序列特征 `[batch_size, seq_len, total_feature_dim]` （经过特征融合DNN后变为 `[batch_size, seq_len, hidden_units]`）
    *   Transformer处理后，输出`log_feats`，形状仍为 `[batch_size, seq_len, hidden_units]`，但其中的每个向量都编码了该位置及其之前所有位置的信息。
*   **预测 (`forward`)：**
    *   `log_feats` `[batch_size, seq_len, hidden_units]`
    *   `pos_embs` `[batch_size, seq_len, hidden_units]`
    *   `neg_embs` `[batch_size, seq_len, hidden_units]` (通常是多个负样本，可能维度为 `[batch_size, seq_len, num_neg, hidden_units]`)
    *   通过点积计算相似度：`pos_logits`, `neg_logits` 形状为 `[batch_size, seq_len]` (每个序列位置一个正负样本对的得分)。

通过这种分层的设计，模型能够灵活地处理复杂的用户-物品交互数据，并利用Transformer的强大序列建模能力来捕获用户兴趣的动态变化。在接下来的章节中，我们将更深入地剖析每个模块的细节和其背后的原理。

---

### 特征工程与Embedding层

在推荐系统中，特征工程的重要性不言而喻。原始的用户ID、物品ID、甚至文本图片等多模态数据，都需要被有效地转换为模型可以理解的数值表示。这个模型在特征处理上做得非常细致和全面。

#### 2.1 特征分类与初始化 (`_init_feat_info`)

`_init_feat_info` 方法是模型特征处理的起点，它负责解析输入的 `feat_statistics` (特征的基数，即有多少个不同的取值) 和 `feat_types` (特征的类型和对应的特征ID列表)，然后将它们组织成模型内部易于访问的字典结构。

模型将特征清晰地分为四种类型：

1.  **稀疏特征 (Sparse Features):**
    *   `USER_SPARSE_FEAT` 和 `ITEM_SPARSE_FEAT`。
    *   这类特征通常是**单值类别特征**，例如用户性别、物品品牌、物品类别ID等。
    *   对于这类特征，我们通常使用**Embedding查找表**将其映射为稠密的向量。每个独立的特征值（如男性、女性）对应Embedding表中的一个唯一向量。

2.  **数组特征 (Array Features):**
    *   `USER_ARRAY_FEAT` 和 `ITEM_ARRAY_FEAT`。
    *   这类特征通常是**多值类别特征**，例如用户拥有的标签列表、物品的关键词列表、电影的演员列表等。一个用户或物品可能对应多个此类特征值。
    *   处理方式通常是将每个值进行Embedding查找，然后将这些Embedding向量**求和或求平均**，得到一个代表整个数组的稠密向量。

3.  **多模态特征 (Multi-modal Embedding Features):**
    *   `ITEM_EMB_FEAT`。
    *   这类特征是**预训练的稠密向量**，例如物品图片的ResNet特征、文本的BERT特征、视频的C3D特征等。它们通常是高维向量，直接代表了丰富的语义信息。
    *   模型通过一个**线性变换层 (`torch.nn.Linear`)** 将这些高维多模态特征进一步映射到模型的 `hidden_units` 维度，以便与其他特征进行融合。代码中 `EMB_SHAPE_DICT` 定义了这些预训练特征的原始维度（如特征82是1024维，特征84是4096维等），这表明模型确实设计来处理非常丰富的多媒体内容特征。

4.  **连续特征 (Continual Features):**
    *   `USER_CONTINUAL_FEAT` 和 `ITEM_CONTINUAL_FEAT`。
    *   这类特征是**数值型特征**，例如用户年龄、物品价格、评分等。
    *   它们通常不需要Embedding查找，而是直接作为数值输入模型。在特征融合时，通常会将其与其他Embedding向量拼接起来。

#### 2.2 特征张量化 (`feat2tensor`)

`feat2tensor` 它解决了将变长的特征序列和多值的数组特征转换为固定形状张量的问题，这通常通过**Padding**来实现。

*   **输入形式：** `seq_feature` 是一个 `[batch_size, maxlen]` 的列表，其中每个元素是一个字典 `{feature_id: feature_value}`。这意味着每个用户序列中的每个行为（物品）都带有一组对应的特征。

*   **处理逻辑：**
    *   **稀疏特征 (Sparse):** 对于单值类别特征，只需要在序列长度维度上进行Padding。例如，如果 `maxlen` 是50，而某个用户的行为序列只有30个物品，那么后20个位置就会用0进行填充。输出形状 `[batch_size, max_seq_len]`。
    *   **数组特征 (Array):** 这是更复杂的情况，因为既有序列长度的变长，又有每个数组内部的变长（例如，一个电影有3个演员，另一个有5个演员）。
        *   代码通过**二级Padding**来处理：首先找到批次内最大的序列长度 `max_seq_len`，再找到批次内所有数组特征的最大数组长度 `max_array_len`。
        *   然后预分配一个 `[batch_size, max_seq_len, max_array_len]` 的NumPy数组。
        *   最后，将实际数据填充进去，不足的部分用0填充。
        *   这种方式确保了所有数组特征都能被整齐地放置在固定维度的张量中，便于后续的批量Embedding查找。

*   **优化考虑：** 使用NumPy预分配内存和批量操作，避免Python循环中频繁的张量创建和内存拷贝，提高效率。

#### 2.3 特征嵌入与融合 (`feat2emb`)

`feat2emb` 方法是特征处理的核心流水线，它将经过张量化的各种特征最终转换为统一的 `hidden_units` 维度的Embedding，并进行初步融合。

*   **ID特征Embedding：**
    *   用户ID (`self.user_emb`) 和物品ID (`self.item_emb`) 是最基本的Embedding。`padding_idx=0` 意味着ID为0的Embedding会被固定为0向量，通常用于填充（padding）。
    *   `user_mask` 和 `item_mask` 的使用非常巧妙，它根据 `mask`（指示当前token是用户还是物品）来决定加载用户Embedding还是物品Embedding。这在Transformer处理用户行为序列时非常有用，因为序列中可能既有物品（用户点击的），也有用户自身的属性（如用户画像作为序列开头）。
    *   例如 `user_embedding = self.user_emb(user_mask * seq)`：当 `user_mask` 为 True 的位置，`seq` 的值会被保留并用于查找用户Embedding；当 `user_mask` 为 False（即 `0`）的位置，`seq` 的值会变为 `0`，从而查找 `padding_idx=0` 对应的0向量，避免对非用户位置的用户Embedding查找。

*   **其他特征的处理：**
    *   **稀疏特征：** `self.sparse_emb[k](tensor_feature)` 简单地进行Embedding查找。
    *   **数组特征：** `self.sparse_emb[k](tensor_feature).sum(2)`。先进行Embedding查找，然后沿着数组维度（第2个维度）求和。求和是一种常见的将多值特征聚合成一个向量的方式。
    *   **连续特征：** `tensor_feature.unsqueeze(2)`。由于连续特征是数值，本身就是一个标量，需要 `unsqueeze(2)` 增加一个维度，使其变为 `[batch_size, seq_len, 1]`，以便后续与其他 `hidden_units` 维度的Embedding拼接。
    *   **多模态特征：** `self.emb_transform[k](tensor_feature)`。经过线性变换 `Linear` 层，将高维预训练向量映射到 `hidden_units` 维度。

*   **特征融合 (`torch.cat` 和 `userdnn`/`itemdnn`):**
    *   所有相同主体的（用户或物品）的Embedding列表 (`item_feat_list`, `user_feat_list`) 会在最后一个维度 (`dim=2`) 上进行拼接 (`torch.cat`)。
    *   拼接后的巨大向量 (`all_item_emb`, `all_user_emb`) 会再通过一个独立的线性层 (`self.itemdnn`, `self.userdnn`) 进行最终的融合和降维，将其维度统一到 `hidden_units`。这允许模型学习不同特征之间的复杂交互，并得到一个统一且信息丰富的物品或用户表示。

---

### Transformer编码器

Transformer模型是当前深度学习领域最重要的架构之一，尤其在处理序列数据方面表现卓越。这个 `BaselineModel` 的核心正是其Transformer编码器，它通过多层自注意力机制来学习用户行为序列中物品之间的复杂关系。

#### 3.1 Transformer概览

Transformer模型摒弃了传统的循环神经网络（RNN）和卷积神经网络（CNN）对序列的依赖，完全基于**自注意力机制 (Self-Attention)**。它的核心优势在于：
*   **并行化：** 能够同时处理序列中的所有位置，而非像RNN那样顺序处理，大大提高了训练效率。
*   **长距离依赖：** 自注意力机制可以直接捕获序列中任意两个位置之间的关系，有效解决了RNN在处理长序列时梯度消失/爆炸和长距离依赖捕获困难的问题。
*   **捕获上下文：** 通过多头注意力，模型可以从不同表示子空间和不同位置关注上下文信息，从而生成更丰富的上下文敏感的Embedding。

#### 3.2 Flash多头注意力机制 (`FlashMultiHeadAttention`)

`FlashMultiHeadAttention` 是这个模型的一个重要优化点。它实现了Transformer中的核心组件——多头注意力机制，并强调了对Flash Attention的集成。

**数学表示回顾：**
对于一个输入序列的某个位置 $X$，多头注意力首先将其线性变换为查询 (Query) $Q$、键 (Key) $K$ 和值 (Value) $V$：

$$
Q = X W_Q, K = X W_K, V = X W_V
$$

然后，计算注意力得分：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^\top}{\sqrt{d_k}}\right) V
$$

其中 $d_k$ 是每个头的维度（`head_dim`），用于缩放点积，防止梯度过大。多个头独立计算注意力，然后将它们的输出拼接，再通过一个线性层投影回原始维度。

**代码实现细节与维度追踪：**
*   **线性变换：** `self.q_linear`, `self.k_linear`, `self.v_linear` 将输入 `hidden_units` 维度投影到 `hidden_units` 维度。
    *   输入：`[batch_size, seq_len, hidden_units]`
    *   输出 Q, K, V：`[batch_size, seq_len, hidden_units]`
*   **重塑为多头格式：** `view` 和 `transpose` 操作将张量重塑为 `[batch_size, num_heads, seq_len, head_dim]`。
    *   `Q = Q.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)`
        *   先 `view` 将 `hidden_units` 拆分成 `num_heads * head_dim`。形状：`[batch_size, seq_len, num_heads, head_dim]`
        *   再 `transpose(1, 2)` 将 `seq_len` 和 `num_heads` 维度交换，使得 `num_heads` 成为批次后的第二个维度。形状：`[batch_size, num_heads, seq_len, head_dim]`。这符合多头注意力并行计算的习惯。
*   **Flash Attention集成：**
    *   `if hasattr(F, 'scaled_dot_product_attention'):` 这是检测PyTorch 2.0+是否支持Flash Attention的标志。如果支持，直接调用 `F.scaled_dot_product_attention`。
    *   **Flash Attention的优化特性：**
        *   **内存效率 (O(N) 空间复杂度而非 O(N²))：** Flash Attention通过将Attention计算和Softmax操作融合在一个CUDA核中，避免了显存的读写瓶颈，尤其在序列长度 $N$ 很大时，可以显著降低内存占用。
        *   **计算速度：** 减少了显存访问，提高了计算吞吐量。
        *   **自动降级：** 这是一个良好的工程实践，确保了模型在不同PyTorch版本和硬件环境下都能运行。
    *   **标准注意力降级：** 如果不支持Flash Attention，则回退到标准的矩阵乘法和Softmax计算。这部分代码严格按照Attention的数学公式实现：
        *   `scores = torch.matmul(Q, K.transpose(-2, -1)) * scale` 计算注意力分数。
        *   `scores.masked_fill_(attn_mask.unsqueeze(1).logical_not(), float('-inf'))` 应用注意力掩码。将需要被掩盖的位置的分数设为负无穷，在Softmax后这些位置的权重将趋近于0。`attn_mask.unsqueeze(1)` 是为了在多头维度上广播。
        *   `F.softmax(scores, dim=-1)` 计算注意力权重。
        *   `F.dropout(...)` 应用Dropout进行正则化。
        *   `attn_output = torch.matmul(attn_weights, V)` 计算加权后的值向量。
*   **输出处理：** 再次使用 `transpose` 和 `contiguous().view` 将多头输出重塑回 `[batch_size, seq_len, hidden_units]`，然后通过 `self.out_linear` 投影回最终输出维度。

#### 3.3 逐点前馈网络 (`PointWiseFeedForward`)

`PointWiseFeedForward` 层是Transformer中的另一个关键组件，它在每个位置独立地对特征进行非线性变换。

*   **网络结构：** 它是一个两层的全连接网络（MLP），中间使用ReLU激活函数和Dropout。
*   **卷积实现 (`Conv1d`) 的巧妙之处：**
    *   代码使用 `torch.nn.Conv1d(hidden_units, hidden_units, kernel_size=1)` 来实现全连接层。
    *   **原因：** `Conv1d` 期望输入是 `[batch_size, channels, sequence_length]`。当 `kernel_size=1` 时，它相当于对每个序列位置（`sequence_length`）独立地应用一个 `channels` 到 `channels` 的线性变换。这正是逐点前馈网络的行为。
    *   **优点：** 在GPU上，1D卷积通常比直接的全连接层在处理这种逐点操作时有更好的并行计算性能和内存局部性，从而提高效率。
*   **维度转置：** `inputs.transpose(-1, -2)` 和 `outputs.transpose(-1, -2)` 是为了适应 `Conv1d` 的输入格式要求，并最终恢复原始维度顺序。
    *   输入：`[batch_size, seq_len, hidden_units]`
    *   转置后：`[batch_size, hidden_units, seq_len]` (Conv1D期望的 `N, C, L` 格式)
    *   Conv1D处理后：`[batch_size, hidden_units, seq_len]`
    *   恢复后：`[batch_size, seq_len, hidden_units]`

#### 3.4 序列特征编码 (`log2feats`)

`log2feats` 方法将所有前述组件整合起来，实现了从原始用户行为序列到最终特征表示的完整Transformer编码过程。

*   **1. 特征嵌入 (`self.feat2emb`)：** 首先将 `log_seqs` (ID序列) 和 `seq_feature` (辅助特征) 通过 `feat2emb` 转换为统一的 `hidden_units` 维度的稠密Embedding `seqs`。
*   **2. 特征缩放：** `seqs *= self.item_emb.embedding_dim**0.5`
    *   这是Transformer中的一个标准操作，旨在**缩放Embedding**。当Embedding维度较大时，点积结果也会很大，这可能导致Softmax函数的输入过大，使得梯度变得非常小（Softmax饱和），从而影响训练稳定性。
    *   通过除以 $\sqrt{d_{model}}$ (这里是 `hidden_units` 的平方根)，可以有效防止这种情况，保持梯度在合适的范围内。
*   **3. 位置编码 (`self.pos_emb`)：**
    *   `poss = torch.arange(1, maxlen + 1, device=self.dev).unsqueeze(0).expand(batch_size, -1).clone()`：生成一个 `[1, 2, ..., maxlen]` 的序列，并扩展到 `[batch_size, maxlen]`。
    *   `poss *= log_seqs != 0`：**关键步骤！** 这确保了只有非填充位置（即 `log_seqs != 0`）才拥有非零的位置编码。填充位置的 `poss` 仍然是0，对应的 `self.pos_emb(0)` 是一个0向量，不会影响填充位置的Embedding。
    *   `seqs += self.pos_emb(poss)`：将位置编码直接加到物品/用户Embedding上。这种可学习的位置编码能够捕获序列中元素的顺序信息，因为自注意力本身是位置无关的。
*   **4. Embedding Dropout：** `self.emb_dropout(seqs)` 在Embedding上应用Dropout，进一步进行正则化。
*   **5. 注意力掩码 (`attention_mask`)：**
    *   **因果掩码 (`attention_mask_tril`)：** `torch.tril(ones_matrix)` 创建一个下三角矩阵。这强制模型在预测序列中当前位置的输出时，只能关注到当前位置及之前的输入，而不能“偷看”未来的信息。这对于序列预测任务至关重要。
    *   **Padding掩码 (`attention_mask_pad`)：** `(mask != 0).to(self.dev)`。根据 `mask`（`0` 表示填充，`1` 表示物品，`2` 表示用户），生成一个布尔掩码，确保注意力不会计算到填充位置。
    *   **组合掩码：** `attention_mask_tril.unsqueeze(0) & attention_mask_pad.unsqueeze(1)`。通过逻辑与操作，将因果掩码和Padding掩码结合起来。`unsqueeze` 是为了进行广播，使其适配 `[batch_size, num_heads, seq_len, seq_len]` 的注意力得分矩阵。
*   **6. 多层Transformer编码器：**
    *   循环遍历 `args.num_blocks` 次，每个循环包含一个多头注意力和一个前馈网络。
    *   **Pre-LN vs. Post-LN (`norm_first`)：** 模型提供了两种LayerNorm（LN）的放置方式：
        *   **Pre-LN (norm_first=True)：** `LN(x)` -> `Attn(LN(x))` -> `x + Attn_out` -> `LN(x)` -> `FFN(LN(x))` -> `x + FFN_out`。在每个子层输入前先进行LayerNorm。这种结构被认为在训练深层Transformer时更稳定，尤其是在使用残差连接和Dropout的情况下。
        *   **Post-LN (norm_first=False)：** `Attn(x)` -> `x + Attn_out` -> `LN(x + Attn_out)` -> `FFN(x)` -> `x + FFN_out` -> `LN(x + FFN_out)`。在每个子层输出后进行LayerNorm。这是原始Transformer论文中采用的方式。
    *   **残差连接 (`seqs = seqs + ...`)：** 每个子层的输出都与子层的输入相加。这有助于解决深层网络的梯度消失问题。
*   **7. 输出LayerNorm：** `self.last_layernorm(seqs)` 对最终的 `log_feats` 进行归一化。

`log2feats` 函数的输出 `log_feats` 是一个 `[batch_size, seq_len, hidden_units]` 的张量，其中每个 `[hidden_units]` 维度的向量都编码了该序列位置及其之前所有位置的丰富上下文信息。这些高质量的序列表示将用于后续的推荐预测。

---

