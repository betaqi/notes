
> 本阶段目标:能独立完成"找模型 → 下载模型 → 跑起来 → 部署成服务 → 评估性能"的完整链路,并理解 2026 年主流推理工具栈的取舍。

---

### 0-1 OpenAI 兼容 API 范式【必学】

**学习重点:**

- `/v1/chat/completions` / `/v1/completions` / `/v1/embeddings` 请求与响应结构
- `messages` 数组结构(role: system / user / assistant / tool)
- `stream=true` 流式输出与 SSE 协议
- `tools` / `tool_calls` / `tool_choice` 字段
- `response_format`(JSON mode / JSON Schema 严格输出)
- `usage` 字段:`prompt_tokens` / `completion_tokens` / `cached_tokens` / `reasoning_tokens`

**理解:**

- 闭源 API(OpenAI / Anthropic / Gemini)与开源推理引擎(vLLM / SGLang / Ollama)对外协议的统一性
- 为什么"换模型不换代码"成为可能
- OpenAI SDK / LangChain / LlamaIndex / Cherry Studio 都基于这套协议

---

### 0-2 模型仓库与发布生态【必学】

**学习重点:**

- Hugging Face Hub:model / dataset / space
- ModelScope(国内首选镜像)
- 其他渠道:Kaggle Models、Ollama Library、Replicate、厂商官方发布
- 标准文件:`config.json` / `tokenizer.json` / `tokenizer_config.json` / `generation_config.json` / `chat_template.jinja`
- 权重格式:`safetensors`(主流)/ `pytorch_model.bin`(旧)/ `GGUF`(llama.cpp/Ollama)/ `MLX`(Apple Silicon)/ `AWQ` / `GPTQ`(量化)
- checkpoint shard 分片:`model-00001-of-0000N.safetensors` + `model.safetensors.index.json`
- model revision / branch / tag,LFS 大文件机制
- 许可证类型:Apache 2.0 / MIT / Llama Community / Qwen License / Gemma 等的商用边界
- GGUF 量化等级与质量/显存对应(Q4_K_M / Q5_K_M / Q6_K / Q8_0)

**理解:**

- 看到一个仓库目录,能立刻判断:模型类型、参数规模、是否量化、能否商用

---

### 0-3 模型下载与本地管理【必学】

**学习重点:**

- 三种下载方式:`huggingface-cli download` / `modelscope download` / `git lfs clone`
- 缓存路径约定:`HF_HOME` / `HF_HUB_CACHE` / `MODELSCOPE_CACHE`
- 国内镜像(`hf-mirror.com`)与 `HF_ENDPOINT` 环境变量
- 代理配置 / 断点续传 / hash 校验
- `snapshot_download` 与 `local_dir` 差异(软链接 vs 实文件)
- 模型完整性自检:必须存在哪些文件才能推理

**理解:**

- 多模型共享缓存的目录规划
- 离线环境的模型分发策略

---

### 0-4 Transformers 推理基础【必学】

**学习重点:**

- `AutoTokenizer` / `AutoModelForCausalLM` / `AutoModel`
- `pipeline` 快捷接口
- `tokenizer.apply_chat_template(messages, add_generation_prompt=True)`
- `model.generate()` 参数:`max_new_tokens` / `temperature` / `top_p` / `top_k` / `repetition_penalty` / `do_sample`
- `TextStreamer` / `TextIteratorStreamer` 流式输出
- batch inference 与 padding side
- `device_map="auto"` / `torch_dtype=torch.bfloat16` / `trust_remote_code`
- `attn_implementation="flash_attention_2"` / `"sdpa"`

**理解:**

- Tokenizer:BPE / SentencePiece / Tiktoken 差异
- context length 与位置编码(RoPE / YaRN 扩展)
- KV Cache 原理与显存占用
- Prefill 阶段(并行)vs Decode 阶段(串行)
- EOS / BOS / PAD / 特殊 token
- **Chat Template 是模型的一部分**:Qwen / Llama / DeepSeek / GLM 模板差异巨大,99% 的"模型胡言乱语"是模板用错了

---

### 0-5 推理引擎全景对比【必学】⭐

**学习重点:**

