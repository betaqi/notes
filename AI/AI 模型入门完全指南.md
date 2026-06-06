

> 面向零基础的渐进式教程。从"模型是什么"一路讲到"在自己电脑上跑起来"。
> 阅读建议:按章节顺序读,遇到不懂的术语先跳过,最后看附录的术语表。

---

## 目录

1. 第 1 章 · 什么是 AI 模型(从直觉开始)
2. 第 2 章 · 模型是怎么"造"出来的:训练流程
3. 第 3 章 · 模型家族图谱:语言、多模态、专用模型
4. 第 4 章 · 看懂模型名字:后缀和标签全解
5. 第 5 章 · 量化是什么:为什么 GGUF / AWQ / FP8 这么多
6. 第 6 章 · 推理工具生态:Ollama / LM Studio / vLLM 怎么选
7. 第 7 章 · 实战一:在自己电脑上跑第一个模型
8. 第 8 章 · 实战二:用 API 接入应用
9. 第 9 章 · 进阶玩法:RAG、微调、Agent
10. 第 10 章 · 学习路径与资源
11. 附录 · 术语表

---

## 第 1 章 · 什么是 AI 模型(从直觉开始)

### 1.1 一句话比喻

可以把"模型"想成一个**超级复杂的函数**。你给它输入(文字、图片、声音),它给你输出(文字、图片、声音)。

```
输入 → [ 模型 ] → 输出
"今天天气真" → [ 语言模型 ] → "好"
```

这个函数里面有几亿到几千亿个"旋钮"(叫**参数**,英文 parameter)。这些旋钮的具体数值,就决定了模型的能力。

### 1.2 参数和模型大小

参数量常见单位:
- M = Million(百万)
- B = Billion(十亿)

例子:
- Qwen3-0.6B = 6 亿参数,手机能跑
- Llama-3.1-8B = 80 亿参数,普通游戏显卡能跑
- DeepSeek-V3-671B = 6710 亿参数,要服务器集群

**经验公式**:用 FP16(半精度)存,显存需求 ≈ 参数量 × 2 GB。
- 8B 模型 ≈ 16 GB 显存
- 70B 模型 ≈ 140 GB 显存

后面讲的"量化"就是把这个数字打折,让小机器也能跑大模型。

### 1.3 模型 ≠ 应用

很多人把 ChatGPT 和 GPT-4 混为一谈。其实:
- **GPT-4** 是模型(那个函数本身)
- **ChatGPT** 是应用(网页 + 模型 + 系统提示词 + 工具调用)

你下载到本地的,只是模型权重文件。要让它"会聊天",还需要一个推理程序(Ollama / LM Studio 等)给它套个壳。

### 1.4 模型权重文件长什么样

打开 Hugging Face 上一个模型仓库,常见文件:

```
config.json              ← 模型结构说明
tokenizer.json           ← 分词器(把文字切成 token)
model-00001-of-00004.safetensors  ← 权重(分片存储)
model-00002-of-00004.safetensors
...
```

`.safetensors` 是新一代安全格式,`.bin` / `.pt` 是旧的 PyTorch 格式。`.gguf` 是 llama.cpp 用的单文件格式,后面会讲。

---

## 第 2 章 · 模型是怎么"造"出来的:训练流程

理解训练流程,才能看懂为什么会有 Base、Instruct、Chat、DPO 这些版本。

### 2.1 三个阶段

```
阶段一:预训练 (Pretraining)
  数据:整个互联网级别的文本(几万亿 token)
  目标:学会"预测下一个词"
  产物:Base 模型(只会续写,不会聊天)
  耗时:几个月,几千张 H100 显卡,成本几百万到几千万美元

      ↓

阶段二:监督微调 (SFT, Supervised Fine-Tuning)
  数据:人工写的高质量"问答对",几万到几百万条
  目标:学会按指令回答,而不是无脑续写
  产物:Instruct 模型 / SFT 模型
  耗时:几天到几周

      ↓

阶段三:偏好对齐 (RLHF / DPO 等)
  数据:人类对多个回答的排序("A 比 B 好")
  目标:更符合人类偏好,更有用、更无害
  产物:Chat 模型 / RLHF 模型 / DPO 模型(成品,可直接对话)
  耗时:几天到几周
```

### 2.2 类比:培养一个学生

- **预训练** = 让他从小博览群书,知识量爆炸,但不知道怎么"做题"
- **SFT** = 老师教他做题套路,看了一堆例题,会按规矩答题了
- **RLHF/DPO** = 通过模拟考试和反馈,让他答得更让人满意

