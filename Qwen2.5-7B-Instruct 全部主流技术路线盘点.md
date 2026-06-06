

除了 GGUF (Ollama/llama.cpp) 和原版 (transformers/vLLM),实际上还有十多条技术路线,按**用途**分成五大类。

---

## 一、推理路线(把模型跑起来)

### 1. **MLX**(Apple Silicon 专属)

- **定位**:苹果官方为 M 系列芯片设计的机器学习框架
- **格式**:`.mlx`(从 safetensors 转换)
- **依赖**:`mlx`、`mlx-lm`
- **优势**:充分利用 M 芯片统一内存架构,M2 Max/M3 Pro 上比 llama.cpp 还快 20-40%
- **场景**:Mac 用户的最优本地推理方案
- **安装**:
    
    ```bash
    pip install mlx-lmmlx_lm.generate --model Qwen/Qwen2.5-7B-Instruct --prompt "你好"
    ```
    
- **缺点**:仅限 Apple 芯片,Windows/Linux 用不了

### 2. **TensorRT-LLM**(NVIDIA 极致优化)

- **定位**:N 卡上的"性能天花板"
- **格式**:编译成 `.engine` 文件(针对特定 GPU 架构优化)
- **依赖**:CUDA、cuDNN、TensorRT、`tensorrt-llm` Python 包
- **优势**:用上 FP8、INT4 量化 + 算子融合 + kernel autotuning,延迟最低、吞吐最高
- **场景**:大规模生产部署,A100/H100/L40 集群
- **缺点**:编译过程复杂,模型一改就要重编译,锁定 NVIDIA

### 3. **SGLang**(后起之秀,LMSYS 出品)

- **定位**:vLLM 的强力竞争者,主打"结构化生成"
- **依赖**:torch、`sglang[all]`、FlashInfer
- **优势**:
    - RadixAttention 缓存机制,多轮对话/Agent 场景比 vLLM 快 2-5 倍
    - 原生支持 JSON Schema 约束输出
    - 支持视觉模型(LLaVA、Qwen-VL)
- **场景**:Agent 编排、复杂 prompt 流水线、多轮对话服务
- **启动**:
    
    ```bash
    python -m sglang.launch_server --model Qwen/Qwen2.5-7B-Instruct --port 30000
    ```
    

### 4. **TGI(Text Generation Inference)**(HuggingFace 官方)

- **定位**:HF 生态官方推理服务器
- **依赖**:Rust + Python,通常用 Docker 跑
- **优势**:与 HF Hub 无缝集成,生产级稳定,支持 streaming/batching
- **场景**:HuggingFace Inference Endpoints 后端、企业内部服务
- **缺点**:协议授权较严(HFOIL),商用大公司需注意

### 5. **LMDeploy**(上海 AI Lab 出品)

- **定位**:国产推理引擎,InternLM 团队开发
- **依赖**:`lmdeploy`、TurboMind 后端
- **优势**:对国产 GPU 适配好,W4A16 量化效果不错,支持 Qwen 全系
- **场景**:国内企业、需要国产化适配
- **启动**:
    
    ```bash
    pip install lmdeploylmdeploy serve api_server Qwen/Qwen2.5-7B-Instruct
    ```
    

### 6. **OpenVINO**(Intel 出品)

- **定位**:Intel CPU/GPU/NPU 优化推理
- **格式**:IR 中间格式
- **优势**:在 Intel 集显、Arc 独显、酷睿 Ultra NPU 上表现远好于通用方案
- **场景**:Intel 硬件、AI PC、边缘设备
- **依赖**:`openvino`、`optimum-intel`

### 7. **ONNX Runtime**(微软主导,跨平台)

- **定位**:统一中间格式,一次导出,到处运行
- **格式**:`.onnx`
- **优势**:支持 CPU/GPU/手机/浏览器(WebGPU),DirectML 后端能跑 AMD 卡
- **场景**:跨平台部署、嵌入到 .NET/Java/C++ 应用
- **缺点**:LLM 支持不如专用引擎成熟,导出 7B 模型有坑

### 8. **MLC LLM**(陈天奇团队,TVM 系)

- **定位**:编译型推理,目标"任何设备都能跑"
- **优势**:能编译到 iOS/Android/WebGPU/Vulkan,浏览器里直接跑 LLM
- **场景**:移动端 App、网页端 AI、跨平台分发
- **缺点**:编译复杂,生态较小

### 9. **PowerInfer / PowerInfer-2**(上交大)

- **定位**:稀疏激活推理,CPU 也能高速跑大模型
- **优势**:利用神经元激活稀疏性,7B 模型在 PC CPU 上能到 10+ t/s
- **场景**:无 GPU 但要跑大模型的研究/低成本部署

### 10. **DeepSpeed-MII**(微软)

- **定位**:DeepSpeed 推理子项目
- **优势**:与 DeepSpeed 训练栈无缝衔接,适合训练完直接部署
- **场景**:Azure 云、微软系企业

