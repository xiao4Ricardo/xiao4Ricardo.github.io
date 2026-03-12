---
layout:     post
title:      "NLP 实战：用 Transformers 做 NER 和情感分析"
subtitle:   "Hands-on NLP with Hugging Face Transformers"
date:       2026-01-14 14:30:00
author:     "Tony L."
header-img: "img/post-bg-js-module.jpg"
tags:
    - NLP
    - Transformer
    - Machine Learning
    - Python
---

## 引言

自然语言处理（NLP）已经从传统的规则系统和统计方法全面转向基于 Transformer 的预训练模型。Hugging Face 的 Transformers 库让使用这些强大模型变得前所未有的简单。本文通过两个经典任务——命名实体识别（NER）和情感分析——展示如何用 Transformers 库解决实际 NLP 问题。

## 环境准备

```bash
pip install transformers datasets evaluate seqeval
pip install torch  # 或 tensorflow
```

## Part 1: 命名实体识别 (NER)

NER 的任务是从文本中识别出人名、地名、组织名等命名实体。

### 快速使用（Pipeline）

```python
from transformers import pipeline

ner = pipeline("ner", model="dslim/bert-base-NER",
               aggregation_strategy="simple")

text = "Apple CEO Tim Cook announced a new product at the Cupertino headquarters."
results = ner(text)

for entity in results:
    print(f"{entity['word']:20s} {entity['entity_group']:10s} "
          f"score: {entity['score']:.4f}")
```

输出：
```
Apple                ORG        score: 0.9987
Tim Cook             PER        score: 0.9993
Cupertino            LOC        score: 0.9956
```

### 自定义 NER 模型训练

当预训练模型无法识别你的领域实体时（如医学术语、产品名称），需要自定义训练。

**Step 1: 数据准备**

NER 使用 BIO 标注格式：
```
Apple    B-ORG
CEO      O
Tim      B-PER
Cook     I-PER
announced O
```

**Step 2: 数据加载与 Tokenization**

```python
from datasets import load_dataset
from transformers import AutoTokenizer

dataset = load_dataset("conll2003")
tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")

def tokenize_and_align_labels(examples):
    tokenized = tokenizer(examples["tokens"],
                         truncation=True,
                         is_split_into_words=True)
    labels = []
    for i, label in enumerate(examples["ner_tags"]):
        word_ids = tokenized.word_ids(batch_index=i)
        label_ids = []
        previous_word_idx = None
        for word_idx in word_ids:
            if word_idx is None:
                label_ids.append(-100)
            elif word_idx != previous_word_idx:
                label_ids.append(label[word_idx])
            else:
                label_ids.append(-100)
            previous_word_idx = word_idx
        labels.append(label_ids)
    tokenized["labels"] = labels
    return tokenized
```

关键点：由于 WordPiece/BPE tokenization 会将一个词分成多个子词，我们需要将标签对齐到子词级别。只有每个词的第一个子词保留原始标签，其余子词标记为 -100（在 loss 计算中被忽略）。

**Step 3: 训练**

```python
from transformers import (AutoModelForTokenClassification,
                          TrainingArguments, Trainer)

model = AutoModelForTokenClassification.from_pretrained(
    "bert-base-cased",
    num_labels=len(label_names)
)

training_args = TrainingArguments(
    output_dir="ner-model",
    learning_rate=2e-5,
    per_device_train_batch_size=16,
    num_train_epochs=3,
    weight_decay=0.01,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_train,
    eval_dataset=tokenized_val,
    compute_metrics=compute_metrics,
)

trainer.train()
```

### NER 评估指标

NER 使用 seqeval 库进行评估，关注 precision、recall 和 F1-score（按实体类型分别计算）：

```python
import evaluate
seqeval = evaluate.load("seqeval")

def compute_metrics(p):
    predictions, labels = p
    predictions = predictions.argmax(axis=-1)

    true_labels = [[label_names[l] for l in label if l != -100]
                   for label in labels]
    true_predictions = [[label_names[p] for (p, l) in zip(pred, label)
                        if l != -100]
                       for pred, label in zip(predictions, labels)]

    results = seqeval.compute(predictions=true_predictions,
                              references=true_labels)
    return {
        "precision": results["overall_precision"],
        "recall": results["overall_recall"],
        "f1": results["overall_f1"],
    }
```

## Part 2: 情感分析

情感分析是文本分类的一种，判断文本的情感倾向（正面/负面/中性）。

### Pipeline 快速上手

```python
from transformers import pipeline

sentiment = pipeline("sentiment-analysis",
                    model="nlptown/bert-base-multilingual-uncased-sentiment")

texts = [
    "This product is amazing! Best purchase I've ever made.",
    "Terrible quality. Broke after one week.",
    "It's okay, nothing special but does the job.",
]

for text in texts:
    result = sentiment(text)[0]
    print(f"{result['label']:10s} ({result['score']:.4f}): {text[:50]}")
```

### 自定义情感分析模型

使用 IMDB 电影评论数据集训练一个二分类情感模型：

```python
from datasets import load_dataset
from transformers import (AutoTokenizer,
                          AutoModelForSequenceClassification,
                          TrainingArguments, Trainer)

dataset = load_dataset("imdb")
tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")

def preprocess(examples):
    return tokenizer(examples["text"], truncation=True,
                    max_length=512, padding="max_length")

tokenized = dataset.map(preprocess, batched=True)

model = AutoModelForSequenceClassification.from_pretrained(
    "distilbert-base-uncased",
    num_labels=2
)

training_args = TrainingArguments(
    output_dir="sentiment-model",
    learning_rate=2e-5,
    per_device_train_batch_size=16,
    num_train_epochs=2,
    evaluation_strategy="epoch",
    fp16=True,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized["train"],
    eval_dataset=tokenized["test"],
)

trainer.train()
```

### 模型选型建议

| 需求 | 推荐模型 | 说明 |
|------|----------|------|
| 英文通用 NER | dslim/bert-base-NER | 4 类实体，效果好 |
| 中文 NER | hfl/chinese-roberta-wwm-ext | 中文理解能力强 |
| 英文情感分析 | distilbert-base-uncased | 轻量快速 |
| 多语言情感 | nlptown/bert-base-multilingual | 6 种语言支持 |
| 低资源场景 | SetFit | 少量标注数据即可 |

## 部署注意事项

1. **模型量化**：使用 ONNX Runtime 或 TensorRT 加速推理
2. **批处理**：将多个请求合并为一个 batch 处理
3. **缓存**：对相同输入缓存预测结果
4. **监控**：记录推理延迟、准确率漂移等指标

## 总结

Hugging Face Transformers 库极大地降低了 NLP 任务的门槛。从快速原型（Pipeline）到自定义训练，整个流程都有良好的工具支持。关键是选择合适的预训练模型，准备高质量的标注数据，以及合理设置训练超参数。

## References

1. Devlin, J., et al. (2018). "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding."
2. Hugging Face Transformers Documentation.
3. Tjong Kim Sang, E.F. & De Meulder, F. (2003). "Introduction to the CoNLL-2003 Shared Task."