所以你看到的版本后缀对应:
- `Base` / `Pretrained` → 阶段一产物,不会聊天,适合二次开发
- `SFT` / `Instruct` / `IT` → 阶段二产物,能听话
- `Chat` / `RLHF` / `DPO` → 阶段三产物,日常使用首选

### 2.3 几种新名词速通

**蒸馏 (Distill)**
让小模型模仿大模型的输出。例如 `DeepSeek-R1-Distill-Qwen-7B`,意思是"用 R1 当老师,把 Qwen-7B 训练得像 R1"。好处:小模型也能继承推理能力。

**LoRA / QLoRA**
微调时不改全部参数,只训练一个小"补丁"(几十 MB 到几 GB)。普通人在家用一张消费级显卡就能微调大模型。QLoRA 在 LoRA 基础上加了量化,更省显存。

**MoE (Mixture of Experts,混合专家)**
模型里有很多"专家",每次推理只激活其中几个。例如 `Qwen3-235B-A22B`,总参数 235B 但每次只用 22B 的算力。优点:省算力。缺点:显存照吃 235B。

**推理模型 (Reasoning Model)**
比如 OpenAI o1、DeepSeek-R1、QwQ。回答前会先"想"一长串(叫 Chain-of-Thought),再给最终答案。数学和编程任务特别强。

---

## 第 3 章 · 模型家族图谱

### 3.1 按"能干什么"分类

```
                       AI 模型
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
   语言模型           多模态模型          专用模型
   (LLM)             (VLM/Omni)        (Embedding 等)
       │                 │                 │
   ChatGPT             GPT-4o          Embedding
   Claude              Gemini          Reranker
   Llama               Qwen-VL         代码模型
   Qwen                InternVL        数学模型
   DeepSeek            CosyVoice       图像生成
                                        视频生成
```

### 3.2 语言模型(LLM)主流家族

**闭源(只能通过 API 用)**
- GPT 系列(OpenAI):GPT-4o、o1、o3
- Claude 系列(Anthropic):Sonnet、Opus、Haiku
- Gemini 系列(Google)
- 文心一言、通义千问商业版、豆包等

**开源(可以下载权重)**
- **Llama 系列**(Meta):开源界标杆,Llama 3.1 / 3.2 / 3.3
- **Qwen 系列**(阿里):中文最强之一,Qwen2.5 / Qwen3
- **DeepSeek 系列**:V3(通用)、R1(推理)
- **Mistral 系列**(法国):Mistral、Mixtral(MoE)
- **Gemma 系列**(Google):轻量级
- **Phi 系列**(微软):小模型典范
- **Yi 系列**(零一万物)、**GLM 系列**(智谱)、**Baichuan**(百川)

### 3.3 多模态模型

按输入输出模态分:

| 类型 | 输入 | 输出 | 例子 |
|------|------|------|------|
| VLM(视觉语言) | 文+图 | 文 | Qwen2-VL、InternVL、LLaVA |
| 图像生成 | 文 | 图 | Stable Diffusion、FLUX、DALL·E |
| 视频生成 | 文/图 | 视频 | Sora、Runway、可灵、Veo |
| ASR(语音识别) | 音 | 文 | Whisper、SenseVoice |
| TTS(语音合成) | 文 | 音 | CosyVoice、ChatTTS、ElevenLabs |
| Omni(全模态) | 任意 | 任意 | GPT-4o、Qwen2.5-Omni |

### 3.4 专用模型

**Embedding 模型**
把文字变成向量(一长串数字),用于语义搜索和 RAG。
代表:`bge-large`、`text-embedding-3`、`gte`、`m3e`。
特点:体积小(几百 MB),非常快。

**Reranker(重排序)模型**
给一批候选结果按相关性打分,精度比 embedding 高但慢。
代表:`bge-reranker`、`Cohere rerank`。

**代码模型**
代码语料占比高,补全和生成代码更强。
代表:Qwen-Coder、DeepSeek-Coder、Code Llama、StarCoder。

**数学模型**
数学题强化训练。代表:Qwen-Math、DeepSeekMath、NuminaMath。

**科学模型**
- AlphaFold:蛋白质结构预测
- GraphCast:天气预报
- ESM:蛋白质语言模型

---

## 第 4 章 · 看懂模型名字

模型名其实是一串"标签",拆开看就懂。

### 4.0 先理清:四个独立维度(超级重要)

很多人把 `Instruct` 和 `GGUF` 当成同一类东西,其实它们属于**完全不同的维度**。读模型名要分四层来看:

