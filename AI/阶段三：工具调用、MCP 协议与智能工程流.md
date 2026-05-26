
> 本阶段目标:让大模型长出"手脚",从原生 Function Calling 出发,掌握 MCP 这一 2025 年起的事实标准协议,能编写自己的 MCP Server,接入 Cursor / Claude Code / Claude Desktop 等客户端,并在自研前端中实现工具状态可视化与人机协同审批。

---

### 2-1 Function Calling 原理深入【必学】

**学习重点:**

- `tools` / `tool_choice`(`auto` / `none` / `required` / 指定函数名)
- 工具 JSON Schema:`type` / `properties` / `required` / `enum` / 嵌套 object / array
- `strict: true` 严格 schema 模式
- `parallel_tool_calls` 并行工具调用
- 各家 Provider 工具协议差异:OpenAI `tool_calls` / Anthropic `tool_use` & `tool_result` / Gemini `functionCall`
- 工具描述(`description`)写作规范
- 工具 schema 设计 5 原则:动词命名、单一职责、必填最小化、枚举优先、错误返回结构化

**理解:**

- 模型如何"选择"工具:本质是预测下一个 token,被 schema 约束
- Tool Hallucination 的常见来源:描述模糊、参数命名歧义、过多无关工具
- 工具数量经验值:常用模型在 20-50 个工具后选择质量明显下降 → 引出工具路由

---

### 2-2 闭环工具执行循环(Agent Loop 雏形)【必学】

**学习重点:**

- 完整循环:用户消息 → 模型 → `tool_calls` → 本地执行 → `role:tool` 回填 → 模型 → ... → 最终回复
- Tool Dispatcher / Registry:工具注册表与动态路由
- 异步工具执行(`asyncio.gather` 并发跑多个工具)
- 工具超时控制(`asyncio.wait_for`)
- 工具结果序列化与大结果处理(截断 / 摘要 / 写文件传引用)
- 最大循环次数保护(防死循环)

**理解:**

- ReAct(Reasoning + Acting)模式与现代 tool use 的关系
- Agent Loop 是阶段三 Agent 框架的基础,这里先手写一遍

**实战:**

- 手写一个不依赖任何框架的 Agent Loop,支持多工具、并行调用、超时、最大轮数

---

### 2-3 工具异常处理与自我纠错【必学】

**学习重点:**

- 异常分类:参数校验失败 / 业务异常 / 系统异常 / 超时
- 结构化错误返回:`{"error": "...", "hint": "..."}`
- invalid args 自动重试 prompt
- Guardrail:工具白名单、参数范围校验、危险操作拦截
- HITL(Human-in-the-Loop):危险工具强制人工审批

**理解:**

- 让模型自己纠错 vs 程序兜底的边界
- "失败也是上下文":错误信息回填可显著提升下一轮调用质量

---

### 2-4 MCP 协议核心架构【必学】🔥

**学习重点:**

- MCP 三角色:**Host**(Claude Desktop / Cursor / VS Code)/ **Client**(Host 内的连接器)/ **Server**(暴露工具的进程)
- MCP 三类资源:**Tools** / **Resources** / **Prompts**
- Transport(2026 现状):
    - `stdio`(本地进程,最常用)
    - **Streamable HTTP**(远程 / 容器化部署主流,2025 替代旧 SSE transport)
    - 旧 `SSE transport` 兼容但已 deprecated
- 生命周期:`initialize` → `capabilities` 协商 → 调用 → `shutdown`
- JSON-RPC 2.0 消息格式
- **MCP Authorization(OAuth 2.1)**:远程 MCP Server 的标准鉴权方案
- **Elicitation**:Server 反向向用户提问(2025 新增)
- **Roots**:声明 Server 能访问的文件系统范围
- **Sampling**:Server 反向请求 Host 调用 LLM

**理解:**

- 为什么需要 MCP:把"N 客户端 × M 工具"的 N×M 集成,降为 N+M
- MCP 与 Function Calling 的关系:MCP 是工具**分发与发现协议**,Function Calling 是**调用协议**
- MCP 与 OpenAPI 的对比

---

### 2-5 用 FastMCP 构建自定义 MCP Server【必学】

**学习重点:**

