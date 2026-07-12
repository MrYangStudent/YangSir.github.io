---
title: "从零微调大模型：用 Qwen2.5-7B 训练一个会写小说的 AI"
date: 2026-07-12
draft: false
tags:
  - LLM
  - Fine-tuning
  - QLoRA
  - LoRA
  - NLP
  - PyTorch
  - 中文小说
  - 深度学习
summary: "零基础友好的 LLM 微调实战指南。从 CUDA 环境搭建到模型推理，手把手教你用 QLoRA 微调 Qwen2.5-7B，训练一个会续写中文小说的 AI。"
ShowToc: true
TocOpen: true
---

## 先说个故事

去年有朋友问我："你看那些 AI 写的小说，挺像那么回事的。我自己能不能也搞一个？"

我当时回答："能，但你得先有张好显卡，然后学一堆东西。"

后来我发现这个回答有问题。它把一个"能做"的事情说成了"很难"的事情。

所以有了这篇文章。我会用一个真实跑通的项目，带你从零开始，一步步微调一个会续写中文小说的 AI。不需要你懂 CUDA，不需要你写过 PyTorch。你只需要知道——这件事可以做到，而且没那么难。

---

## 微调到底是什么？和训练一个新模型有什么区别？

你可能听过"预训练"（Pre-training）和"微调"（Fine-tuning）这两个词，但一直不太确定它们的关系。

打个比方。

**预训练**相当于把一个小孩送进学校，学语文、数学、英语、科学……什么都学。这个过程极其烧钱，GPT-4 级别的预训练成本是上亿美元级别。学完之后，这个小孩就有了"通识能力"——你问他什么，他都能说上几句。

**微调**相当于这个小孩已经毕业了，现在你想让他专攻一门手艺——比如写小说。你不需要让他重新上小学，只需要给他看大量小说范例，告诉他"这种写法是对的"，他很快就能学会。成本也从"上亿美元"变成了"家里一台游戏电脑加几小时电费"。

所以，**微调是在一个已经"博学"的模型基础上，用少量领域数据让它专精某个方向**。

### 那 SFT 又是什么？

你可能还会看到 SFT（Supervised Fine-Tuning，监督微调）这个词。简单来说，SFT 就是微调的一种具体做法——**你给模型看"输入+标准答案"的配对数据，让它照着学**。

比如：

```
输入：张三推开房门，发现屋里一片狼藉。
标准答案：地上散落着碎玻璃，抽屉被翻了个底朝天，墙上的挂钟歪到了一边——显然有人来过。
```

模型看到足够多的这种配对后，就学会了"看到一句小说开头，应该怎么合理续写下去"。

---

## 那 LoRA 和 QLoRA 又是啥？为什么需要它们？

好，现在问题来了。

Qwen2.5-7B 这个模型有 **70 亿个参数**，如果用 fp16 精度加载，光模型本身就要占 **~15GB 显存**。再加上优化器状态、梯度、激活值……训练时需要的内存远超这个数。一张普通的 RTX 4090（24GB 显存）根本装不下。

这就是 LoRA 和 QLoRA 登场的原因。

### LoRA：只学"增量"，不碰本体

LoRA 的核心思想用一句话概括：**不修改原模型的任何参数，只在旁边挂两个小矩阵，只训练这两个小矩阵。**

如果你玩过游戏 Mod，LoRA 就是"游戏 Mod"——原游戏文件一个不动，你的所有修改都存在一个单独的 mod 文件夹里。想还原？直接删掉 mod 文件夹就行。

数学上不展开了，但你只需要知道一个关键数字：原本需要训练 70 亿个参数，LoRA 只需要训练 **几百万到几千万个**——少了 99% 以上。

### QLoRA：先把模型"压缩"，再挂 LoRA

LoRA 虽然减少了训练参数量，但原模型本身还是 15GB——你还是要把它完整加载到显存里。

QLoRA 的解决思路很直接：**先把原模型从 16-bit 精度压缩到 4-bit，再挂 LoRA 训练。**

具体怎么做呢？它用了两个技术：

