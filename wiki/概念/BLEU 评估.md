---
created: 2026-04-03
tags:
  - concept
  - deep-learning
  - NLP
---

# BLEU 评估

Bilingual Evaluation Understudy Score，一种衡量生成文本（通常是翻译）与参考译文相似程度的标准指标。

## 核心思想

BLEU 通过计算 **n-gram 精确度**（precision）来衡量候选文本与参考文本的相似程度：
- 统计候选文本中每个 n-gram 在参考文本中出现的次数
- 除以候选文本中该 n-gram 的总出现次数
- 对 1-gram 到 4-gram 分别计算，再取几何平均

## 计算公式（简化）

```
BLEU = BP · exp( Σ_{n=1}^{N} w_n · log P_n )
```

- P_n：n-gram 精确度
- w_n：权重（通常 uniform）
- BP：简短惩罚（Brevity Penalty），防止过短输出获得高分

BP = 1（若 cand_len ≥ ref_len）或 exp(1 - ref_len/cand_len)（若 cand_len < ref_len）

## BLEU 与 Transformer 结果

| 任务 | 模型 | BLEU |
|---|---|---|
| WMT EN→DE | Transformer Base | 25.8 |
| WMT EN→DE | Transformer Big | **28.4** |
| WMT EN→FR | Transformer Big | **41.8** |

## BLEU 的局限性

- **只看 n-gram 精确度**，不考虑语义和流畅性
- 对措辞差异大的同义翻译可能给出低分
- 对长序列任务（摘要）表现较差

后续改进指标：METEOR、ROUGE（摘要）、BERTScore（语义相似度）、GLEU（Google 内部）等。

## 相关概念

- [[序列转导]] — BLEU 所评估的任务类型
- [[Transformer 架构]] — 产生 BLEU 评估数据的模型
- [[编码器-解码器]] — BLEU 主要用于评估 seq2seq 模型输出