```
   ┌──────────────────────────────────────────────┐
   │  维度 1:系列 + 规模    它是谁              │
   │  例:Qwen2.5-7B、Llama-3.1-70B              │
   ├──────────────────────────────────────────────┤
   │  维度 2:训练阶段       它会不会聊天        │
   │  例:Base / Instruct / Chat / DPO           │
   ├──────────────────────────────────────────────┤
   │  维度 3:格式           用哪个推理引擎加载  │
   │  例:原始(safetensors)/ GGUF / AWQ / MLX  │
   ├──────────────────────────────────────────────┤
   │  维度 4:量化精度       体积和质量平衡      │
   │  例:FP16 / FP8 / Q4_K_M / Q8_0             │
   └──────────────────────────────────────────────┘
```

**关键认知**:同一个模型可以同时存在多种格式版本,内容完全一样,只是打包方式不同:

```
Qwen2.5-7B-Instruct                 ← 原版(safetensors,Transformers 加载)
Qwen2.5-7B-Instruct-GGUF            ← GGUF 格式(Ollama / llama.cpp 用)
Qwen2.5-7B-Instruct-AWQ             ← AWQ 量化(vLLM 用)
Qwen2.5-7B-Instruct-GPTQ-Int4       ← GPTQ 4 位量化
Qwen2.5-7B-Instruct-MLX             ← Mac 专用格式
```

它们的"脑子"是同一个(都是 Qwen2.5-7B 的 Instruct 阶段产物),只是文件打包方式不同,适配不同的推理引擎。

**类比**:
- 维度 2(训练阶段)= 这部电影的内容(动作片 / 喜剧 / 纪录片)
- 维度 3(格式)= 它是 MP4 还是 MKV 还是 BluRay
- 维度 4(量化)= 是 4K 原盘还是 1080p 压缩版

**所以常见问法的正确回答**:

> 「GGUF 是哪个训练阶段?」
> → 错位了。GGUF 是格式,跟训练阶段平行。`Base-GGUF` 和 `Instruct-GGUF` 都合法,前者仍然不会聊天。

> 「我下载哪个版本?」
> → 先按维度 2 选 Instruct/Chat,再按维度 3+4 选适合你硬件的格式量化组合。

下面进入具体例子。

### 4.1 一个完整例子

`Qwen3-Coder-Next-30B-A3B-Instruct-FP8`

拆开:
- `Qwen3` → 系列名(通义千问第 3 代)
- `Coder` → 能力标签(代码专用)
- `Next` → 版本标签(下一代,可能是预览版)
- `30B` → 参数量(300 亿)
- `A3B` → MoE 激活参数(每次推理用 30 亿)
- `Instruct` → 训练阶段(已做指令微调,能聊天)
- `FP8` → 量化格式(8 位浮点)

### 4.2 标签清单速查

**系列 / 版本**
| 标签 | 含义 |
|------|------|
| V2 / V3 / 3.1 | 第几代 |
| Next / Preview / Beta | 预览版,可能不稳定 |
| RC | Release Candidate,接近正式版 |

**规模**
| 标签 | 含义 |
|------|------|
| 0.5B / 7B / 70B | 总参数量(B=十亿) |
| Mini / Small / Medium / Large | 规模档位 |
| A3B / A22B | MoE 模型的激活参数 |

**训练阶段**
| 标签 | 含义 | 能直接对话吗 |
|------|------|----------|
| Base / Pretrained | 仅预训练 | 不能,只会续写 |
| SFT | 监督微调 | 能,但可能粗糙 |
| Instruct / IT | 指令微调 | 能,日常使用首选 |
| Chat | 对话优化 | 能 |
| DPO / RLHF / ORPO | 偏好对齐 | 能,通常更好 |
| Distill | 蒸馏自更大模型 | 看具体 |

**能力 / 模态**
| 标签 | 含义 |
|------|------|
| VL / Vision | 看图(视觉语言) |
| Audio / ASR / TTS | 音频处理 |
| Omni / AnyToAny | 全模态 |
| Coder / Code | 代码专用 |
| Math | 数学专用 |
| Embedding | 向量化 |
| Reranker | 重排序 |
| Reasoning / Thinking / R1-style | 长链推理 |
| MoE | 混合专家架构 |

**上下文长度**
| 标签 | 含义 |
|------|------|
| 8K / 32K / 128K / 1M | 最大支持的 token 数 |

**量化(下一章详讲)**
| 标签 | 含义 |
|------|------|
| FP16 / BF16 | 半精度,原始 |
| FP8 | 8 位浮点,新硬件 |
| GPTQ / AWQ | GPU 量化 |
| GGUF | CPU/Mac 友好 |
| MLX | Apple Silicon 专用 |
| EXL2 / EXL3 | ExLlama 格式 |
| MNN / ONNX | 端侧部署格式 |

