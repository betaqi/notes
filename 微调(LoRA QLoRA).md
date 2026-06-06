好问题。LoRA / QLoRA 是**轻量微调技术**,让普通人用一张消费级显卡就能"教"大模型新东西。

## 一、为什么需要 LoRA

先理解传统微调的痛点。

一个 7B 模型有 70 亿参数,**全量微调**意味着要更新全部 70 亿参数。这带来三个问题:

1. **显存爆炸**:训练时除了存权重,还要存梯度、优化器状态(Adam 优化器要存 2 倍权重),7B 模型全量微调要 80GB+ 显存,普通人玩不起。
2. **存储灾难**:每微调一个版本就是一份完整权重(14GB+),想要 10 个领域专家就要存 140GB。
3. **过拟合**:小数据集去更新所有参数,容易"学坏"。

**核心洞察**:微调时模型权重的变化其实是"低秩"的——也就是说,变化的部分可以用一个很小的矩阵来近似表达。既然变化很小,何必动整个模型?

## 二、LoRA 的核心思想

LoRA = **Lo**w-**R**ank **A**daptation(低秩适配)。2021 年微软提出。

### 一句话原理

冻结原模型,在旁边加一个**小补丁**,只训练这个补丁。

### 数学直觉(不怕数学的话)

原模型某一层的权重是矩阵 W(假设 4096×4096,有 1600 万个数)。 全量微调:直接改 W,变成 W + ΔW,ΔW 也是 4096×4096。 LoRA 的发现:这个 ΔW 其实可以拆成两个瘦长矩阵的乘积:

```
ΔW (4096×4096) ≈ A (4096×8) × B (8×4096)
   1600 万个参数         总共  6.5 万个参数
```

这里的 `8` 叫 **rank(秩)**,通常取 4 / 8 / 16 / 32 / 64。 参数量直接降了 **245 倍**,但效果接近全量微调。

### 推理时怎么用

```
y = W·x + (A·B)·x
    ↑          ↑
   原模型    LoRA 补丁
```

训练完后,你可以:

- **挂着用**:推理时把 LoRA 补丁加载到原模型旁边,动态参与计算
- **合并掉**:把 A·B 算出来加到 W 里,变成一个新的全量模型,推理无额外开销

## 三、QLoRA 又是什么

QLoRA = **Q**uantized **LoRA**(量化 + LoRA),2023 年华盛顿大学提出。

### 它解决了什么

LoRA 虽然只训练补丁,但**原模型还得加载到显存里参与前向计算**。7B 模型 FP16 加载也要 14GB,加上训练开销,24GB 显卡都吃力,70B 模型基本免谈。

### QLoRA 的做法

1. 把原模型量化到 **4-bit**(NF4 格式,专门为 QLoRA 设计的精度)
2. 量化后的权重只读、不训练
3. LoRA 补丁仍然用 FP16 训练
4. 计算时临时把 4-bit 反量化回去,算完丢弃

效果对比(以 7B 模型微调为例):

|方法|显存需求|训练速度|效果|
|---|---|---|---|
|全量微调|80 GB+|基准|最好|
|LoRA|20 GB|略快|接近全量|
|QLoRA|**6-8 GB**|略慢|接近 LoRA|

**结论**:QLoRA 让 24GB 显卡能微调 33B 模型,48GB 能微调 70B,普通人在家就能玩。

## 四、关键超参数(实操要点)

如果你要动手微调,以下几个参数最常见:

**rank (r)** LoRA 补丁的"宽度"。常用 8 / 16 / 32 / 64。越大越能学到复杂的东西,但也更容易过拟合。一般任务 8-16 够用,复杂任务上 32-64。

**alpha** 缩放系数,控制 LoRA 补丁的影响力。经验法则:`alpha = 2 × r`。

**target_modules** 给哪些层挂补丁。常见选择:

- 只挂注意力层(`q_proj, v_proj`)→ 参数最少
- 全部线性层(`q,k,v,o,gate,up,down`)→ 效果最好,推荐