- `FastMCP`(Python 官方推荐高层 SDK)
- 装饰器:`@mcp.tool()` / `@mcp.resource("scheme://...")` / `@mcp.prompt()`
- 工具参数类型注解 → 自动生成 JSON Schema
- 异步工具(`async def`)
- 上下文对象 `Context`:日志、进度上报、Sampling
- 本地文件访问、subprocess、HTTP 客户端等典型工具
- 启动方式:`mcp dev`(开发调试)/ `mcp run`(stdio)/ HTTP 部署
- **MCP Inspector** 调试工具
- `mcp.json` / `claude_desktop_config.json` 配置规范
- 远程 MCP Server 容器化部署(Docker + Streamable HTTP)
- 鉴权实现:静态 token / OAuth 2.1 / 与现有身份系统对接

**理解:**

- Tool / Resource / Prompt 三种暴露形态的取舍
- Server 如何被 Host 自动发现

**实战:**

- 编写一个支持文件搜索、Git 操作、命令执行的本地 MCP Server,用 MCP Inspector 调试通过

---

### 2-6 主流 Host 联动实战【必学】

**学习重点:**

- **Claude Desktop**:`claude_desktop_config.json` 配置
- **Claude Code (CLI)**:`.mcp.json` 项目级配置、`/mcp` 命令、subagent
- **Cursor**:`mcp.json`、Cursor Rules(`.cursorrules` / `.cursor/rules/*.mdc`)与 MCP 协同
- **VS Code**(原生 MCP 支持,2025 起)
- **Cline / Continue / Windsurf / Cody / Gemini CLI** 了解即可
- **Anthropic Skills**(2025 推出):`SKILL.md` + 资源文件的可组合能力包,与 MCP 互补

**理解:**

- Skills(静态能力包)与 MCP(动态工具)的分工
- 同一个 MCP Server 在不同 Host 中的表现差异

**实战:**

- 用同一个自研 MCP Server,在 Claude Desktop、Cursor、Claude Code 三个 Host 中分别跑通

---

### 2-7 工具沙箱与安全【必学】

**学习重点:**

- 权限分级:只读工具 / 受控写工具 / 危险工具
- 沙箱方案:容器(Docker)/ 子进程隔离 / VM / WASM(Pyodide / wasmtime)
- 文件系统隔离:chroot / 路径白名单 / read-only mount
- 网络隔离:无网络 / 代理白名单
- 命令执行危险面:shell 注入、路径遍历、删库操作
- 凭据隔离:短期 token / OAuth 代理

**理解:**

- "能力越大,审批越严"—— HITL 在哪些工具上必须开
- 间接 Prompt Injection:通过工具结果反向攻击模型,以及防御思路

---

### 2-8 Computer Use / Browser Use【选学但推荐】

**学习重点:**

- Anthropic Computer Use API(屏幕截图 + 鼠标键盘)
- Browser Use / Playwright MCP / Stagehand 浏览器自动化方向
- 视觉定位(coordinate)vs DOM 定位(selector)
- 操作循环的延迟、稳定性、断点重试

**理解:**

- 何时用 Computer Use,何时用专用 API:有 API 永远优先 API
- 价格与延迟成本

---

### 2-9 前端工具状态可视化与 HITL【必学】

**学习重点:**

- SSE 事件协议扩展(承接 1-8):
    - `tool_call_start` / `tool_call_approval` / `tool_call_progress` / `tool_call_result` / `tool_call_error`
- 工具状态机:`pending → awaiting_approval → running → success / error / cancelled`
- 动态 JSON Schema → 卡片自动渲染
- 流式 markdown 与卡片混合渲染(打字机 + 卡片交错)
- 审批弹层 / Inline 审批按钮(参考 Claude Code / Cursor 的批准 UI)
- 工具执行时间线(Timeline)
- 可中断 / 可回退:用户在工具未执行时可改参数或拒绝
- 工具结果折叠展示

**理解:**

- 把 Agent 动作"看得见、拦得住"是 2026 主流 AI 产品的基础体验
- 前端不只是渲染层,是 HITL 的执行层

---

### 阶段实战项目

**《基于 MCP 协议的私有代码库 Agent 助手》**

- 编写一个 MCP Server,暴露:代码搜索(ripgrep / 向量搜索)、Git 操作、文件读写(路径白名单)、命令执行(白名单 + 沙箱)
- 在 Claude Desktop / Cursor / Claude Code 三个 Host 中分别接入
- 同时实现自研前端 + FastAPI 后端,调用同一个 MCP Server,展示:流式工具调用过程、HITL 审批 UI、工具状态时间线、错误自我纠错
- 加 OAuth 2.1 鉴权,验证远程 Streamable HTTP 部署