**特殊后缀**
| 标签 | 含义 |
|------|------|
| LoRA / Adapter | 微调"补丁",需配 base |
| GGUF-Q4_K_M | GGUF 4 位量化中等精度 |
| CustomVoice / VoiceClone | TTS 声音克隆版本 |

### 4.3 实战阅读练习

练一练,自己拆解下面这些名字:

1. `Llama-3.1-8B-Instruct`
   → Llama 第 3.1 代 + 8B + 已指令微调,可直接聊天

2. `DeepSeek-R1-Distill-Qwen-32B-AWQ`
   → DeepSeek-R1 蒸馏到 Qwen-32B + AWQ 量化(GPU 推理)

3. `Qwen2.5-VL-7B-Instruct-GGUF`
   → 通义千问 2.5 视觉语言版 + 7B + 指令微调 + GGUF 格式(Mac/CPU 能跑)

4. `bge-reranker-v2-m3`
   → BGE 系列重排序模型 + 第 2 代 + M3 变体

---

## 第 5 章 · 量化是什么

### 5.1 为什么需要量化

模型权重原本用 16 位浮点数(FP16)存。一个 8B 模型 ≈ 16 GB,70B 模型 ≈ 140 GB。
普通人没那么大显存。**量化**就是把每个数字用更少的位数表示,文件变小、显存变少、速度变快,代价是损失一点精度。

类比:把一张 4K 照片压缩成 720p,肉眼看差不多,但文件小很多。

### 5.2 精度档位对照表

以 7B 模型为例(原始 FP16 ≈ 14 GB):

| 精度 | 每个参数占用 | 7B 体积 | 质量损失 | 适用场景 |
|------|------------|--------|---------|---------|
| FP32 | 32 位(4 字节) | 28 GB | 基准 | 训练 |
| FP16 / BF16 | 16 位(2 字节) | 14 GB | 几乎无 | 高端推理 |
| FP8 | 8 位浮点 | 7 GB | 极小 | H100/新卡 |
| INT8 | 8 位整数 | 7 GB | 小 | 通用 |
| INT4 / Q4 | 4 位 | 3.5 GB | 中等 | 消费级显卡、Mac |
| Q3 / Q2 | 3 位 / 2 位 | 更小 | 较大 | 极限场景 |

**经验规律**:
- 4 位以上,日常用基本感觉不到差别
- 3 位以下,会明显变笨
- 同等位数下,新算法(AWQ、GGUF 的 K 系列)比老算法精度更好

### 5.3 主流量化格式详解

**GGUF(最重要,新手必学)**
- 推理框架:llama.cpp、Ollama、LM Studio
- 平台:CPU、GPU 混合、Mac M 系列、Windows、Linux 都能跑
- 文件:**单文件**,方便分发(一个 .gguf 文件就够了)
- 命名:`Q4_K_M`、`Q5_K_S`、`Q8_0` 等
  - 数字:位数(4=4 位,8=8 位)
  - K_S / K_M / K_L:Small / Medium / Large,精度档位
  - 推荐档位:**Q4_K_M**(平衡)、**Q5_K_M**(质量更好)、**Q8_0**(几乎无损)
- 适合:个人电脑、Mac、没有强 GPU 的环境

**AWQ(GPU 推荐)**
- 全称:Activation-aware Weight Quantization
- 推理框架:vLLM、SGLang、AutoAWQ、TGI
- 特点:激活感知,4 位下精度比 GPTQ 好
- 适合:NVIDIA GPU,生产部署

**GPTQ(老牌 GPU 量化)**
- 推理框架:AutoGPTQ、ExLlama、vLLM
- 早期主流,现在被 AWQ 慢慢取代,但兼容性广
- 适合:NVIDIA GPU

**FP8**
- 不是后训练量化,而是原生 8 位浮点训练/推理
- 需要 H100、H200、RTX 5090 等支持 FP8 的硬件
- 精度几乎无损,数据中心新趋势
- `Qwen3-Coder-Next-FP8` 就是这种

**MLX**
- Apple 自家深度学习框架的格式
- 专为 M1/M2/M3/M4 芯片优化
- 在 Mac 上比 GGUF 还快一点
- 工具:mlx-lm、LM Studio(支持 MLX 后端)

**EXL2 / EXL3**
- ExLlama / ExLlamaV2 格式
- GPU 速度极快,可指定每层不同位宽(混合精度)
- 适合:有 NVIDIA 显卡的玩家,追求速度

