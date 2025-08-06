---
title: LayerNorm
date: 2025-08-03
slug: blog-post-slug
tags:
  - 深度学习
  - 归一化方法
  - Transformer
  - 自然语言处理
categories:
  - 分类
description: 描述
draft: true
state: "0"
---

---

### **1. LayerNorm 核心思想与数学原理**

#### **问题动机**

深度网络中，每一层输入的分布随训练动态变化（Internal Covariate Shift），导致梯度不稳定、收敛慢。LayerNorm 通过对**单个样本的所有特征**进行归一化，统一数据尺度，与 Batch Size 无关，适用于序列数据（如 NLP 任务）。

#### **数学公式推导**

给定输入向量 $\mathbf{x} \in \mathbb{R}^d$（如一个词向量的嵌入表示）：

1. **计算均值与方差**：  
    $$\mu = \frac{1}{d} \sum_{i=1}^{d} x_i, \quad \sigma^2 = \frac{1}{d} \sum_{i=1}^{d} (x_i - \mu)^2$$  
    （$d$ 为特征维度，如嵌入维度 512）
    
2. **归一化**：  
    $$\hat{x}_i = \frac{x_i - \mu}{\sqrt{\sigma^2 + \epsilon}}$$  
    （$\epsilon$ 为数值稳定性常数，如 $10^{-5}$）
    
3. **仿射变换**：  
    $$y_i = \gamma \hat{x}_i + \beta$$  
    （$\gamma, \beta$ 为可学习参数，恢复模型的表达能力）
    

#### **与 BatchNorm 的关键区别**

- **BatchNorm**：对 Batch 内所有样本的**同一特征通道**归一化（依赖 Batch 统计量）。
- **LayerNorm**：对每个样本的**所有特征维度**归一化（独立于 Batch）
    
    。
    
    ```
    # BatchNorm 示例 (图像数据)
    bn = nn.BatchNorm2d(num_features=64)  # 对每个通道归一化
    
    # LayerNorm 示例 (序列数据)
    ln = nn.LayerNorm(normalized_shape=512)  # 对每个词向量的 512 维归一化
    ```
    

---

### **2. LayerNorm 的实际作用**

#### **稳定训练机制**

- **梯度控制**：  
    深层网络中，激活值过大或过小会导致梯度爆炸/消失。LayerNorm 将输入约束到 $\mathcal{N}(0, 1)$ 附近，确保反向传播时梯度幅值稳定
    
    。  
    **实例**：Transformer 训练中，若无 LayerNorm，注意力层的输出可能因点积操作而指数级增大（如 $\text{Softmax}(QK^T)$ 的输入值过大）。
    
- **加速收敛**：  
    统一特征尺度后，优化器（如 Adam）可对所有权重使用相近的学习率，避免某些维度更新过快/慢
    
    。
    

#### **在序列模型中的关键性**

- **RNN/Transformer 适配性**：  
    序列数据长度可变（如句子长度不同），BatchNorm 无法计算同一特征通道的稳定统计量（因序列长度不同）。LayerNorm 对每个时间步独立操作，天然适配
    
    。  
    **代码对比**：
    
    ```
    # RNN 中 LayerNorm 应用
    class RNNWithLN(nn.Module):
        def __init__(self, input_size, hidden_size):
            super().__init__()
            self.rnn = nn.RNN(input_size, hidden_size)
            self.ln = nn.LayerNorm(hidden_size)  # 对每个时间步的输出归一化
    
        def forward(self, x):
            x, _ = self.rnn(x)  # x.shape = [seq_len, batch, hidden_size]
            return self.ln(x)
    ```
    

---

### **3. PyTorch 手写实现**

#### **完整代码**

```
import torch
import torch.nn as nn

class LayerNorm(nn.Module):
    def __init__(self, d_model, eps=1e-5):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(d_model))  # 缩放参数 γ
        self.beta = nn.Parameter(torch.zeros(d_model))  # 平移参数 β
        self.eps = eps

    def forward(self, x):
        # x.shape = [batch, seq_len, d_model] 或 [batch, d_model]
        mean = x.mean(dim=-1, keepdim=True)  # 沿特征维度求均值
        var = x.var(dim=-1, keepdim=True, unbiased=False)  # 方差（无偏估计=False）
        x_norm = (x - mean) / torch.sqrt(var + self.eps)
        return self.gamma * x_norm + self.beta
```

#### **关键细节**

- **张量维度**：`dim=-1` 表示对最后一个维度（特征维度）操作。
- **无偏方差**：训练时使用样本方差（除以 $d$ 而非 $d-1$)，与 PyTorch 官方实现一致
    
    。
- **训练/推理一致性**：LayerNorm 无全局统计量，无需区分模式（与 BatchNorm 不同）。

---

### **4. 应用案例**

#### **案例 1：Transformer 文本分类**

```
class TransformerBlock(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.attn = nn.MultiheadAttention(d_model, num_heads)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, 4*d_model),
            nn.ReLU(),
            nn.Linear(4*d_model, d_model)
        )
        self.norm1 = LayerNorm(d_model)  # Pre-Norm 结构
        self.norm2 = LayerNorm(d_model)

    def forward(self, x):
        # Pre-Norm：先归一化再进入子层
        x = x + self.attn(self.norm1(x), self.norm1(x), self.norm1(x))[0]
        x = x + self.ffn(self.norm2(x))
        return x
```

- **Pre-Norm vs Post-Norm**：
    - Post-Norm（原始 Transformer）：`x = norm(x + sublayer(x))`
    - Pre-Norm（主流）：`x = x + sublayer(norm(x))`，更稳定且支持更深网络。

#### **案例 2：多维输入处理**

- **图像+文本混合模型**：  
    对卷积特征图（4D 张量）使用 LayerNorm：
    
    ```
    # 输入形状 [batch, channels, height, width]
    ln = nn.LayerNorm([channels, height, width])  # 归一化所有空间位置
    ```
    

---

### **5. 扩展：LayerNorm 变种**

#### **RMSNorm（均方根归一化）**

- **思想**：移除均值中心化，仅用标准差缩放：  
    $$\hat{x}_i = \frac{x_i}{\sqrt{\frac{1}{d}\sum_{i=1}^d x_i^2 + \epsilon}}$$
- **优势**：计算量减少 15-20%，在 LLM（如 LLaMA）中广泛使用。
- **代码**：
    
    ```
    class RMSNorm(nn.Module):
        def __init__(self, d_model, eps=1e-6):
            self.scale = nn.Parameter(torch.ones(d_model))
            self.eps = eps
    
        def forward(self, x):
            rms = torch.sqrt(torch.mean(x**2, dim=-1, keepdim=True) + self.eps)
            return x / rms * self.scale
    ```
    
