如果目标只是：**企业级文档 AI + 通用大模型私有化部署**，你不需要把整套 LangGraph / 多 Agent / MCP 都学完。

够用范围是：

**学到阶段三为主，再补阶段五的一部分。**

具体来说：

1. **阶段一：全学**  
    这是底盘，必须会。
    
    - LLM SDK / OpenAI-compatible API
    - Prompt
    - Structured Output
    - Pydantic
    - Token / 上下文管理
    - Streaming
    - FastAPI
    - SSE
2. **阶段二：只学工具调用前半部分**  
    够学：
    
    - Function Calling 原理
    - 工具执行闭环
    - 工具异常处理
    
    MCP 可以暂时跳过。企业文档 AI 不一定需要 MCP，除非你要让模型操作本地文件系统、数据库、Git、企业内部工具。
    
3. **阶段三：重点全学**  
    这是你的核心战场。
    
    必须掌握：
    
    - 文档加载：PDF / Word / Markdown / HTML
    - 数据清洗
    - 文档切片
    - Embedding
    - 向量数据库：Qdrant / Milvus / pgvector 选一个
    - Hybrid Search
    - Rerank
    - Metadata filter
    - Citations 引用来源
    - RAG 评估
    - FastAPI 封装
    
    这个阶段学完，你就已经能做企业文档 AI 了。
    
4. **阶段四：基本可以不学**  
    LangGraph 不是企业文档 AI 的必需品。
    
    除非你要做：
    
    - 多步骤审批流
    - 多 Agent 协作
    - 长流程任务
    - 人工介入
    - 可恢复工作流
    
    否则先跳过。最多了解一下 State / Node / Edge 就够。
    
5. **阶段五：挑生产部署相关的学**  
    必学：
    
    - Docker
    - docker-compose
    - FastAPI 部署
    - vLLM / Ollama / llama.cpp 模型服务部署
    - Qdrant / pgvector 部署
    - Redis 缓存
    - API 限流
    - 超时重试
    - 日志
    - 权限控制
    - 数据备份
    - GPU 显存估算