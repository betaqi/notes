
> 主线：**文档上传解析 → 向量化存储 → 问答检索** 参考来源：阶段一 (0-x)、阶段二 (1-x)、阶段四 (3-x) 原则：只学跑通主线必需的部分，扩展点（Rerank / Agentic / GraphRAG / 评估）先打标记，后期再补。
---

### 模块 0 · 前置基础（1–2 天）

|主题|重点|参考|
|---|---|---|
|Python 异步|`async/await`、`asyncio.gather`、`Semaphore`|1-1|
|Pydantic v2|`BaseModel` / `Field` / `model_validate` / `model_dump`|1-3|
|环境与依赖|`uv` 或 `pip`、`.env`、`pydantic-settings`、`loguru`|1-7|
|Docker 基础|起一个 Qdrant 容器、端口映射、持久化 volume|3-4|

**产出**：能用 `AsyncOpenAI` 跑一个 hello-world 流式对话。

---

### 模块 1 · OpenAI API 必备（1 天）

参考 **阶段一 0-1** + **阶段二 1-1 / 1-5 / 1-6**

- `/v1/chat/completions` 请求/响应结构、`messages` 数组（role: system/user/assistant）
- `stream=True` 流式 + SSE 协议
- `/v1/embeddings`：`text-embedding-3-small`（1536 维）vs `-3-large`（3072 维）
- `usage` 字段：`prompt_tokens / completion_tokens / cached_tokens`
- `tiktoken` 算 token、成本估算
- **Prompt Caching 命中率**：静态内容前置、动态内容后置（1-5 关键点）
- 重试 / 超时 / 限流基础（1-9，先了解概念）

**产出**：能调用 chat + embeddings，理解输入输出 token 成本差异。

---

### 模块 2 · FastAPI 后端骨架（1–2 天）

参考 **阶段二 1-7 / 1-8**

- `APIRouter` 模块化、`Depends` 依赖注入
- `lifespan` 初始化共享 `AsyncOpenAI` 客户端
- Pydantic 请求/响应模型、`UploadFile` 文件上传
- `BackgroundTasks` 后台任务（用于异步入库）
- `StreamingResponse(media_type="text/event-stream")` 实现 SSE
- 自定义 SSE 事件协议：`message` / `citation` / `usage` / `done` / `error`
- 中间件：trace_id、请求日志、耗时统计
- 全局异常处理器

**产出**：一个能跑的 FastAPI 服务，含 `/health`、上传、流式 chat 三类接口骨架。

---

### 模块 3 · 文档解析与数据准备 ⭐（2–3 天）

参考 **阶段四 3-2**

**主线必学（够用版）**：

- PDF：`PyMuPDF (fitz)` —— 提取文本 + 页码
- DOCX：`python-docx`
- Markdown：直接读
- HTML：`BeautifulSoup`
- 元数据抽取：文件名、页码、章节标题、上传时间、`doc_id`

**理解（最重要）**：

- "Garbage In, Garbage Out"——企业 RAG 80% 的效果差在解析
- 表格直接切片 = 召回灾难，必须单独处理（先转 Markdown 表格）
- 元数据用于过滤 + 召回时增强（如标题前缀拼到 chunk 文本里）

**进阶（先打标记）**：MinerU / Docling / LlamaParse / Unstructured / OCR（PaddleOCR）

**产出**：能把 PDF/DOCX/MD 转成统一的 `Chunk` 列表（带 page、section 元数据）。

---

### 模块 4 · 切片策略 ⭐（1 天）

参考 **阶段四 3-3**

**主线必学**：

- `RecursiveCharacterTextSplitter`（langchain-text-splitters）
- **结构化切片**：按解析出来的标题/段落切，比按字符切召回率高得多
- 经验值：中文 300–500 字 / overlap 10–15%
- chunk 元数据增强：标题前缀、章节路径、文档摘要都拼进 chunk 文本

**理解**：

- chunk 大小 ↔ 召回率 / 精确率 / 生成质量 三角权衡
- 何时需要 Parent-Child（先了解概念，主线不实现）

**进阶（打标记）**：Semantic Chunking、Late Chunking（Jina）、**Contextual Retrieval（Anthropic 2024）**——后期加配 Prompt Caching 成本极低。

**产出**：把模块 3 的原子块切成 200–500 字的可索引 chunk。

---

### 模块 5 · Embedding 与向量库 ⭐⭐（2 天）

参考 **阶段一 0-7** + **阶段四 3-4**

**必学**：

- OpenAI `text-embedding-3-small` 批量调用（注意：单次最多 2048 个输入，要分批 + 限流）
- 向量基础：cosine / dot product / 归一化、ANN（HNSW）
- 向量维度 ↔ 存储成本（1024 维 × 100 万 ≈ 4GB float32）

**向量库（主线选一个）**：

- **Qdrant（推荐）**：docker 一键、性能强、Python SDK 友好、支持 metadata filter
- **Chroma**：极简本地、原型最快
- 了解：Milvus（生产首选）、pgvector（已有 PG）、Weaviate

**必须会做**：

- `upsert(chunks)` 批量入库（含 embedding + payload 元数据）
- `search(query_vec, top_k, filter)` 带元数据过滤的检索
- `delete_by_doc_id` 按文档删除
- HNSW 参数：`M` / `ef_construction` / `ef_search` 的含义

**理解**：

- BiEncoder（embedding）vs CrossEncoder（rerank）分工
- 非对称检索：query 短、chunk 长，不同模型的影响