---

## 二、量化路线(把模型压小)

不同量化方案产出不同格式的模型,影响选什么推理引擎。

### 1. **GGUF 量化**(llama.cpp 系)

- 工具:`llama.cpp` 的 `quantize` 程序
- 类型:Q2_K、Q3_K_M、Q4_0、Q4_K_M、Q5_K_M、Q6_K、Q8_0、IQ 系列
- 用途:配 Ollama / LM Studio / llama.cpp

### 2. **GPTQ**(后训练量化经典方案)

- 工具:`auto-gptq`、`gptqmodel`
- 类型:4bit / 8bit,group size 通常 128
- 用途:配 transformers / vLLM / TGI

### 3. **AWQ**(MIT 韩松团队)

- 工具:`autoawq`
- 优势:激活感知量化,精度损失小于 GPTQ
- 用途:vLLM / SGLang / TensorRT-LLM 都原生支持

### 4. **bitsandbytes**(在线动态量化)

- 工具:`bitsandbytes` 库
- 类型:NF4、FP4、INT8
- 用途:QLoRA 微调主力,加载时即时量化,无需预处理

### 5. **SmoothQuant / SqueezeLLM / OmniQuant**

- 学术界量化方案,各有特色,生产用得少

### 6. **FP8**(H100/L40 新硬件原生支持)

- 工具:TensorRT-LLM、vLLM 0.5+
- 优势:几乎无损,速度接近 INT8
- 限制:只有 Hopper 架构以上的卡支持

---

## 三、微调路线(让模型变聪明/专业)

### 1. **全参数微调(Full Fine-tuning)**

- 框架:transformers + DeepSpeed / FSDP
- 资源:7B 全参微调需要 80GB+ 显存
- 场景:有充足 GPU,要彻底改变模型行为

### 2. **LoRA / QLoRA**(最常用)

- 框架:`peft` + `bitsandbytes` + `trl`
- 资源:7B QLoRA 在单张 4090(24GB)就能跑
- 场景:领域适配、风格定制、95% 的微调需求

### 3. **LLaMA-Factory**(国产王牌微调框架)

- 网址:github.com/hiyouga/LLaMA-Factory
- 优势:
    - Web UI 零代码微调
    - 支持 Qwen 全系列开箱即用
    - 集成 LoRA/QLoRA/全参/DPO/PPO/KTO/ORPO
- 场景:工程化微调首选,中国用户极多

### 4. **Axolotl**(国外热门微调框架)

- 优势:配置文件驱动,支持各种新方法
- 场景:海外开源社区主流

### 5. **Unsloth**(微调速度王)

- 优势:重写 Triton kernel,LoRA 训练速度提升 2 倍,显存降 40%
- 场景:单卡微调追求极致效率

### 6. **DPO / KTO / ORPO / SimPO**(偏好对齐)

- 框架:`trl`、LLaMA-Factory
- 场景:让模型符合人类偏好,替代复杂的 RLHF

### 7. **强化学习(PPO / GRPO)**

- 框架:`trl`、`OpenRLHF`、`verl`
- 场景:RLHF 对齐、推理能力增强(类似 R1)
- 难度:高,需要奖励模型 + 大算力

---

## 四、应用层路线(把模型变成产品)

### 1. **RAG(检索增强)**

- 框架:**LangChain**、**LlamaIndex**、**Haystack**、**Dify**
- 向量库:Chroma、Milvus、Qdrant、Weaviate、PGVector
- 嵌入模型:bge-m3、gte-Qwen2、jina-embeddings
- 场景:私有知识库问答、文档智能助手

### 2. **Agent 编排**

- 框架:**LangGraph**、**AutoGen**(微软)、**CrewAI**、**MetaGPT**
- 场景:多步任务、工具调用、自动化流程

### 3. **低代码平台**

- **Dify**:开源 LLMOps,可视化 RAG + Agent + Workflow
- **FastGPT**:国产,知识库问答场景强
- **n8n + LLM**:工作流自动化
- **Coze**(扣子):字节出品,生态丰富
- **场景**:产品经理/非技术人员搭 AI 应用

### 4. **桌面客户端**

- **LM Studio**:GUI 跑 GGUF,小白友好
- **Jan**:开源 LM Studio 替代品
- **Cherry Studio**:国产,接 API 体验好
- **Chatbox**:轻量,接 OpenAI/Ollama 都行
- **GPT4All**:跨平台桌面应用

### 5. **IDE 集成**

- **Continue.dev**:VS Code/JetBrains 接 Ollama 做代码助手
- **Aider**:命令行 AI 编程
- **Cursor / Windsurf**:AI 优先 IDE(主要接云模型,也能配本地)

### 6. **浏览器/Web 端**

- **WebLLM**(MLC):浏览器 WebGPU 直接跑 7B 模型
- **Transformers.js**:HF 出的 JS 版 transformers

