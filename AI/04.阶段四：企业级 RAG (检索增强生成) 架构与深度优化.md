
> 本阶段目标:跳出"切片 + 向量库 + LLM"的玩具 RAG,掌握 2026 年企业级 RAG 的完整链路:文档解析 → 智能切片 → 查询理解 → 混合检索 → 精排 → 生成 → 评估 → 可观测,并理解 Agentic RAG / GraphRAG / 视觉 RAG 的演进方向。

---

### 3-1 RAG 全景与现代演进【必学】

**学习重点:**

- 经典 RAG 三阶段:Ingestion / Retrieval / Generation
- 2025-2026 RAG 演进谱系:
    - **Naive RAG**(单次检索)
    - **Advanced RAG**(查询改写 + 混合检索 + Rerank)
    - **Modular RAG**(可插拔组件)
    - **Agentic RAG**(检索作为工具,多步推理)
    - **GraphRAG**(图结构增强)
    - **Visual / Multimodal RAG**(ColPali 等视觉范式)
- 核心指标:Grounding / Faithfulness / Citation / Hallucination
- 框架选型:LlamaIndex / LangChain / Haystack / **Dify** / **RAGFlow** / 自研

**理解:**

- 何时不需要 RAG(短上下文 + Prompt Caching 可能更便宜)
- RAG vs 长上下文 vs 微调的取舍

---

### 3-2 文档解析与数据准备【必学】

**学习重点:**

- 现代文档解析工具栈:
    - **MinerU**(国产,PDF / 公式 / 表格效果好)
    - **Docling**(IBM 开源)
    - **LlamaParse**(LlamaIndex 商业版)
    - **Unstructured.io**
    - **PyMuPDF / pdfplumber**(轻量基础)
- 多模态文档元素提取:标题层级、段落、表格、图片、公式、代码块
- 表格处理:转 Markdown / HTML / JSON,或单独索引
- 图片处理:Vision LLM caption + OCR 兜底
- OCR 工具:PaddleOCR / Tesseract / GOT-OCR2.0 / Vision LLM 直接读
- HTML / Markdown / DOCX / PPTX 解析
- 元数据抽取:来源、章节、页码、作者、日期、权限标签

**理解:**

- "Garbage In, Garbage Out":企业 RAG 80% 的效果差距在解析阶段
- 表格直接切片 = 召回灾难,必须单独处理
- 元数据的价值不仅用于过滤,也用于召回时增强

**实战:**

- 用 MinerU 或 Docling 解析一批真实企业 PDF,对比纯文本提取效果

---

### 3-3 切片策略【必学】

**学习重点:**

- 基础切片:`RecursiveCharacterTextSplitter`、固定窗口 + overlap
- **结构化切片**:按标题 / 章节 / 段落切(基于解析出的结构)
- **语义切片**(Semantic Chunking):基于句向量相似度断点
- **Hierarchical / Parent-Child Chunking**:小块用于检索,大块用于生成
- **Late Chunking**(Jina 提出,2024):先长文本嵌入,再切分向量
- **Contextual Retrieval**(Anthropic 2024):用 LLM 给每个 chunk 生成上下文摘要再嵌入
- 切片大小经验值:中文 200-500 字 / 英文 256-512 tokens,具体看模型
- overlap 比例:10%-20%
- chunk 元数据增强:标题前缀、章节路径、文档摘要

**理解:**

- chunk 大小与召回率、精确率、生成质量的三角权衡
- Contextual Retrieval 为什么是 2025 的"近免费午餐"(配合 Prompt Caching 成本极低)
- 何时需要 parent-child(法律 / 技术手册典型场景)

---

### 3-4 向量化与向量数据库【必学】

**学习重点:**

- Embedding 模型选型(承接阶段 0):BGE-M3 / Qwen3-Embedding / BCE / jina-v3
- 主流向量库对比:

