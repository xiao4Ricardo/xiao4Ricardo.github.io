---
layout:     post
title:      "LLM 微调实战：LoRA 与 QLoRA"
subtitle:   "Parameter-Efficient Fine-Tuning for Large Language Models"
date:       2025-12-29 13:00:00
author:     "Tony L."
header-img: "img/post-bg-dreamer.jpg"
tags:
    - LLM
    - Fine-tuning
    - Machine Learning
---

## 为什么需要微调？

预训练 LLM 虽然拥有强大的通用能力，但在特定领域或任务上往往无法达到生产级别的效果。微调（Fine-tuning）让我们能够将通用模型适配到特定场景，例如：

- 让模型遵循特定的输出格式（JSON Schema、Markdown 表格等）
- 提升模型在垂直领域（医疗、法律、金融）的专业能力
- 调整模型的"人格"和语气风格
- 教会模型处理私有数据中的特殊模式

## 全参数微调的挑战

对一个 7B 参数的模型进行全参数微调（Full Fine-tuning）：

- **显存需求**：约 28GB（FP32）或 14GB（FP16），再加上优化器状态和梯度，实际需要 ~60GB
- **存储成本**：每个微调版本保存完整模型权重 ~14GB
- **灾难性遗忘**：全参数更新容易破坏预训练学到的通用知识

这使得全参数微调对大多数团队来说不切实际。

## LoRA：低秩适配

LoRA（Low-Rank Adaptation）是目前最流行的参数高效微调方法。核心思想极其优雅：

### 核心原理

预训练权重矩阵 W（维度 d x k）的更新 ΔW 可以分解为两个低秩矩阵的乘积：

```
ΔW = B * A
where B: d x r, A: r x k, r << min(d, k)
```

前向传播变为：
```
h = W*x + (B*A)*x = W*x + B*(A*x)
```

其中 r（秩）通常取 4, 8, 16 或 32。

### 为什么有效？

研究表明，预训练模型的权重更新在微调过程中具有"低内在维度"（low intrinsic dimensionality）。也就是说，虽然模型有数十亿个参数，但有效的更新方向可能只存在于一个低维子空间中。

### 数学直觉

如果原始权重矩阵 W 是 4096 x 4096：
- 全参数微调：更新 16,777,216 个参数
- LoRA (r=8)：更新 4096*8 + 8*4096 = 65,536 个参数
- **参数减少 256 倍！**

### 实践代码

使用 Hugging Face 的 PEFT 库：

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                          # 秩
    lora_alpha=32,                 # 缩放因子
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

model = get_peft_model(base_model, lora_config)
model.print_trainable_parameters()
# trainable params: 13,631,488 || all params: 6,751,326,208
# trainable%: 0.2019%
```

### 关键超参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| r (秩) | 8-64 | 越大能力越强，但开销越大 |
| lora_alpha | 2*r | 缩放因子，通常设为 r 的 1-2 倍 |
| target_modules | 所有线性层 | 作用在哪些层上 |
| lora_dropout | 0.05-0.1 | 正则化 |

## QLoRA：量化 + LoRA

QLoRA 将 LoRA 与量化技术结合，进一步降低显存需求：

### 核心创新

1. **4-bit NormalFloat (NF4)**：一种信息论最优的 4-bit 量化数据类型，针对正态分布权重设计
2. **Double Quantization**：对量化常数本身再进行量化，进一步节省内存
3. **Paged Optimizers**：利用 CPU 内存处理 GPU 显存的 OOM 峰值

### 显存对比

微调 LLaMA-7B：
- 全参数微调 (FP16)：~60GB → 需要 A100 80GB
- LoRA (FP16)：~26GB → 需要 A100 40GB
- QLoRA (NF4)：~6GB → **单张 RTX 3090 即可！**

### 代码实现

```python
from transformers import BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto"
)

# 然后正常应用 LoRA
model = get_peft_model(model, lora_config)
```

## 数据准备

微调数据的质量比数量更重要。几个关键原则：

1. **格式统一**：使用标准的 instruction/input/output 格式
2. **质量优先**：1000 条高质量数据 > 10000 条低质量数据
3. **多样性**：覆盖不同的任务类型和难度
4. **去重**：避免重复或高度相似的样本

数据格式示例：
```json
{
  "instruction": "将以下文本翻译成英文",
  "input": "机器学习是人工智能的一个分支",
  "output": "Machine learning is a branch of artificial intelligence."
}
```

## 训练技巧

1. **学习率**：LoRA 的学习率通常比全参数微调高 10 倍（1e-4 vs 1e-5）
2. **Epochs**：2-5 个 epoch 通常足够，过多会过拟合
3. **Batch Size**：使用 gradient accumulation 模拟大 batch size
4. **评估**：预留 10% 数据作为验证集，监控 loss 曲线

## LoRA 合并与部署

训练完成后，LoRA 权重可以合并回基础模型：

```python
merged_model = model.merge_and_unload()
merged_model.save_pretrained("merged-model")
```

合并后的模型推理速度与原始模型完全相同，没有额外开销。或者也可以保持 LoRA adapter 单独存储，在推理时动态加载——这对于服务多个不同微调版本很有用。

## 总结

LoRA 和 QLoRA 让 LLM 微调变得触手可及。一张消费级 GPU 就能微调一个 7B 甚至 13B 的模型。选择正确的微调策略、准备高质量的数据、合理设置超参数，你就能在特定任务上获得远超通用模型的效果。

## References

1. Hu, E., et al. (2021). "LoRA: Low-Rank Adaptation of Large Language Models."
2. Dettmers, T., et al. (2023). "QLoRA: Efficient Finetuning of Quantized Language Models."
3. Hugging Face PEFT Documentation.
