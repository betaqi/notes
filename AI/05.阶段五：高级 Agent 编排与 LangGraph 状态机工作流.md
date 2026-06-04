Agent 编排与状态机工作流(重写版)

> 本阶段目标:跳出"调 API + Tool Use 循环"的初级形态,掌握现代 Agent 工程的核心:状态图、Context Engineering、记忆系统、HITL、多 Agent 协作、持久化执行、可观测与评估。学完能用任意主流框架(LangGraph / OpenAI Agents SDK / PydanticAI 等)搭建可上生产的 Agent。

---

### 4-1 Agent 设计模式与框架全景【必学】

**学习重点:**

- **Workflow vs Agent**(Anthropic 定义):
    - Workflow:LLM 与工具按预定义代码路径编排
    - Agent:LLM 自主决定流程与工具调用
- 经典工作流模式:
    1. **Prompt Chaining**(串行)
    2. **Routing**(分流)
    3. **Parallelization**(并行 / 投票)
    4. **Orchestrator-Workers**(主管 + 工人)
    5. **Evaluator-Optimizer**(生成 + 评审循环)
- 2026 主流 Agent 框架对比:

|框架|定位|特点|
|---|---|---|
|**LangGraph**|状态图,生产首选|灵活、生态全、Checkpoint 强|
|**OpenAI Agents SDK**|OpenAI 官方|轻量、Handoff 抽象|
|**PydanticAI**|类型安全|Pydantic 风格,DX 好|
|**CrewAI**|角色协作|多 Agent 抽象优雅|
|**AutoGen**|微软,会话式|学术与原型|
|**Google ADK**|Google 官方|与 Gemini / A2A 协同|
|**LlamaIndex Workflows**|事件驱动|RAG 一体|
|**smolagents**|Hugging Face|代码即 Action 范式|
|**Dify / Coze / FastGPT**|低代码|业务侧拖拽|

**理解:**

- 选 Workflow 还是 Agent:可预测性 vs 灵活性的取舍
- 框架是脚手架,**底层范式(状态、循环、工具、记忆、HITL)是通用能力**
- "代码即工作流" vs "DAG / 状态图 / 事件驱动"三种风格

---

### 4-2 LangGraph 核心:Node / Edge / State【必学】🔥

**学习重点:**

- `StateGraph` / `START` / `END`
- `add_node` / `add_edge` / `add_conditional_edges`
- 节点函数签名:`(state) -> partial_state`
- 编译 `graph.compile()` 与可视化(`get_graph().draw_mermaid()`)
- Router 节点与条件路由
- `ToolNode` 预制节点
- 子图(Subgraph)嵌入
- Streaming 模式:`values` / `updates` / `messages` / `custom`(2025 推出)

**理解:**

- 为什么图比链强:循环、条件、并行天然支持
- 节点应保持纯函数,副作用隔离到工具层

---

### 4-3 State 设计与 Context Engineering【必学】

**学习重点:**

- `TypedDict` / Pydantic State
- **Reducer**(`Annotated[list, add_messages]`)合并语义
- 多通道 State:`messages` / `scratchpad` / `artifacts` / `user_context`
- **Context Engineering**(2025 关键词):
    - 给模型"恰到好处"的上下文:不多不少、结构清晰
    - 上下文打包优先级:任务 > 工具反馈 > 摘要历史 > 完整历史
    - 上下文剪枝、压缩、检索式注入
- 长跑 Agent 的状态膨胀控制:消息修剪、摘要节点、关键事实抽取

**理解:**

- Prompt Engineering 是"句子级别"工程,Context Engineering 是"信息系统级"工程
- 上下文质量 = Agent 质量上限

---

### 4-4 ReAct 与 Tool Calling Agent【必学】

**学习重点:**

- ReAct 范式:Thought / Action / Observation 循环
- 现代实现:不再显式提示 Thought,而是依赖模型原生 tool calling + reasoning
- 单步与多步规划
- 迭代次数控制、超时熔断
- **Code Execution as Tool**(Anthropic 2025 推):让模型写代码调多个工具,减少多轮 tool call 开销

**理解:**

- ReAct 是底层范式,被各框架内化
- "一个 code execution 工具 + 一堆 import 库"在某些场景优于"100 个细粒度 MCP 工具"

---

### 4-5 记忆系统【必学】

**学习重点:**

- 三层记忆:
    - **短期记忆**(会话内 messages)
    - **长期记忆**(跨会话用户偏好、事实)
    - **情景记忆 / 语义记忆 / 程序记忆**(认知科学分类)
- 主流方案:
    - **LangMem**(LangChain 出品)
    - **mem0**
    - **Zep**(基于知识图谱)
    - **Letta / MemGPT**(分层记忆)
- 记忆写入策略:每轮写 / 总结写 / LLM 决策写
- 记忆检索:向量召回 / 时间召回 / 重要性召回
- 记忆冲突与更新(用户改名了怎么办)

**理解:**

- 记忆 = 跨会话的 RAG,本质还是检索
- 何时该用记忆,何时该用每轮显式注入

---

### 4-6 持久化与 Checkpoint【必学】

**学习重点:**