**MNN**
- 阿里巴巴的端侧推理框架
- 主打手机、嵌入式设备
- ModelScope 上有专门的 MNN 模型仓库

**ONNX**
- 跨框架通用格式(ONNX = Open Neural Network Exchange)
- 适合部署到 TensorRT、OpenVINO、Web、移动端
- 大部分模型都能转成 ONNX

**TensorRT-LLM / vLLM 专用包**
- NVIDIA 数据中心高性能推理
- 通常需要在自己机器上"编译",不直接下载

### 5.4 我该选哪种格式?

按硬件选:

| 你的硬件 | 推荐格式 | 推荐工具 |
|---------|---------|---------|
| Mac M1/M2/M3/M4 | GGUF 或 MLX | Ollama、LM Studio |
| Windows + N 卡(8-24G) | GGUF 或 AWQ | LM Studio、Ollama、vLLM |
| Linux + 多张 N 卡 | AWQ / FP8 / GPTQ | vLLM、SGLang |
| Windows / Linux 没显卡 | GGUF(纯 CPU) | Ollama、LM Studio |
| Android / iOS / IoT | MNN / ONNX | MNN、TensorFlow Lite |

按目的选:

- **就想体验** → GGUF + Ollama,最省心
- **想做 RAG / 应用开发** → GGUF + Ollama,或者直接用云 API
- **公司部署给多人用** → AWQ + vLLM(并发高)
- **超大模型(70B+)有限显存** → 选低位 GGUF(Q3、Q4)+ CPU 卸载

### 5.5 一个常见误区

**不是位数越低越好。** 看到 `Q2_K` 和 `Q4_K_M`,别只看体积。Q2 在 7B 这种小模型上几乎不可用,在 70B 这种大模型上才勉强。位数越低,对模型规模要求越高。

经验法则:
- 7B / 8B 模型:不要低于 Q4
- 13B / 14B:Q4 够用
- 32B / 34B:Q4 / Q3 都可
- 70B+:Q3 甚至 Q2 都能用

---

## 第 6 章 · 推理工具生态

模型权重只是文件,得有"推理引擎"加载它才能用。下面按场景介绍主流工具。

### 6.1 个人本地玩(零基础首选)

