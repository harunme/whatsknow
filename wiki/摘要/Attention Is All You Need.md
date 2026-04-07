---
created: 2026-04-03
source: "[[raw/Attention Is All You Need]]"
tags:
  - summary
  - source
---

# Attention Is All You Need — 摘要

**来源：** Vaswani et al., Google Brain / Google Research, 2017
**原始文件：** `raw/Attention Is All You Need.pdf`

## 概述

本文提出了 **Transformer**——一种完全基于注意力机制（Attention）的序列转导（sequence transduction）模型，摒弃了传统的循环和卷积结构。该模型在 WMT 2014 英德翻译任务上达到 28.4 BLEU（在测试集上），在 WMT 2014 英法翻译任务上达到 41.8 BLEU，创下当时单模型的最佳成绩，且训练成本显著低于此前最佳模型。

## 背景：序列转导问题

序列转导是将一个输入序列转换为输出序列的任务，是无数 NLP 任务（翻译、摘要、问答等）的核心问题。在此之前，主流方法是**编码器-解码器架构**搭配 RNN（含 LSTM/GRU），但存在以下固有缺陷：

- **顺序计算**阻碍并行化，训练时间长
- **长距离依赖**难以捕捉（信息需经过多步传递）
- **上下文表示压缩**瓶颈问题

注意力机制被引入作为对 RNN 的补充，但此前工作仍以 RNN 为主体。

## 核心贡献：Transformer 架构

### 整体结构

```
输入序列
    ↓
编码器（6 层堆叠）
    ↓ 编码表示
解码器（6 层堆叠）
    ↓
输出序列
```

编码器和解码器各由 **6 个相同层**堆叠而成。

### 编码器（Encoder）

每层包含两个子层：
1. **多头自注意力层（Multi-Head Self-Attention）**
2. **基于位置的全连接前馈网络（Position-wise Feed-Forward）**

每个子层周围有**残差连接**（residual connection），并通过**层归一化**（Layer Normalization）处理：`LayerNorm(x + Sublayer(x))`

### 解码器（Decoder）

每层包含三个子层：
1. **掩码多头自注意力（Masked Multi-Head Attention）** — 防止看到未来位置
2. **编码器-解码器注意力层（Encoder-Decoder Attention）** — Query 来自解码器，Key/Value 来自编码器输出
3. **前馈网络**

同样使用残差 + 层归一化。

### 注意力机制

**缩放点积注意力（Scaled Dot-Product Attention）：**
```
Attention(Q, K, V) = softmax(QK^T / √d_k) V
```
除以 √d_k 是为了防止点积值过大导致 softmax 梯度消失。

**多头注意力（Multi-Head Attention）：**
```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) W^O
where head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)
```
使用 h=8 个注意力头，每个头的维度 d_k = d_model/h = 64。

### 位置编码（Positional Encoding）

由于 Transformer 没有循环也没有卷积，无法自然获得序列位置信息，因此引入**基于正弦和余弦函数的位置编码**：

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

该编码方式允许模型学习相对位置关系（因为 `sin/cos` 的线性组合可表示位移）。

### 训练细节

| 项目 | 细节 |
|---|---|
| 优化器 | Adam（β₁=0.9, β₂=0.98, ε=10⁻⁹） |
| 学习率调度 | 预热（warmup）步数 4000，预热后用 inverse sqrt 衰减 |
| 正则化 | 残差 Dropout（rate=0.1）、标签平滑（LS=0.1） |
| 句子长度 | 最大 64 |
| batch | 近 38000 个源词 / 目标词 |
| 训练时长 | 3.5 天（8 块 P100 GPU） |

## 实验结果

| 任务 | 模型 | BLEU |
|---|---|---|
| WMT EN→DE | Transformer（big） | **28.4**（单模型 SOTA） |
| WMT EN→FR | Transformer（big） | **41.8**（单模型 SOTA） |
| 英德对比 | RNN-seq2seq + 注意力 | 25.9 |
| 英德对比 | CFSC（ConvS2S） | 26.3 |
| 英德对比 | ByteNet | 23.8 |

## 消融分析（Ablation Studies）

对 Transformer（big）模型各组件进行了消融实验，结论：
- **多头注意力**比单头效果好（重要）
- **注意力头数 h=16** 略优于 h=8 和 h=4
- 注意力维度 d_model=64 是效果与效率的最佳平衡点
- 位置编码使用正弦函数与可学习编码效果相当（正弦函数可泛化到更长序列）
- 移除残差连接或层归一化会导致训练崩溃
- 前馈网络中的 ReLU 改为 GeLU 效果相近（GeLU 更常用）

## 核心结论

> "Attention is All You Need."

Transformer 完全抛弃了循环和卷积，仅用注意力机制构建序列转导模型。这一设计带来了：
- **高度并行化**：训练速度大幅提升
- **直接建模任意距离依赖**：注意力权重直接连接任意两个位置
- **可扩展性**：为后继大规模预训练模型（GPT、BERT 等）奠定基础

## 本 wiki 的处理方式

本文（`raw/Attention Is All You Need.pdf`）正是 Karpathy 所描述的 LLM 知识库工作流的典型原材料：
- 通过 [[数据摄入]] 进入 `raw/`
- 经 [[Wiki 编译]] 生成摘要及 9 篇概念文章（Transformer 架构、注意力机制等）
- 通过 [[LLM 原生问答]]，可向 LLM 提问关于 Transformer 的任何细节
- 相关概念均已互相链接，形成结构化知识网络

参见：[[Thread by @karpathy — 摘要]]

## 相关概念

- [[Transformer 架构]]
- [[注意力机制]]
- [[多头注意力]]
- [[位置编码]]
- [[编码器-解码器]]
- [[残差连接与层归一化]]
- [[自注意力]]
- [[序列转导]]