|数据库|定位|适用|
|---|---|---|
|**Milvus**|分布式生产首选|大规模、强需求|
|**Qdrant**|性能强 + 易用|中大型生产|
|**Weaviate**|多模态友好|模块化场景|
|**pgvector / pgvecto.rs**|Postgres 扩展|已有 PG 栈|
|**LanceDB**|嵌入式 / 列式|本地、小中型|
|**Chroma**|极简本地|原型、个人|
|**Elasticsearch / OpenSearch**|全文 + 向量|已有 ES 栈|
|**Vespa**|复杂排序|搜索厂大规模|

- HNSW / IVF / DiskANN 索引原理与参数(`ef_construction` / `ef_search` / `M`)
- 量化:PQ / SQ / Binary Quantization(向量库存储成本降 4-32 倍)
- Metadata Filter / Payload / 复合索引
- 批量插入、持久化、备份、热更新
- 多租户隔离方案

**理解:**

- 选库的真实决策因素:数据量、QPS、过滤复杂度、运维栈、成本
- 向量维度与存储成本:1024 维 100 万条 ≈ 4GB(float32)

---

### 3-5 查询理解【必学】

**学习重点:**

- **Query Rewriting** 查询改写:口语化 → 检索友好
- **Multi-Query**:生成多个变体查询并行检索
- **HyDE**(Hypothetical Document Embeddings):让 LLM 先编造一个"理想答案"再嵌入检索
- **Step-back Prompting**:抽象成更高层问题再检索
- **Query Decomposition** 查询分解:复杂问题拆子问题
- **Routing**(承接 3-6):意图分类、知识库路由、闲聊兜底
- 上下文重写:基于历史对话补全指代("它"、"这个"指向什么)

**理解:**

- 用户原始 query 几乎从不是好的检索 query
- 查询理解的成本(多调一次 LLM)vs 收益(召回率提升)

---

### 3-6 混合检索【必学】🔥

**学习重点:**

- **稠密向量检索**(语义)
- **稀疏检索**:BM25 / SPLADE / BGE-M3 自带稀疏向量
- **RRF**(Reciprocal Rank Fusion)融合
- 加权融合(weighted score)
- **Late Interaction**(ColBERT / ColPali / BGE-M3 ColBERT 模式)
- 元数据过滤先于 / 后于向量检索的差异
- 检索结果去重(near-duplicate)

**理解:**

- 中文场景为什么必须混合检索(专有名词、产品编号、人名,纯向量很容易召回失败)
- BM25 不死的原因

---

### 3-7 重排序(Rerank)【必学】

**学习重点:**

- Rerank 主流模型(2026):
    - **Qwen3-Reranker**(国产首选)
    - **bge-reranker-v2-m3**
    - **Cohere Rerank 3.5**(闭源 API)
    - **Jina Reranker v2**
    - **FlashRank**(轻量)
- CrossEncoder 原理与延迟特点
- LLM-based Rerank(用大模型直接打分,贵但效果最好)
- 召回 → Rerank 流水线参数:召回 top 50-100 → rerank 取 top 5-10
- Rerank 失败兜底(超时回退原始顺序)

**理解:**

- Rerank 是性价比最高的精度提升手段之一
- Rerank 模型 ≠ 越大越好,看延迟预算

---

### 3-8 上下文压缩与生成【必学】

**学习重点:**

- 召回内容打包到 prompt 的格式(分隔符、来源标注)
- **Contextual Compression**:LLM 摘要 / `LLMLingua` 压缩冗余
- **Citation 引用生成**:让模型输出 `[1][2]` 标注,后处理映射回 chunk
- 流式输出 + 流式引用(SSE 事件协议扩展:`citation` 事件)
- "我不知道"训练:不在召回内容里的问题如何让模型拒答

**理解:**

- Citation 是企业 RAG 的可信度生命线
- 上下文 ≠ 越多越好,Lost in the Middle 真实存在

---

### 3-9 Agentic RAG【必学】

**学习重点:**

