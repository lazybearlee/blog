---
title: Transformer中的位置编码
date: 2025-08-01
slug: blog-post-slug
tags:
  - 深度学习
  - 自然语言处理
  - Transformer
categories:
  - 分类
description: 描述
draft: true
state: "0"
---
### 自注意力机制的“缺陷”

首先，我们必须认识到：**纯粹的自注意力机制是“无序”的**。

对于自注意力来说，句子“猫坐在垫子上”和“垫子坐在猫上”是完全一样的。因为它在计算每个词的注意力时，只是对所有词进行一次加权求和，并不关心这些词的原始顺序。这种性质被称为**排列不变性（Permutation Invariance）**。

在很多任务中，顺序至关重要。例如，对于推荐系统，用户是先点击A再点击B，还是先点击B再点击A，其意图可能完全不同。因此，我们必须找到一种方法，将序列的“位置信息”注入到模型中。这就是位置编码的使命。

### 探索解决方案

在直接看最终答案前，我们先像研究者一样思考，尝试几种“朴素”的解决方案，并分析它们的缺陷。这会让我们更深刻地理解最终方案的巧妙之处。

#### 方案一 整数索引

最直接的想法：给每个位置一个整数编号，比如序列的第一个词位置是0，第二个是1，第三个是2...然后把这个整数加到词向量上。

*   **缺陷**：
    1.  **数值范围问题**：序列越长，位置编码的数值越大。这会导致位置编码的数值主导了词向量本身，使得模型训练不稳定。
    2.  **泛化能力差**：如果模型在训练时见过的最长序列是50，那它完全不知道位置51代表什么。
    3.  **缺乏距离感**：模型很难从 `5` 和 `6` 这两个数字中，学到它们代表“相邻”这个概念。对模型来说，`5` 和 `50` 的差异可能只是数值大小，而不是位置的远近。

#### 方案二 归一化索引

为了解决数值范围问题，我们可以将位置索引归一化到 `[0, 1]` 区间。例如，对于一个长度为 `L` 的序列，第 `pos` 个位置的编码是 `pos / (L-1)`。

*   **缺陷**：
    1.  **相对距离不一致**：在长度为10的序列中，位置1和位置2的编码差是 `1/9`。在长度为100的序列中，它们的编码差是 `1/99`。这意味着“相邻”这个概念的度量，会随着序列长度的变化而变化。模型无法学到一个稳定、一致的相对位置关系。

### 最终方案 正弦与余弦

经历了朴素方案的失败，我们总结出一个好的位置编码应该具备的特质：
1.  **唯一性**：每个位置都应该有一个独一无二的编码。
2.  **有界性**：编码的数值应该在一定范围内，不能无限增长。
3.  **确定性**：对于相同长度的序列，位置编码应该是固定和可复现的。
4.  **泛化性**：模型应该能处理比训练时更长的序列。
5.  **相对性**：编码应该能体现出位置间的相对关系。

正弦/余弦（sinusoidal）编码方案，完美地满足了以上所有要求。其公式如下：

对于位置 `pos` 和维度索引 `i`（从 `0` 到 `d_model-1`），位置编码向量 $PE_{(pos)}$ 的每个分量计算如下：

$$
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$
$$
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$

其中：
*   `pos` 是词在序列中的位置 (0, 1, 2, ...)。
*   `d_model` 是词向量的维度 (e.g., 512)。
*   `2i` 和 `2i+1` 是指编码向量中的维度索引，即偶数维度用 `sin`，奇数维度用 `cos`。

#### 直观理解公式

不要被公式吓到。我们可以把它想象成一种**用不同频率的波来表示位置**的方式。

*   **分母 `10000^(2i/d_model)`**：这部分是关键，它控制了波的**波长（wavelength）**。
    *   当 `i` 很小（即在向量的开头部分），`2i/d_model` 趋近于0，分母趋近于1。这对应着**高频率**（短波长）的波。这些维度的值会随着 `pos` 的变化而快速变化，它们编码了位置的**精细信息**。
    *   当 `i` 很大（即在向量的结尾部分），`2i/d_model` 趋近于1，分母趋近于10000。这对应着**低频率**（长波长）的波。这些维度的值变化非常缓慢，它们编码了位置的**粗略信息**。

**一个绝佳的类比是二进制编码**：
一个数字 `5` 可以表示为 `...00101`。最低位（最右边）变化最快（0, 1, 0, 1...），最高位变化最慢。位置编码也是如此，它用一个连续的向量，通过不同频率的组合，为每个位置生成了一个独一无二的“数字签名”。