1. **NF4（NormalFloat4）量化**：不是简单粗暴地把小数点后全砍掉，而是用了一种专门为神经网络权重分布设计的数据格式，信息损失比普通 4-bit 小得多。
2. **双重量化**：对量化过程中的"缩放因子"再做一次量化，进一步省显存。听起来很绕，但效果就是"再省 0.4GB"。

最终效果：**7B 模型从 15GB 压缩到 ~5GB，加上 LoRA 适配器和训练开销，总共 ~12GB 显存就能跑。**

一张 RTX 4070（12GB）刚好够用，RTX 4090（24GB）绰绰有余。

---

## 显存到底要多少？一个简单公式

如果你想精确估算，这里有一个实用公式（来源：[QLoRA 论文](https://arxiv.org/abs/2305.14314)及社区验证）：

```
训练总显存 ≈ 模型加载量 + 优化器状态 + 梯度 + 激活值

其中：
- 模型加载量（4-bit）：参数量 × 0.5 GB（每10亿参数约0.5GB）
- 模型加载量（fp16）：参数量 × 2 GB
- LoRA 可训练参数：几乎可忽略（< 0.5GB）
- 优化器 + 梯度：约为可训练参数的 3-4 倍
```

拿 7B 模型 + QLoRA 为例：

| 项目 | 显存占用 |
|------|----------|
| 4-bit 基座模型 | ~5 GB |
| LoRA 适配器 | ~0.2 GB |
| 优化器状态 + 梯度 | ~1 GB |
| 激活值（batch=2, seq=1024） | ~5-6 GB |
| **合计** | **~12 GB** |

这就是为什么 12GB 显存是 QLoRA + 7B 模型的最低门槛。

---

## 五个技能的完整学习路径

这篇文章对应一个真实可跑的实战项目，它由五个技能串联成一条完整链路。按执行顺序：

```
┌──────────────────────────────────────────────────────────────┐
│                    完整微调学习路径                              │
├────────┬──────────────────────┬───────────────────────────────┤
│  顺序   │       技能            │          做什么                │
├────────┼──────────────────────┼───────────────────────────────┤
│   0    │ cuda-environment-setup│ CUDA + PyTorch 环境搭建        │
│   1    │ model-vram-selector  │ 根据显存选最优模型和量化方案     │
│   2    │ finetune-method-selector│ LLaMA-Factory/peft/Unsloth 三选一 │
│   3    │ novel_finetune (核心) │ 端到端实战：数据→训练→推理      │
│  3.5   │ data-cleaning-auditor│ 训练数据四维度质量审核           │
└────────┴──────────────────────┴───────────────────────────────┘
```

下面我们逐个走一遍。每个环节我会挑最关键的部分讲——你不需要把所有代码背下来，理解"输入→做了什么→输出"就够了。

---

## 第零步：CUDA 环境搭建

这是最容易被忽视、也最容易出问题的一步。

很多人兴冲冲装完 PyTorch，跑一段代码，结果报错：`CUDA not available` 或者 `torch not compiled with CUDA support`。一查发现 PyTorch 版本和 CUDA 版本不匹配。

关键原则就一条：**先确定你的 CUDA 版本，再装对应版本的 PyTorch，不要反过来。**

```bash
# 先查你的 CUDA 版本
nvidia-smi

# 比如输出显示 CUDA 12.6，那就去 PyTorch 官网找对应的安装命令
# https://pytorch.org/get-started/locally/
# 通常长这样：
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

如果你懒得自己查，项目里的 `01_environment_check.py` 会自动帮你诊断一切：

```python
# 它会检查并输出：
# ✓ Python 版本是否 >= 3.8
# ✓ CUDA 是否可用
# ✓ GPU 型号、显存大小、计算能力
# ✓ PyTorch 版本与 CUDA 的兼容性
# ✓ 关键依赖包是否安装
# ✓ 磁盘空间是否够用（模型 ~15GB + 数据 ~3GB）
# → 最后根据你的显存推荐最合适的微调方案
python scripts/01_environment_check.py
```

**常见踩坑点**：Windows 上装 CUDA Toolkit 时别装错版本。如果你用的是 Anaconda，用 conda 创建独立环境可以避免很多依赖冲突：

```bash
conda create -n emollmlearn python=3.10
conda activate emollmlearn
```

---

## 第一步：根据显存选模型

不同 GPU 能跑的模型天差地别。项目里 `model-vram-selector` 给了一个实用分级表：

| 显存 | 能做什么 | 推荐方案 |
|------|----------|----------|
| < 8GB | QLoRA + ≤3B 模型 | Qwen2.5-3B + 4-bit |
| 8-12GB | QLoRA + 7B 模型 | Qwen2.5-7B + 4-bit |
| 12-16GB | QLoRA + 7B（余量充足） | Qwen2.5-7B + 4-bit |
| 16-24GB | QLoRA + 14B 或 LoRA(fp16) + 7B | 按需选择 |
| 24GB+ | 更大模型或全量微调 | 看预算 |

本项目使用的是 **Qwen2.5-7B-Instruct + QLoRA 4-bit**，RTX 2080（8GB 显存）不够，需要至少 12GB。

为什么选 Qwen2.5-7B-Instruct 而不是别的模型？三个原因：

1. **中文能力第一梯队**：Qwen 系列在中文基准测试中表现极好，尤其适合中文创作
2. **Instruct 版本**：已经经过指令微调，能理解"请续写"这种指令，不是你凭空训练
3. **社区生态好**：LLaMA-Factory、Unsloth 都原生支持，下载也方便（ModelScope 国内加速）

模型下载推荐用 ModelScope（魔搭社区）而非 HuggingFace——在国内，前者的下载速度是后者的 50-500 倍：

```python
# 用 ModelScope SDK 下载，10-50 MB/s
from modelscope import snapshot_download
model_dir = snapshot_download('qwen/Qwen2.5-7B-Instruct')
```

---

## 第二步：选微调框架

市面上有很多微调框架，但新手真正需要纠结的其实就三个：

| 框架 | 特点 | 适合谁 |
|------|------|--------|
| **LLaMA-Factory** | Web UI + CLI，配置即训练，几乎不用写代码 | 新手首选，想快速出结果 |
| **peft + trl** | HuggingFace 官方生态，代码灵活 | 想深入理解每个步骤 |
| **Unsloth** | 2-3x 训练加速，显存省 50%+ | 显卡小但想跑快 |

三个方案项目里都提供了完整代码，你可以先跑 LLaMA-Factory 快速出结果，再研究 peft+trl 理解原理。

---

## 第三步：数据准备——最耗时但最重要的环节

如果说模型是引擎，数据就是汽油。汽油不好，引擎再好也跑不远。

### 原始数据

项目用了 **30+ 部中文网络小说**，按 11 个类别分目录存放：

```
data/raw/
├── 武侠仙侠/    (3部，~225MB)
├── 玄幻小说/    (5部，~228MB)
├── 科幻小说/    (3部，~39MB)
├── 都市小说/    (3部，~151MB)
├── 青春校园/    (2部，~145MB)
├── 军事历史/    (3部，~44MB)
├── 灵异推理/    (3部，~124MB)
├── 网游竞技/    (6部，~284MB)
├── 管理哲学/    (3部，~36MB)
├── 纪实文学/    (3部，~85MB)
└── 耽美小说/    (暂无)
```

你可能想问：为什么选网文而不是经典文学？

因为目标是 **续写网文风格的小说**。如果你拿《红楼梦》去训练，模型学到的就是"半文半白"的语感，续写出来的东西跟网文完全不搭。**训练数据决定了模型风格，这是微调最重要的直觉之一。**

### 清洗流水线

原始小说文本有多脏？广告、乱码、重复段落、格式混乱……直接喂给模型只会让它学到垃圾。

数据清洗脚本 `02_data_cleaning.py` 做了这么几件事：

```python
# 清洗流程（6步流水线）
raw_files = glob("data/raw/*/*.txt")        # 1. 读取所有原始小说
sentences = read_and_split(raw_files)        # 2. 按句切分（处理中英文标点）
deduped = md5_dedup(sentences)              # 3. MD5去重（不同小说可能重复）
cleaned = filter_noise(deduped)             # 4. 过滤乱码/广告/无效内容
labeled = classify(cleaned)                  # 5. 按11类关键词自动标注类别
train_data = to_alpaca_format(labeled)      # 6. 转成Alpaca训练格式
```

最终产出 1103 万条清洗后的句子，约 2.85GB。

### 关键设计：双层训练数据

这是本项目最巧妙的设计——数据不是简单的一条条独立句子，而是分了两层：

**句子级**：用前 1-3 句预测下一句。
```
输入：他推开门，一股霉味扑面而来。屋里没有开灯，只有窗帘缝隙透进来的微光。
输出：他摸索着找到墙上的开关，啪嗒一声——灯没亮。
```

**章节级**：用前 2 章预测下一章。
```
输入：[第一章全文] + [第二章全文]
输出：[第三章全文]
```

为什么要分两层？

句子级数据教会模型**句间连贯性和文笔风格**。怎么断句、用什么修辞、对话怎么组织——这些都是短程依赖。

章节级数据教会模型**长程情节发展**。人物怎么成长、悬念怎么铺设、冲突怎么升级——这些需要看到更大的上下文。

训练时大概 8:2 混合，相当于"先练好基本功，再学高级技巧"。

### 数据审核

训练前一定要审核数据质量。项目的 `04_data_audit.py` 会检查四个维度：

- **类别分布**：11 个类别是否均衡？（防止模型偏科）
- **长度统计**：input/output 是否在合理范围？（防止截断过多）
- **格式完整性**：JSON 有没有格式错误？
- **数据泄漏**：train/val 是否有重叠？（训练集和验证集必须严格分开）

---

## 第四步：训练配置——每个参数在干什么

训练是整个流程里看起来最"黑盒"的部分——跑起来就是一行行的 loss 数字。但配置里的每一个参数都有它的道理。我挑最重要的几个解释。

### 核心超参数

```yaml
# 模型加载
model_name: Qwen2.5-7B-Instruct  
quantization: 4-bit              # 量化到 4-bit（QLoRA）
double_quantization: true        # 双重量化，再省 ~0.4GB
bnb_4bit_compute_dtype: fp16      # 计算时转回 fp16，保证精度

# LoRA 配置
lora_r: 8                        # LoRA 秩（rank），越大可学的东西越多，但也越慢
lora_alpha: 16                   # 缩放因子，通常取 r 的 2 倍
lora_target: [q_proj, v_proj]    # 只在 Q 和 V 投影上加 LoRA（Attention 的核心）
lora_dropout: 0.1                # 随机丢弃 10%，防止过拟合

# 训练配置
per_device_batch_size: 2         # 每次放 2 条数据进 GPU
gradient_accumulation: 4         # 累计 4 次再更新 → 有效 batch size = 2×4 = 8
learning_rate: 5e-5              # 学习率，QLoRA 推荐 1e-4~5e-5
num_epochs: 3                    # 把全部数据过 3 遍
max_seq_length: 1024             # 每条最多 1024 个 token（约 700-800 个汉字）
lr_scheduler: cosine             # 余弦学习率衰减：前期大步、后期小步
```

你可能想问："每个参数都调吗？"

先说结论：**新手别调，先用默认值跑通。** 这几个值经过大量社区验证，对 QLoRA + 7B 基本是通用配置。等你跑出 baseline 了，再考虑调 `lora_r`、`learning_rate` 这几个影响最大的。

### 为什么只训练 Q 和 V 投影？

Transformer 的注意力机制有四个关键投影：Q（Query，查询）、K（Key，键）、V（Value，值）、O（Output，输出）。

研究表明，**只训练 Q 和 V 就能达到训练全部投影 90%+ 的效果**，但参数量少了一半。这是业界的常用"免费午餐"——效果好，还省显存。

当然，Unsloth 方案更激进，它默认训练全部 7 个线性层（Q/K/V/O + gate/up/down），因为它的优化做得足够好，显示训练全部也不会慢多少。

### 让模型只学"回答"，不学"问题"

有一个很容易忽视的技术细节：

Alpaca 格式的训练数据包含 `instruction`（指令）、`input`（上文）和 `output`（续写）。训练时，我们只想让模型学习 `output` 部分——那些"请根据上文续写"的指令词，模型早就懂了，不需要再学一遍。

```python
# 关键代码：只计算 output 部分的 loss
from trl import DataCollatorForCompletionOnlyLM

collator = DataCollatorForCompletionOnlyLM(
    response_template="### 回答:",  # 从这个标记之后才开始计算 loss
    tokenizer=tokenizer
)
```

这听起来像细节，但对训练质量影响很大。如果不做这个处理，模型会把宝贵的"学习容量"浪费在重复学习指令格式上。

---

## 第五步：开始训练

三种方案都可以一键启动。

### 方案 A：LLaMA-Factory（最省心）

```bash
cd llama_factory
llamafactory-cli train train_qwen2_lora.yaml
```

所有配置都在 YAML 文件里，改参数 = 改 YAML，不用动代码。

训练期间可以用 `monitor.bat` 实时看 GPU 状态：
```
+-------------------------------------------------------+
| GPU  名称                  显存使用   利用率   温度     |
|   0  NVIDIA GeForce RTX 2080  7645MiB /  8192MiB  92%  68°C  |
+-------------------------------------------------------+
```

### 方案 B：peft + trl（最能学东西）

```bash
cd peft_trl
python train.py --model_name /path/to/Qwen2.5-7B-Instruct
```

代码里每个步骤都写了注释。如果你想深入理解"训练循环到底在干什么"，看这个方案。

### 方案 C：Unsloth（最快）

```bash
cd unsloth
python train.py --model_name /path/to/Qwen2.5-7B-Instruct
```

Unsloth 通过手写 CUDA 内核优化了注意力计算和 MLP，速度是方案 A/B 的 2-3 倍，显存占用少 50-70%。如果你显卡小（12-16GB），优先用它。

### 训练时间预估

| 方案 | GPU | 数据量 | 预估时间 |
|------|-----|--------|----------|
| LLaMA-Factory | RTX 2080 (8GB) | 1100万句 | ❌ 显存不够 |
| LLaMA-Factory | RTX 4090 (24GB) | 1100万句 | 6-10 小时 |
| Unsloth | RTX 4090 (24GB) | 1100万句 | 3-5 小时 |
| Unsloth | RTX 4070 (12GB) | 1100万句 | 5-8 小时 |

如果只是跑通流程、验证效果，**建议先只用 10% 的数据**（修改数据加载时的采样比例），30 分钟就能出结果。

### 断点续训

训练中途断电？别慌。项目配置了自动保存 checkpoint（每 500 步一次），下次启动会自动从最新 checkpoint 恢复：

```yaml
save_steps: 500                   # 每 500 步保存一次
save_total_limit: 3               # 只保留最近 3 个 checkpoint
resume_from_checkpoint: true      # 自动恢复
```

---

## 第六步：推理测试——让 AI 真的"写"给你看

训练完成后，你得到了一个 LoRA 适配器（只有几十 MB）。把它"挂"到基座模型上就能用了。

```python
# 加载基座模型（4-bit）+ LoRA 适配器
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

base_model = AutoModelForCausalLM.from_pretrained(
    "Qwen2.5-7B-Instruct",
    load_in_4bit=True,                       # 4-bit 量化加载
    torch_dtype=torch.float16,
    device_map="auto"
)
model = PeftModel.from_pretrained(
    base_model,
    "./output/checkpoint-1000"               # LoRA 适配器路径
)
model.eval()

# 构造推理 prompt
prompt = """请根据上文，续写下一句话：
张三站在悬崖边，晚风吹动他的衣角。"""

inputs = tokenizer(prompt, return_tensors="pt").to("cuda")
outputs = model.generate(
    **inputs,
    max_new_tokens=200,        # 最多生成 200 个 token
    temperature=0.8,           # 0.8 适中，既不太死板也不太随机
    top_p=0.9,                 # nucleus sampling
    do_sample=True
)
print(tokenizer.decode(outputs[0]))
```

你可能想问：temperature 和 top_p 怎么调？

简单指南：
- **temperature=0.1**：几乎每次输出相同（适合翻译、摘要）
- **temperature=0.8**：有一定随机性，但不离谱（适合续写、对话）
- **temperature=1.5**：天马行空，完全放飞（适合 brainstorming）
- **top_p=0.9**：从概率累计 90% 的词里随机选，大多数情况不需要改

### 对比测试：三个方案效果一样吗？

项目的 `07_comparison_test.py` 会自动用同一段 prompt 跑三个方案的模型，给出对比结果：

```python
# 输入同一段小说开头，各方案分别续写
# → 输出字数、推理耗时、显存峰值
# → 保存为 comparison_results.json
python scripts/07_comparison_test.py --auto
```

结论通常是：**三个方案在生成质量上差异极小**（毕竟底层都是 Qwen2.5-7B）。差异主要体现在训练速度和显存使用上——Unsloth > LLaMA-Factory > peft+trl。

---

## 训练效果怎么评价？

模型微调没有"正确答案"。对于一个写小说的 AI，效果好坏很大程度上是主观感受。但有几个客观指标可以参考：

### 1. Loss 曲线

训练 loss 持续下降且没有剧烈波动 = 配置基本正确。验证 loss 下降但最终趋于平缓 = 可以考虑停止训练。

如果验证 loss 开始上升而训练 loss 还在下降 → **过拟合了**，减少 epoch 数或增大 dropout。

### 2. 人工评估

把你的测试 prompt 同时输入原始 Qwen2.5-7B 和微调后的模型，对比输出：

- **文笔风格**：微调后是否更接近网文风格？
- **连贯性**：续写是否和前文衔接自然？
- **内容质量**：有没有逻辑矛盾或重复内容？
- **多样性**：不同的 prompt 是不是产生了不同的续写？（还是动不动就"嘴角勾起一抹微笑"）

### 3. 分类平衡性

11 个类别各抽 10 条 prompt 测试，看看有没有某些类别（比如"玄幻小说"）由于数据量更多而导致生成质量明显更好。如果有——下个版本就该调整数据采样策略了。

---

## 常见坑点速查

| 坑 | 现象 | 解决 |
|----|------|------|
| CUDA 版本不匹配 | `CUDA not available` | `nvidia-smi` 看版本后重装 PyTorch |
| 显存不够 (OOM) | `CUDA out of memory` | 减小 batch_size，增大 gradient_accumulation |
| 数据格式不对 | 训练 loss 不收敛 | 检查 JSON 字段是否与 Alpaca 格式一致 |
| 编码问题 | 中文变乱码 | 统一用 UTF-8 保存所有文件 |
| Windows 路径问题 | 路径中有空格或中文 | 避免中文路径，或用 `r""` 原始字符串 |
| 训练太慢 | 一个 epoch 跑一天 | 先用 10% 数据做实验，确认没问题再全量跑 |
| 模型输出重复 | 反复生成同一句话 | 增大 temperature，加 repetition_penalty |
| 忘记只计算 output loss | loss 收敛但效果差 | 检查是否用了 `DataCollatorForCompletionOnlyLM` |
| checkpoint 太大 | 每个 10GB+ | 确认只保存 adapter 权重而非完整模型 |

---

## 回到最初的问题：这件事你能做吗？

我们回到开篇那个问题。

**硬件门槛**：一张 12GB 显存以上的 NVIDIA 显卡。RTX 4070、4080、4090 都行，二手 3090 也行。如果没有，Google Colab 免费版只有 T4（16GB），刚好够用。

**知识门槛**：看完这篇文章，你应该已经理解了核心概念。接下来就是——

1. 搭好环境（跟着 `01_environment_check.py` 走）
2. 准备好数据（参照 `02_data_cleaning.py` 的处理逻辑）
3. 选一个方案跑起来（LLaMA-Factory 最快出结果，Unsloth 最省显存）
4. 拿到第一个 checkpoint 后赶紧测一测（能看到自己训练的 AI 写出第一句话的快感，是支撑你走下去的最大动力）

**时间投入**：环境搭建半天，数据准备半天（取决于原始数据质量），训练一晚上——加起来一个周末够用。

最后说一句。

AI 微调曾经是实验室里的高端操作。但 QLoRA 出现后，这件事已经彻底民主化了。现在能阻止你做这件事的，只有"我不知道原来我也可以"这一念之差。

而你已经看完这篇文章了。接下来，就差动手了。

---

*项目地址：[novel_finetune](https://github.com/MrYangStudent/novel_finetune)（跟随 `llm-full-stack` 技能体系发布）*

*如果你在跑这个项目时遇到任何问题，欢迎在 GitHub Issues 留言。*
