---
created: 2026-04-03
tags:
  - concept
  - deep-learning
  - NLP
---

# Transformer 架构

2017 年 Vaswani 等人在论文 *"Attention Is All You Need"* 中提出的序列建模架构，是现代大语言模型（LLM）的基石。

## 核心设计原则

Transformer 完全基于**注意力机制**，摒弃了循环神经网络（RNN）和卷积神经网络（CNN）：
- 无顺序依赖 → 可高度并行化训练
- 任意位置间直接建模依赖 → 解决长距离依赖问题
- 架构简洁，可堆叠 →天然具备 Scaling 能力

## 整体结构

```
输入嵌入 + 位置编码
        ↓
   [编码器层] × N（论文中 N=6）
        ↓
   [解码器层] × N（论文中 N=6）
        ↓
     线性层 + Softmax
        ↓
      输出概率
```

## 编码器（Encoder）

每层包含两个子层：
1. **多头自注意力**（Multi-Head Self-Attention）
2. **前馈网络**（Position-wise Feed-Forward Network）

每个子层周围有**残差连接** + **层归一化**：`Output = LayerNorm(x + Sublayer(x))`

## 解码器（Decoder）

每层包含三个子层：
1. **掩码多头自注意力**（Masked Multi-Head Attention）— 遮蔽未来 token
2. **编码器-解码器注意力**（Cross Attention）— 关注编码器输出
3. **前馈网络**

## 规模与变体

| 配置 | 编码器层数 | d_model | 注意力头数 | 前馈维度 |
|---|---|---|---|---|
| Base | 6 | 512 | 8 | 2048 |
| Big | 6 | 1024 | 16 | 4096 |

## 影响

Transformer 之后，几乎所有顶级 NLP 模型都基于此架构：
- **GPT 系列**（仅解码器）：生成式预训练
- **BERT**（仅编码器）：掩码语言建模
- **T5**（编码器-解码器）：文本到文本统一框架
- 以及后续的 GPT-4、LLaMA、Claude 等所有主流 LLM

## 相关概念

- [[注意力机制]] — Transformer 的核心计算单元
- [[多头注意力]] — 编码器和解码器中使用的注意力形式
- [[位置编码]] — 为 Transformer 注入序列位置信息
- [[编码器-解码器]] — Transformer 的结构模板
- [[自注意力]] — 多头自注意力的特例
- [[残差连接与层归一化]] — 每层的稳定机制
- [[序列转导]] — Transformer 所解决的核心任务