**Ollama**(命令行,极简)
- 官网:ollama.com
- 跨平台:Windows / Mac / Linux
- 一行命令拉模型:`ollama run llama3.1`
- 自动管理量化版本,默认 GGUF Q4
- 提供 OpenAI 兼容 API(http://localhost:11434)
- 适合:开发者、想用脚本接入应用的人

**LM Studio**(图形界面,小白友好)
- 官网:lmstudio.ai
- 跨平台,带 GUI,内置模型搜索和下载
- 支持 GGUF、MLX、多种量化
- 也能开本地 API 服务
- 适合:零代码用户、Mac 用户

**Jan**(开源版 LM Studio)
- 官网:jan.ai
- 全开源,本地优先,隐私好

**GPT4All**
- 老牌本地 LLM 应用,简单易用

### 6.2 进阶 / 服务器部署

**llama.cpp**(GGUF 的"亲妈")
- GitHub:ggerganov/llama.cpp
- C++ 实现,极致优化,Ollama 和 LM Studio 都基于它
- 支持 CPU、CUDA、Metal、Vulkan、OpenCL 等
- 直接命令行用比较硬核,但性能最好

**vLLM**(GPU 高并发推理首选)
- GitHub:vllm-project/vllm
- 专为生产设计,PagedAttention 技术让吞吐极高
- 支持 AWQ、GPTQ、FP8、原始 BF16
- 提供 OpenAI 兼容 API
- 适合:多人同时用、做 SaaS

**SGLang**
- 类似 vLLM,部分场景更快
- 国产 DeepSeek 等模型常推荐用 SGLang

**TGI(Text Generation Inference)**
- Hugging Face 出品的生产级推理服务

**TensorRT-LLM**
- NVIDIA 官方,极致性能,但部署门槛高

### 6.3 专项工具

**ComfyUI / Automatic1111 / Forge**
- 图像生成(Stable Diffusion / FLUX)的主流 UI
- ComfyUI 是节点式,上限高;A1111 / Forge 更易用

**Whisper.cpp / Faster-Whisper**
- 语音识别本地推理

**Open WebUI**
- 类似 ChatGPT 的 Web 界面,接 Ollama / vLLM 等后端

### 6.4 选型决策树

```
是个人玩?
├── 是 → Mac?
│         ├── 是 → LM Studio (GUI) 或 Ollama (命令行)
│         └── 否 → 有 N 卡?
│                   ├── 是 → LM Studio / Ollama
│                   └── 否 → Ollama 纯 CPU 跑小模型
└── 否(服务器/多人)→ 有 GPU?
                      ├── 是 → vLLM / SGLang
                      └── 否 → llama.cpp server 模式

要做 RAG / Agent 应用?
→ 后端用 Ollama/vLLM,前端用 Open WebUI 或自己写

要训练 / 微调?
→ Hugging Face Transformers + PEFT(LoRA)
→ 或 Unsloth(训练加速,显存友好)
→ 或 LLaMA-Factory(图形化训练工具)
```

---

## 第 7 章 · 实战一:在自己电脑上跑第一个模型

最快路径:**Ollama**。下面是从零到聊天的完整步骤。

### 7.1 安装 Ollama

- Windows / Mac:去 ollama.com 下载安装包,双击装
- Linux:`curl -fsSL https://ollama.com/install.sh | sh`

装完打开终端,输入 `ollama` 看到帮助信息就 OK。

### 7.2 下载并运行模型

挑一个适合你硬件的模型。**第一次推荐 Qwen3 4B 或 Llama 3.2 3B**,小、快、中文也不错。

```bash
# 拉 Qwen3 4B 并直接进入对话
ollama run qwen3:4b

# 或者 Llama 3.2 3B
ollama run llama3.2:3b

# 想要中文更强,Qwen2.5 7B
ollama run qwen2.5:7b
```

第一次会下载几 GB,之后就秒进。退出对话:输入 `/bye` 或按 Ctrl+D。

### 7.3 常用命令

```bash
ollama list              # 看本地装了哪些模型
ollama pull qwen2.5:7b   # 只下载不进入对话
ollama rm qwen3:4b       # 删除某个模型
ollama ps                # 看正在运行的模型
ollama show qwen2.5:7b   # 看模型详情
```

### 7.4 用图形界面更舒服:Open WebUI

Ollama 默认在 `http://localhost:11434` 提供 API。配 Open WebUI 就有 ChatGPT 同款界面。

最简单是用 Docker:
```bash
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data --name open-webui \
  --restart always ghcr.io/open-webui/open-webui:main
```

打开 `http://localhost:3000`,注册一个本地账号,就能聊了。

不想用 Docker,用 LM Studio 也行,自带聊天界面。

### 7.5 换个模型试试

Ollama 模型库:ollama.com/library

按硬件选:
- 4 GB 显存 / 8 GB 内存:`qwen3:1.7b`、`llama3.2:1b`、`gemma3:1b`
- 8 GB 显存 / 16 GB 内存:`qwen3:4b`、`llama3.2:3b`、`phi3.5`
- 16 GB 显存:`qwen2.5:14b`、`llama3.1:8b`、`mistral:7b`
- 24 GB+ 显存:`qwen2.5:32b`、`llama3.3:70b`(Q4)
- 看图能力:`llava`、`qwen2.5vl`、`llama3.2-vision`

### 7.6 常见问题排查

**模型答得很慢**
→ 显存不够,模型被卸载到 CPU 跑了。换更小或更低量化的版本。

**输出乱码 / 重复**
→ 模型选得不对,或者量化太低。换 Q5 或 Q8 试试。

**显示中文不好**
→ 优先选 Qwen、DeepSeek、GLM、Yi 这些国产系列。

**Mac 跑得慢**
→ 关掉浏览器和后台占内存的应用。M 系列要保证统一内存够用。

---

## 第 8 章 · 实战二:用 API 接入应用

让模型从"聊天玩具"变成"应用零件",关键是 API。

### 8.1 OpenAI 兼容 API:事实标准

几乎所有现代推理引擎(OpenAI、Ollama、vLLM、LM Studio、DeepSeek、阿里百炼、智谱)都遵循同一套接口规范。
学会一套,处处通用。

核心接口:`POST /v1/chat/completions`

### 8.2 Python 调用本地 Ollama

先装库:
```bash
pip install openai
```

写代码:
```python
from openai import OpenAI

# 指向本地 Ollama
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 本地随便填
)

resp = client.chat.completions.create(
    model="qwen2.5:7b",
    messages=[
        {"role": "system", "content": "你是一个简洁的助手。"},
        {"role": "user", "content": "用一句话解释什么是量化"}
    ]
)
print(resp.choices[0].message.content)
```

把 `base_url` 改成 OpenAI / DeepSeek / 通义的地址,代码不用改。

### 8.3 流式输出(打字机效果)

```python
stream = client.chat.completions.create(
    model="qwen2.5:7b",
    messages=[{"role": "user", "content": "写一首关于春天的短诗"}],
    stream=True
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

### 8.4 多轮对话

API 是无状态的,得自己维护历史:
```python
history = [{"role": "system", "content": "你是助手。"}]
while True:
    user = input("你: ")
    if not user: break
    history.append({"role": "user", "content": user})
    resp = client.chat.completions.create(
        model="qwen2.5:7b", messages=history
    )
    answer = resp.choices[0].message.content
    print("AI:", answer)
    history.append({"role": "assistant", "content": answer})
```

### 8.5 工具调用 / Function Calling

让模型决定"该调用哪个函数",是 Agent 的基础。
```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询某城市天气",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"]
        }
    }
}]