|引擎|定位|适用场景|
|---|---|---|
|Ollama / LM Studio|本地一键运行|个人开发、原型|
|llama.cpp|CPU / 边缘 / 极致量化|端侧、低资源|
|Transformers|调试、训练后验证|学习、微调验证|
|**vLLM**|生产级高并发|服务化主力|
|**SGLang**|高性能 + 结构化输出强|Agent / 复杂控制流|
|TensorRT-LLM|NVIDIA 极致性能|大厂生产|
|MLX / MLX-LM|Apple Silicon|Mac 本地开发|

核心概念:

- **Continuous Batching** 连续批处理
- **PagedAttention**(vLLM 核心创新)
- **Prefix Caching / Automatic Prefix Caching**(Agent 与 RAG 提速关键)
- **Speculative Decoding**:Draft model / EAGLE / Medusa
- **Structured Output / Guided Decoding**:`outlines` / `xgrammar` / `lm-format-enforcer`
- **Tensor Parallel** / **Pipeline Parallel** / **Expert Parallel**(MoE)
- **Disaggregated Serving**(Prefill 与 Decode 分离)

**理解:**

- 同一份代码切不同引擎的关键参数
- 不同场景的选型决策树(本地 / 生产 / 边缘 / Mac)

**实战:**

- 用 vLLM 部署一个 OpenAI 兼容服务(`vllm serve`)
- 用 Ollama 部署一个 GGUF 量化模型并写 Modelfile
- 用 SGLang 部署一个支持 JSON Schema 强制输出的服务

---

### 0-6 国内开源模型生态【必学】

**学习重点:**

|模型家族|厂商|特点|
|---|---|---|
|Qwen3 系列(含 Qwen3-Coder、Qwen3-VL)|阿里|综合首选,工具调用强|
|DeepSeek-V3 / R1|DeepSeek|MoE + 推理模型代表|
|GLM-4.6 / GLM-Z1|智谱|中文对齐好|
|Kimi K2|Moonshot|长上下文 + Agent|
|MiniCPM-V / InternVL3|面壁 / 上海 AI Lab|端侧 / 多模态|
|Step / 豆包 / Hunyuan|阶跃 / 字节 / 腾讯|闭源 API 为主|

**理解:**

- Dense 模型 vs MoE 模型:总参数 / 激活参数 / 显存 / 吞吐差异
- Base 模型 vs Instruct 模型 vs Reasoning 模型(R1 / o-系列范式)
- 推理模型的输出结构:`<think>...</think>` 与 `reasoning_content` 字段

---

### 0-7 Embedding 与 Rerank 模型【必学】

**学习重点:**

- 当前主流模型:**BGE-M3**(稠密+稀疏+多向量)/ **Qwen3-Embedding / Qwen3-Reranker** / BCE / gte / jina-embeddings-v3 / **bge-reranker-v2-m3**
- 多模态 embedding:CLIP / SigLIP / Jina-CLIP / ColPali

**理解:**

- Embedding(BiEncoder)vs Rerank(CrossEncoder)分工
- 向量维度、cosine / dot product、归一化
- ANN(HNSW / IVF / ScaNN)近似搜索
- chunk embedding / query embedding / 非对称检索
- top-k 召回 → rerank 精排的两阶段范式
- **Late Interaction**(ColBERT / ColPali)新范式
- embedding 维度与向量库存储成本权衡
- 长文本 embedding:chunk 策略 vs long-context embedding

---

### 0-8 推理服务化与可观测性【必学】

**学习重点:**

- 流式输出协议(SSE)客户端处理(Python / JS)
- 关键指标:**TTFT**(首 token 延迟)/ **TPOT**(每 token 延迟)/ 吞吐(req/s、token/s)/ 并发上限
- 显存估算:`显存 ≈ 参数量 × 精度字节 + KV Cache(并发 × 上下文 × 层数 × 头维度)`
- 压测工具:`vllm bench` / `genai-perf` / `evalscope-perf`
- Prometheus + Grafana 监控 vLLM 指标
- 请求日志、token 计费日志最小可用方案

**理解:**

- TTFT 决定用户体感,TPOT 决定阅读节奏,吞吐决定成本
- 输入 token 与输出 token 单价差异巨大对架构的影响