- `MemorySaver` / `SqliteSaver` / `PostgresSaver`
- `thread_id` 概念,会话恢复
- Checkpoint 存什么:State 快照 + 节点位置 + 待执行任务
- 时间旅行(Time Travel):列出历史 checkpoint,回滚或分支
- 重放(Replay)与从指定 checkpoint 修改 State 后继续
- **Durable Execution** 进阶:
    - LangGraph Platform / Temporal / Inngest / Restate
    - 用于长跑(小时-天级)Agent、外部事件等待、跨进程容错

**理解:**

- Checkpoint 不只是"恢复",更是调试与分支实验的基础
- 长跑 Agent 必须解决:进程崩溃、机器重启、外部回调等待

---

### 4-7 Human-in-the-Loop【必学】🔥

**学习重点:**

- `interrupt()` 中断与 `Command(resume=...)` 恢复(LangGraph 现代 API)
- HITL 三态:
    - **审批**(同意 / 拒绝工具调用)
    - **编辑**(修改工具参数 / 模型输出)
    - **补全**(用户提供模型缺失的信息)
- Streaming 中将 interrupt 事件推到前端
- 前端审批 UI(承接阶段二 2-9)
- 异步 HITL(用户离开后回来,基于 Checkpoint 恢复)

**理解:**

- HITL 不是"安全功能",是 Agent 产品的核心 UX
- 何时必须 HITL:写操作、外部副作用、不可逆动作、低置信度

---

### 4-8 多 Agent 协作【必学】(原"选学"升级)

**学习重点:**

- 协作模式:
    - **Supervisor**(主管路由)
    - **Hierarchical**(分层管理)
    - **Handoff**(交接,OpenAI Agents SDK 原生)
    - **Network / Swarm**(任意互调)
    - **Map-Reduce**(并行分发 + 聚合)
- 子图(Subgraph)与 Agent 隔离
- Agent 之间的"消息" vs "共享 State"
- **A2A 协议**(Google 2025 推出的跨厂商 Agent 通信标准),与 MCP 的关系:MCP 是 Agent ↔ 工具,A2A 是 Agent ↔ Agent
- 上下文隔离:每个子 Agent 应有独立 scratchpad,避免互相污染

**理解:**

- 多 Agent 收益:任务专精、上下文隔离、并行
- 多 Agent 代价:协调成本、错误传播、调试难度
- 何时"一个强 Agent + 多工具"优于"多 Agent"

---

### 4-9 反思与自我纠错【必学】

**学习重点:**

- Reflection 节点:让 LLM 评审上一轮输出
- Critique Loop / Evaluator-Optimizer 模式
- Self-Refine(自我迭代)
- 重试规划:失败原因分类 → 改写策略
- 死循环熔断:最大迭代、状态去重、预算控制(token / 时间 / 成本)

**理解:**

- 反思有效但贵,关键路径才用
- 容易陷入"反思也错"的死循环,必须配硬熔断

---

### 4-10 Streaming 与前端可视化【必学】

**学习重点:**

- LangGraph 4 种 stream 模式的取舍:
    - `values`:每步完整 state
    - `updates`:每步增量
    - `messages`:token 级流式
    - `custom`:节点内自定义事件
- SSE 事件协议扩展(承接前几阶段):
    - `node_start` / `node_end` / `state_update` / `interrupt` / `agent_handoff` / `final`
- 前端可视化:图节点实时高亮、状态流转动画、Agent 间消息时间线
- **Generative UI / AG-UI**(2025 趋势):后端流式驱动前端组件渲染

**理解:**

- Agent 产品的差异化竞争点正在从"模型能力"转向"过程可视化与可控性"

---

### 4-11 Agent 评估与可观测【必学】

**学习重点:**

- Agent 评估与 RAG 评估的差异:**Trajectory Evaluation**(过程评估)而非单点
- 评估维度:
    - 任务成功率(end-to-end)
    - 工具选择正确率
    - 步数 / 成本 / 延迟
    - 路径合理性(LLM-as-Judge 评审轨迹)
- 工具栈:**LangSmith** / **Langfuse** / **Arize Phoenix** / **Braintrust** / **W&B Weave**
- 在线评估 vs 离线评估
- Prompt 版本管理与 A/B
- 生产 Trace 回灌评估集

**理解:**

- 没有 trajectory 评估的 Agent 优化都是猜
- 失败 trace 是最值钱的训练数据(改 prompt / 改工具描述)

---

### 阶段实战项目

**《带 HITL 与多 Agent 协作的软件工程助手》**

- 三 Agent 编排:**产品经理 Agent**(需求澄清) → **架构师 Agent**(方案设计) → **程序员 Agent**(代码生成)
- 用 LangGraph 实现,关键节点 HITL 审批
- 长期记忆:记住用户偏好的技术栈、命名风格
- Checkpoint 持久化到 Postgres,支持中断后第二天继续
- Code Execution 工具(沙箱),程序员 Agent 自测代码
- 前端:实时高亮图节点、HITL 弹窗、Agent 间消息时间线、引用与代码差异面板
- 接入 Langfuse 全链路追踪 + Trajectory 评估
- 同一套需求改用 OpenAI Agents SDK 重写一遍,体会框架差异