**产出**：完成"上传 → 解析 → 切片 → 向量化 → 入 Qdrant"完整链路，能在 Qdrant UI 看到数据。

---

### 模块 6 · 查询理解（先简后繁，0.5–1 天）

参考 **阶段四 3-5**

**主线 MVP**：query 直接 embedding。

**第二步加（性价比高）**：

- **Query Rewriting**：口语化 → 检索友好（让 LLM 改写一次）
- **历史对话指代消解**：把"它/这个"补全

**了解即可（先打标记）**：Multi-Query、HyDE、Step-back、Query Decomposition、Routing

**理解**：用户原始 query 几乎从不是好的检索 query；多调一次 LLM 的成本 vs 召回率收益。

---

### 模块 7 · 检索与混合检索 ⭐（1–2 天）

参考 **阶段四 3-6**

**主线分两步**：

**步骤 A（MVP）**：纯向量 top-k 检索（5–10 条）

**步骤 B（强烈建议加上）**：

- 加 BM25 稀疏检索（`rank_bm25` Python 库，零部署）
- **RRF（Reciprocal Rank Fusion）** 融合稠密 + 稀疏结果
- 元数据过滤（按 `doc_id` / 权限 / 时间范围）

**理解（中文场景关键）**：

- 中文场景必须混合检索——专有名词、产品编号、人名，纯向量很容易召回失败
- BM25 不死的原因

**进阶（打标记）**：Rerank（3-7，Qwen3-Reranker / bge-reranker-v2-m3 / Cohere）——主线打通后第一个要补的就是它，性价比最高的精度提升。

---

### 模块 8 · 上下文组装与生成 ⭐（1 天）

参考 **阶段四 3-8** + **阶段二 1-2 / 1-6**

**必学**：

- 把 top-k chunks 打包进 prompt 的格式（分隔符 + `[来源1]` / `[来源2]` 标号）
- **强制 Citation**：让模型输出 `[1][2]`，后处理映射回 chunk → 前端高亮原文
- **"不知道就拒答"** 的 system prompt 写法（不在召回内容里的问题如何让模型不瞎编）
- 流式生成 + 流式 citation（SSE 多事件类型）

**理解（生命线级）**：

- Citation 是企业 RAG 的可信度生命线
- 上下文 ≠ 越多越好，**Lost in the Middle** 真实存在（top-k 不要给太多）

**进阶（打标记）**：Contextual Compression、LLMLingua 压缩冗余

---

### 模块 9 · 服务化与基础容错（1 天）

参考 **阶段二 1-9** + **阶段四 3-13**

**必学**：

- `tenacity` 重试 + 指数退避 + 抖动
- 区分可重试（429 / 5xx）vs 不可重试（400 / 401）
- 超时分层：连接 / 读取 / 整体
- **Circuit Breaker**（断路器）的作用（理解，先不上）
- 结构化日志（loguru）：请求 ID、模型、token 用量、耗时
- 缓存层（先了解）：Embedding 缓存 / 答案缓存（语义缓存 `GPTCache`）
- 索引更新策略：增量 upsert + 文档级版本

**理解**：

- 重试策略不当 = 成本爆炸（失败也烧 token）
- RAG 是流水线，任何一环慢都拖垮 TTFT

---

### 模块 10 · 后置优化路线（打标记，主线打通后再回来）

按性价比排序，**逐个加**，每次加完跑评估对比：

1. **Rerank 精排**（3-7）—— 收益最大
2. **Contextual Retrieval**（3-3，Anthropic 方案）+ Prompt Caching
3. **Multi-Query / HyDE 查询改写**（3-5）
4. **Ragas 评估闭环**（3-12）—— 没评估的优化都是玄学
5. **Langfuse 全链路追踪**（3-13）
6. **权限过滤**（3-13，元数据 filter）
7. **Agentic RAG**（3-9）—— 复杂多跳问题
8. **GraphRAG / Text-to-SQL**（3-10）—— 跨文档关系 / 数据库问答
9. **视觉 RAG / ColPali**（3-11）—— 扫描件 / 图表密集文档

---

### 推荐学习节奏（约 2 周）

|周|天|内容|
|---|---|---|
|W1|D1–D2|模块 0 + 模块 1（异步、Pydantic、OpenAI SDK、embeddings）|
|W1|D3–D4|模块 2（FastAPI + SSE 骨架）|
|W1|D5–D7|模块 3 + 模块 4（解析 + 切片）|
|W2|D1–D2|模块 5（Qdrant 入库全链路打通）|
|W2|D3|模块 6 + 模块 7A（纯向量检索跑通）|
|W2|D4|模块 8（带引用流式生成）|
|W2|D5|模块 7B（加 BM25 + RRF 混合检索）|
|W2|D6–D7|模块 9 容错 + 端到端联调|

---

### 关键认知锚点（建议背下来）

1. RAG 三阶段：**Ingestion → Retrieval → Generation**（3-1）
2. 企业 RAG 效果差距 **80% 在解析阶段**（3-2）
3. 中文场景必须 **混合检索**（3-6）
4. **Citation 是可信度生命线**，Lost in the Middle 真实存在（3-8）
5. **没有评估的 RAG 优化都是玄学**（3-12）
6. Prompt Caching 是 2025–2026 成本下降 50%+ 的关键（1-2 / 1-5）
7. 输入 token vs 输出 token 单价差异巨大，决定架构选择（0-8 / 1-5）

---

需要我把这份目录导出成 Markdown 文件保存到你的工作区吗？