- 把"检索"做成 Agent 工具,模型自主决定:是否检索、检索几次、用哪个知识库
- 多步检索 / 迭代检索(基于上一轮结果改写下一轮 query)
- Self-RAG / Corrective RAG(CRAG)思路:模型自评检索质量,不够则重检索
- 与阶段四 Agent 框架的衔接

**理解:**

- Agentic RAG 适合复杂多跳问题,简单事实问题反而过度设计
- 延迟 vs 准确率的取舍

---

### 3-10 GraphRAG 与结构化数据 RAG【必学】

**学习重点:**

- **GraphRAG**:微软方案 / **LightRAG** / **nano-graphrag**
- 流程:实体抽取 → 关系抽取 → 社区检测 → 社区摘要 → 图遍历检索
- 适用场景:跨文档关系问题、全局总结、人物事件追踪
- **Text-to-SQL** / **Text-to-Cypher**:结构化数据 RAG
- **TableRAG**:表格专用检索
- Hybrid:文档 RAG + Graph + SQL 联合

**理解:**

- 文档型 RAG 答"局部细节",GraphRAG 答"全局结构"
- 企业 50% 数据在数据库,Text-to-SQL 是必修

---

### 3-11 多模态 / 视觉 RAG【选学但推荐】

**学习重点:**

- **ColPali / ColQwen**:直接对 PDF 页面截图嵌入,绕开 OCR
- 多模态 embedding:CLIP / SigLIP / Jina-CLIP
- Vision LLM 作为"读图器"(Qwen3-VL / InternVL3 / GPT-4o)
- 图文混排文档的端到端方案

**理解:**

- 视觉 RAG 对扫描件 / 图表密集文档优势巨大
- 成本:页面级嵌入向量量大,需要量化

---

### 3-12 RAG 评估【必学】(原"选学"升级)

**学习重点:**

- **Ragas** 核心指标:Faithfulness / Answer Relevancy / Context Precision / Context Recall
- **TruLens** / **DeepEval** / **promptfoo** 评估框架
- **LLM-as-a-Judge** 范式与陷阱(位置偏好、长度偏好)
- 合成评估集:用 LLM 从文档生成 QA 对
- 人工标注 vs 自动评估的混合策略
- 检索单独评估(Recall@K / MRR / NDCG)vs 端到端评估

**理解:**

- 没有评估的 RAG 优化都是玄学
- 建立评估集 → 改造 → 跑评估 → 看数字 → 决策,是企业 RAG 的正确节奏

---

### 3-13 RAG 服务化与可观测【必学】

**学习重点:**

- FastAPI 封装:检索服务 / 生成服务可拆可合
- 流式生成 + 流式引用(SSE 事件协议)
- 缓存层:Embedding 缓存 / 检索结果缓存 / 答案缓存(语义缓存 `GPTCache`)
- 索引更新策略:全量重建 / 增量 upsert / 文档级版本
- 权限过滤(行级权限映射到 metadata filter)
- Langfuse / Phoenix 追踪 RAG 全链路(query → 检索 → rerank → 生成)
- 前端:引用光源高亮、原文跳转、检索过程可视化

**理解:**

- RAG 是流水线,任何一环慢都拖垮 TTFT
- 权限是企业 RAG 上线的硬门槛

---

### 阶段实战项目

**《企业级私有多格式文档智能检索问答平台》**

- 文档解析:MinerU / Docling 处理 PDF / Word / PPT / 扫描件
- 切片:结构化切片 + Contextual Retrieval(给 chunk 加上下文)
- 索引:Milvus 或 Qdrant + BM25 双索引
- 查询理解:Multi-Query + Step-back
- 混合检索 + Qwen3-Reranker 精排
- 生成带引用,流式输出
- 同时支持 Agentic RAG 模式(模型自主多步检索)
- 接入 Ragas 评估管道,跑通"评估 → 改造 → 再评估"闭环
- 前端实现引用光源点击高亮原文、检索过程可视化
- Langfuse 全链路追踪