#### 相对位置的线性表示

这是该公式最天才的设计，也是它为何优于其他方案的根本原因。**一个位置 `pos+k` 的编码，可以表示为位置 `pos` 编码的一个线性变换**。

基于三角函数的和差角公式：

$$ \sin(\alpha+\beta) = \sin(\alpha)\cos(\beta) + \cos(\alpha)\sin(\beta) $$

$$ \cos(\alpha+\beta) = \cos(\alpha)\cos(\beta) - \sin(\alpha)\sin(\beta) $$

令 $\alpha = \frac{pos}{10000^{2i/d_{\text{model}}}}$ 和 $\beta = \frac{k}{10000^{2i/d_{\text{model}}}}$，我们可以推导出：
$PE_{(pos+k, 2i)}$ 和 $PE_{(pos+k, 2i+1)}$ 都可以由 $PE_{(pos, 2i)}$ 和 $PE_{(pos, 2i+1)}$ 线性组合而成。

这意味着什么？

对于自注意力机制来说，它只需要计算查询 $Q$ 和键 $K$ 的点积。由于位置编码是直接加到词向量上的，所以 $Q$ 和 $K$ 都包含了这个位置信息。模型在计算注意力分数时，它**不需要学习绝对位置**，而只需要**学习一个线性变换**，就能推断出两个词之间相隔了 `k` 个位置的相对关系。

这使得模型非常容易捕捉到“前一个词”、“后三个词”这类相对位置信息，并且这种能力可以泛化到任意长的序列。

### PyTorch 代码实现与可视化


```python
import torch
import torch.nn as nn
import math
import matplotlib.pyplot as plt
import seaborn as sns

class PositionalEncoding(nn.Module):
    def __init__(self, d_model: int, max_len: int = 5000):
        """
        初始化位置编码模块
        d_model: 词向量的维度
        max_len: 支持的最大序列长度
        """
        super().__init__()
        
        # 创建一个足够大的位置编码矩阵，形状为 (max_len, d_model)
        pe = torch.zeros(max_len, d_model)
        
        # 创建一个位置张量，形状为 (max_len, 1)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        
        # 计算波长的分母部分，这是公式的核心
        # 我们在log空间计算以提高数值稳定性
        # div_term 的形状为 (d_model / 2)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model))
        
        # 使用广播机制计算 sin 和 cos
        # position (max_len, 1) * div_term (d_model / 2) -> (max_len, d_model / 2)
        pe[:, 0::2] = torch.sin(position * div_term) # 偶数维度
        pe[:, 1::2] = torch.cos(position * div_term) # 奇数维度
        
        # 将pe矩阵增加一个batch维度，变为 (1, max_len, d_model)
        # 这样它就可以方便地与 (batch_size, seq_len, d_model) 的输入相加
        # register_buffer确保这个张量是模型的一部分，但不是可训练的参数
        self.register_buffer('pe', pe.unsqueeze(0))

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        """
        前向传播，将位置编码加到输入张量上
        x: 输入张量，形状 (batch_size, seq_len, d_model)
        """
        # 截取所需长度的位置编码，并加到输入上
        # self.pe 的形状是 (1, max_len, d_model)
        # x.size(1) 是输入的实际序列长度 seq_len
        x = x + self.pe[:, :x.size(1), :]
        return x

# --- 可视化位置编码 ---
d_model_vis = 128
max_len_vis = 100

pe_layer = PositionalEncoding(d_model_vis, max_len_vis)
pe_matrix = pe_layer.pe.squeeze(0).numpy() # 获取 (max_len, d_model) 的矩阵

plt.figure(figsize=(10, 8))
sns.heatmap(pe_matrix, cmap='viridis')
plt.xlabel("Embedding Dimension (i)")
plt.ylabel("Position (pos)")
plt.title("Positional Encoding Matrix")
plt.show()

```

#### 可视化结果分析

![](Blog/Deeplearning/assets/trans.png)

上图是运行代码后看到的热力图。
*   **Y轴**是位置 `pos` (0-99)。
*   **X轴**是词向量的维度 `i` (0-127)。
*   **颜色**代表该位置、该维度的编码值。

你可以清晰地观察到：
1.  **左侧的条纹非常密集**：这对应着高频波（`i` 较小），编码值变化很快。
2.  **右侧的条纹非常宽大**：这对应着低频波（`i` 较大），编码值变化很慢。
3.  **每一行都是独一无二的**：没有任何两行（两个位置）的编码模式是完全相同的。