resp = client.chat.completions.create(
    model="qwen2.5:7b",
    messages=[{"role": "user", "content": "上海天气怎么样"}],
    tools=tools
)
print(resp.choices[0].message.tool_calls)
```

模型会返回它想调用 `get_weather(city="上海")`,你的代码真正去查天气,再把结果喂回给它,它再生成最终回答。

### 8.6 选 API 服务的建议

**国外**
- OpenAI:综合最强,贵
- Anthropic Claude:写作和长文档强
- Google Gemini:多模态、免费额度大

**国内**
- DeepSeek:便宜,V3/R1 性价比超高
- 阿里百炼(Qwen):中文好,Qwen3 系列免费额度
- 智谱 GLM、字节豆包、月之暗面 Kimi 各有特色

**自部署 vs 用云 API**
- 个人玩 / 隐私敏感 → 自部署
- 商用 / 高并发 / 不想运维 → 云 API
- 混合策略:简单任务本地小模型,复杂任务调云端大模型

---

## 第 9 章 · 进阶玩法

### 9.1 RAG(检索增强生成)

让模型回答你**自己的资料**(公司文档、笔记、PDF)。

原理:
```
用户提问
   ↓
[Embedding 模型] 把问题向量化
   ↓
[向量数据库] 找出最相关的几段文档
   ↓
把这几段 + 问题一起喂给 LLM
   ↓