**learning_rate** 通常 1e-4 到 3e-4,比全量微调高一个数量级。

**dropout** LoRA 通常用 0.05-0.1 防过拟合。

## 五、实战:用 Unsloth 微调一个 7B 模型

最简单的工具是 **Unsloth**,2-5 倍训练加速,显存还省一半。

```bash
pip install unsloth
```

最小可运行代码:

```python
from unsloth import FastLanguageModel
from datasets import load_dataset
from trl import SFTTrainer
from transformers import TrainingArguments

# 1. 加载 4-bit 量化的基础模型(QLoRA)
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Qwen2.5-7B-Instruct",
    max_seq_length=2048,
    load_in_4bit=True,
)

# 2. 给模型装上 LoRA 补丁
model = FastLanguageModel.get_peft_model(
    model,
    r=16,                    # rank
    lora_alpha=32,           # alpha = 2r
    target_modules=["q_proj","k_proj","v_proj","o_proj",
                    "gate_proj","up_proj","down_proj"],
    lora_dropout=0.05,
)

# 3. 准备数据(只要是聊天格式的 jsonl 都行)
dataset = load_dataset("json", data_files="my_data.jsonl", split="train")

# 4. 训练
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        num_train_epochs=3,
        learning_rate=2e-4,
        output_dir="my_lora",
    ),
)
trainer.train()

# 5. 保存(只保存 LoRA 补丁,几十 MB)
model.save_pretrained("my_lora_adapter")
```

数据格式:

```json
{"messages": [
  {"role": "user", "content": "公司退货政策是什么?"},
  {"role": "assistant", "content": "我们支持 30 天无理由退货……"}
]}
```

几百到几千条这样的样本就能训出有效果的领域微调。

## 六、什么时候该用 LoRA / QLoRA

**适合**

- 让模型懂某个垂直领域(法律、医疗、客服话术)
- 模仿特定风格(自家品牌的写作语气)
- 让模型稳定输出特定格式(JSON、表格)
- 教会模型新的任务模式

**不适合**

- 注入大量新知识(LoRA 学知识效果一般,RAG 更合适)
- 需要根本改变模型行为(可能要全量微调或继续预训练)
- 数据量极大(几百万条以上,LoRA 容量可能不够,加大 rank 或上全量)

## 七、和其他名词的关系

|术语|是什么|
|---|---|
|PEFT|Parameter-Efficient Fine-Tuning,参数高效微调的总称,LoRA 是其中一种|
|Adapter|早期的参数高效方法,LoRA 算改进版|
|Prefix Tuning / Prompt Tuning|另一类 PEFT,在输入端加可训练 token,效果一般不如 LoRA|
|DoRA|LoRA 改进版,2024 年提出,精度略好|
|LoftQ|QLoRA 的改进,初始化更精准|
|Hugging Face PEFT 库|实现 LoRA / QLoRA 等方法的官方库|

## 八、和"格式"维度的关系(回到上次的问题)

LoRA 训练完得到一个 **adapter**(几十 MB ~ 几 GB),它**不是完整模型**,使用时有两条路:

1. **挂着用**:基础模型 + LoRA adapter 一起加载
    
    - Hugging Face 上 `xxx-lora` 后缀的就是
    - 推理框架支持:vLLM、SGLang、Hugging Face Transformers
2. **合并 + 重新打包**:把 LoRA 合并进基础模型,导出成 GGUF / AWQ
    
    - 这就和我们之前讲的"格式维度"对接上了
    - 合并后变成一个独立模型,部署到 Ollama 等

所以一条完整链路是:

```
Base 模型 → SFT → Instruct
                    ↓
                  LoRA 微调(你的领域数据)
                    ↓
              合并 + 量化打包(GGUF Q4_K_M)
                    ↓
                Ollama 部署
```

这就是个人玩家"训练 + 部署"的标准工作流。

---

要不要我把这一章也补到文档里,作为第 9 章微调小节的展开?