---

## 五、训练前路线(预训练/继续预训练)

一般个人不碰,企业才用:

### 1. **Megatron-LM**(NVIDIA)

- 张量并行 + 流水并行,千卡训练标配

### 2. **DeepSpeed**(微软)

- ZeRO 优化器,降低显存需求

### 3. **Colossal-AI**(国产)

- 异构内存管理,消费级 GPU 也能训

### 4. **NeMo**(NVIDIA)

- 端到端训练框架,从预训练到对齐

### 5. **PyTorch FSDP**(原生方案)

- PyTorch 自带的全分片数据并行

---

## 六、按场景选路线总览图

```
个人本地用
├── Mac 用户         → MLX / Ollama
├── Windows 普通用户 → Ollama + Cherry Studio
├── Windows 有 N 卡  → Ollama / LM Studio
├── 浏览器即开即用   → WebLLM
└── 极致性能(N 卡)  → vLLM / SGLang

服务端部署
├── 高并发生产环境   → vLLM / SGLang / TGI / TensorRT-LLM
├── 国产 GPU        → LMDeploy
├── 企业级 RAG      → Dify / FastGPT
└── Agent 平台      → Coze / LangGraph

研究 / 微调
├── 零代码微调      → LLaMA-Factory(Web UI)
├── 单卡微调最快    → Unsloth
├── 海外社区方案    → Axolotl
└── 多卡全参微调    → DeepSpeed / FSDP

边缘 / 嵌入
├── 手机 App        → MLC LLM
├── 树莓派          → llama.cpp
├── Intel 设备      → OpenVINO
└── 跨平台 SDK      → ONNX Runtime
```

---

## 七、技术路线选择决策树

```
你的目标是什么?
│
├─ 自己用 / 离线聊天
│   ├─ 没编程基础   → LM Studio / Cherry Studio + Ollama
│   ├─ 会点命令行   → Ollama + GGUF
│   └─ Mac 用户     → MLX + LM Studio
│
├─ 给团队/客户提供服务
│   ├─ 并发 < 10    → Ollama 直接顶
│   ├─ 并发 10-100  → vLLM / SGLang
│   └─ 并发 > 100   → TensorRT-LLM + 多卡
│
├─ 做应用产品
│   ├─ 知识库问答   → Dify / FastGPT(快)或 LangChain(灵活)
│   ├─ Agent       → LangGraph / AutoGen / CrewAI
│   └─ 桌面工具     → Tauri/Electron + Ollama API
│
├─ 微调训练
│   ├─ 第一次玩      → LLaMA-Factory Web UI
│   ├─ 追求速度     → Unsloth
│   └─ 多卡分布式    → DeepSpeed / FSDP + transformers
│
└─ 学术研究
    ├─ 论文复现      → transformers + PyTorch
    ├─ 新方法实验    → 自己改 transformers 源码
    └─ 大规模实验    → Megatron / NeMo
```

---

## 八、生态全景图(简化版)

```
                    Qwen2.5-7B-Instruct
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   预训练/继续训练        微调/对齐          推理/部署
        │                   │                   │
   Megatron-LM          LLaMA-Factory      ┌────┴────┐
   DeepSpeed            Unsloth            │         │
   Colossal-AI          Axolotl       全精度路线  量化路线
   NeMo                 trl/peft           │         │
                                      transformers  GGUF→Ollama
                                      vLLM         GPTQ→vLLM
                                      SGLang       AWQ→vLLM
                                      TGI          MLX(Mac)
                                      LMDeploy     OpenVINO(Intel)
                                      TensorRT-LLM
                                            │
                                     ┌──────┴──────┐
                                     │             │
                                  应用层         客户端
                                  Dify          LM Studio
                                  FastGPT       Cherry Studio
                                  LangChain     Chatbox
                                  LlamaIndex    Continue.dev
                                  AutoGen       WebLLM
```

---

## 九、给你的实操建议

如果你只想**用**模型,优先级:

1. **首选**:Ollama + GGUF(已经走的路,够用)
2. **想要更好 UI**:加装 Open-WebUI 或 Cherry Studio
3. **想要更高性能**(有 N 卡):换 vLLM 或 SGLang
4. **想要做产品**:Ollama 当后端 + Dify 当前端

如果你想**深入学**,建议路径:

1. 先把 Ollama 玩熟(已完成 ✅)
2. 学 transformers,会写 `model.generate()`
3. 学 LangChain/LlamaIndex,搭一个 RAG
4. 学 LLaMA-Factory,微调一个领域模型
5. 学 vLLM,做高性能服务化
6. 学 Agent 框架,做复杂应用

需要我详细展开哪一条?比如:

- vLLM 完整部署教程
- LLaMA-Factory 零代码微调实操
- Dify 搭建企业知识库
- MLX 在 Mac 上跑 Qwen2.5
- 浏览器端 WebLLM 部署