LLM 基于资料回答
```

零代码工具:
- **AnythingLLM**(开源、桌面版,接 Ollama)
- **Dify**(开源、Web 平台)
- **MaxKB / FastGPT / RagFlow**

要写代码:**LangChain** 或 **LlamaIndex** 框架,加一个向量库(Chroma / Milvus / Qdrant)。

关键组件:
- Embedding 模型:bge-m3、text-embedding-3
- 向量库:Chroma(本地易用)、Qdrant、Milvus
- Reranker:bge-reranker-v2-m3(显著提升精度)
- LLM:Qwen2.5、Llama 3.1 等

### 9.2 微调(让模型懂你的领域)

什么时候要微调:
- 提示词调不好的特定输出格式
- 某个垂直领域(法律、医疗)需要稳定专业表达
- 想做风格模仿(客服话术、个人写作风格)

主流方法:
- **LoRA / QLoRA**:消费级显卡能做,小数据集就够
- **全参数微调**:效果最好,但要多卡

零基础推荐工具:
- **LLaMA-Factory**:图形化界面,支持各大模型
- **Unsloth**:训练加速 2-5 倍,显存省一半
- **Axolotl**:配置文件驱动,工业界爱用

最小数据量经验:几百条高质量样本就有效果,几千条很稳。

### 9.3 Agent(让模型自己干活)

Agent = LLM + 工具 + 循环。
模型自己思考下一步、调用工具、看结果、再思考……直到完成任务。

主流框架:
- **LangChain / LangGraph**:生态最大
- **AutoGen**(微软):多 Agent 协作
- **CrewAI**:角色扮演式协作
- **MetaGPT**:模拟软件公司
- **Dify / Coze**:零代码搭 Agent

模型层面看,带 `tool use` 能力强的:GPT-4、Claude Sonnet、Qwen2.5、DeepSeek-V3。

### 9.4 MCP(Model Context Protocol)

Anthropic 推动的"工具接入标准协议"。一个标准,所有支持 MCP 的客户端都能用。
现在 Claude、Cursor、Cline、各种 IDE 都在接 MCP。
理解为"AI 界的 USB":一次开发,处处插上即用。

### 9.5 多模态生成实战

**图像生成**:装 ComfyUI 或 Forge,下个 FLUX.1-dev 或 SDXL 模型,文生图。
**视频生成**:Runway、Sora(API)、可灵、Wan2.1(开源)。
**语音克隆**:CosyVoice、F5-TTS、GPT-SoVITS,几秒样本就能仿声。

---

## 第 10 章 · 学习路径与资源

### 10.1 推荐学习顺序

第 1 周:**会用**
- 装 Ollama,跑一个 7B 模型
- 用 LM Studio / Open WebUI 体验 GUI
- 学会写好提示词(Prompt Engineering)

第 2-3 周:**会接**
- Python + OpenAI 兼容 API
- 流式、多轮、工具调用
- 部署 Open WebUI 给家人朋友用

第 4-6 周:**会做应用**
- RAG:用 AnythingLLM 或 Dify 搭一个知识库
- Agent:用 Dify / Coze 做一个工作流
- 学一点向量数据库

第 2-3 个月:**会调优**
- LoRA 微调一个垂直领域模型
- 学 vLLM 部署生产环境
- 理解推理参数(temperature、top_p、repetition_penalty)

第 4 个月以后:**深入**
- Transformer 原理(看 3Blue1Brown、Andrej Karpathy 视频)
- 论文阅读:从 GPT、LLaMA、DeepSeek 技术报告开始
- 自己跑预训练 / 强化学习实验

### 10.2 推荐资源

**模型站**
- Hugging Face(全球最大)
- ModelScope 魔搭(阿里,国内快)
- Ollama Library(开箱即用)

**社区 / 资讯**
- r/LocalLLaMA(reddit,本地 LLM 圣地)
- Hugging Face Daily Papers
- 公众号:机器之心、量子位、HuggingFace 中文站

**视频教程**
- 3Blue1Brown 的神经网络系列
- Andrej Karpathy 的"从零造 GPT"
- 李沐《动手学深度学习》

**实操项目**
- llama.cpp、Ollama 的官方文档
- LangChain / LlamaIndex 教程
- Hugging Face Course(免费)

### 10.3 心态建议

- 这个领域**每月都有新东西**,不可能学完。学方法、学评估、学迁移。
- 别一上来啃论文。先跑起来,有了直觉再深入。
- 多实验,少争论。同一个问题不同模型表现差很多,自己测最准。
- 不要迷信 benchmark 排行榜。你的真实场景才是 benchmark。

---

## 附录 · 术语速查表

| 术语 | 全称 / 解释 |
|------|------------|
| LLM | Large Language Model,大语言模型 |
| VLM | Vision Language Model,视觉语言模型 |
| Token | 模型处理的最小单位,1 个汉字约 1-2 token |
| Context / Context Window | 上下文长度,模型一次能"看到"多少 token |
| Parameter | 参数,模型内部的"旋钮" |
| Pretraining | 预训练 |
| SFT | Supervised Fine-Tuning,监督微调 |
| RLHF | Reinforcement Learning from Human Feedback |
| DPO | Direct Preference Optimization,直接偏好优化 |
| LoRA | Low-Rank Adaptation,低秩适配,轻量微调 |
| QLoRA | 量化 + LoRA |
| MoE | Mixture of Experts,混合专家 |
| Distill | 蒸馏,小模型学大模型 |
| Quantization | 量化 |
| GGUF | llama.cpp 的模型格式 |
| AWQ | Activation-aware Weight Quantization |
| GPTQ | 一种 GPU 量化算法 |
| FP16 / BF16 / FP8 | 不同精度的浮点 |
| Embedding | 把内容转成向量 |
| Reranker | 重排序模型 |
| RAG | Retrieval-Augmented Generation,检索增强生成 |
| Agent | 能自主调用工具完成任务的 AI |
| Function Calling / Tool Use | 工具调用 |
| MCP | Model Context Protocol |
| Inference | 推理(模型对外提供服务) |
| Prompt | 提示词 |
| System Prompt | 系统提示词,定义角色和规则 |
| Temperature | 温度,控制随机性,越高越发散 |
| Top-p / Top-k | 采样策略 |
| Streaming | 流式输出,边生成边返回 |
| Chain-of-Thought / CoT | 思维链,让模型逐步推理 |
| Reasoning Model | 推理模型(o1、R1 等) |
| Hallucination | 幻觉,模型一本正经胡说 |
| Benchmark | 测试榜 |
| Hugging Face | 全球最大模型社区 |
| ModelScope | 阿里的模型社区(魔搭) |

---

**结尾的话**

不要追求"学完",追求"开始"。装一个 Ollama,跑一个 Qwen3,问它一个你最关心的问题——这一刻起,你就已经入门了。
其余的全是在路上慢慢补的。祝玩得开心。




