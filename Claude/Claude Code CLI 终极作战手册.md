
---

## 一、 核心概念：Skills 与会话隔离

### Skills

1. **本质**：存在文件里的指令集（带 YAML frontmatter 的 `SKILL.md`），Claude 读取后知道"用什么工具、按什么步骤做事"。
2. **加载机制**：默认采用 **progressive disclosure** —— 模型只看到 skill 的 `name` 和 `description`，决定要用时才读 `SKILL.md` 全文，"挂着不用"几乎零成本。
3. **限制自动触发**：在 frontmatter 加 `disable-model-invocation: true`，效果是 skill 对模型**完全不可见**，只能由你手动 `/<skill-name>` 触发。适合有副作用的危险操作（部署、删除、付费）。

### 会话隔离

1. **Worktree**：git 的功能，解决**文件系统**问题 —— 同一 repo 的不同分支放在不同目录，桌面版每个新会话自动建一个。
2. **Subagents**：Claude Code 的功能，解决**任务隔离**问题 —— 主 agent 通过 Task 工具调起专家子 agent，子 agent 有独立上下文，结果汇总回主线，不污染主会话。

---

## 二、 项目记忆：`/init` 与 `/memory`

Claude Code 每次在项目根目录启动时**自动、强制读取** `CLAUDE.md`，所以这是约束行为的最高杠杆点。

- **一键初始化：`/init`** 在新项目根目录运行，自动检测技术栈（Vue/React/Node 等），生成定制化的 `CLAUDE.md` 模板。
- **编辑记忆：`/memory`** 直接打开 `CLAUDE.md` / 各级记忆文件的编辑器，**手动改**。
- **追加规则**：直接对话告诉 Claude "把这条规则加进项目 CLAUDE.md：处理看板组件更新时使用 shallowRef 而不是 ref"，它会帮你写进去。

记忆按优先级从低到高有四级：

- `~/.claude/CLAUDE.md` — 用户全局
- `<project>/CLAUDE.md` — 项目级（提交 git）
- `<project>/CLAUDE.local.md` — 项目本地（不提交）
- 在 CLAUDE.md 里用 `@path/to/file.md` 引用外部文件

---

## 三、 为什么"脚本库 / CLAUDE.md"在 CLI 下是最高杠杆？

**你的脚本库就是 Claude 的手脚，`CLAUDE.md` 就是它的指挥官。**

### 1. `CLAUDE.md` 不是备忘录，是 CLI 的启动配置文件

每次启动强制读取。它会按里面写的 `Build & Test Commands` 自动跑编译和单测。重构高性能 Canvas 时，在 `CLAUDE.md` 里写死："_Canvas 渲染主循环禁止任何无谓的内存分配（如 new 对象），必须实现对象池复用_"，Claude Code 在改代码时绝对不敢犯错。

### 2. `scripts/` 是给 Claude CLI 递过去的"终极武器"

Claude Code 拥有直接执行 Bash 命令的权限。

- **传统做法**：让 Claude 肉眼看代码哪里性能不好。
- **CLI + 脚本库**：本地写一个性能审计脚本，对 Claude 说："_运行 `node scripts/perf-audit.js`，根据输出报表把卡顿最严重的三个组件重构成异步组件，并把数据预处理逻辑搬到 Web Worker。_"
- Claude CLI 自己跑脚本 → 看报错 → 改代码 → 再跑测试，直到脚本不再报警。**这就是把经验产品化。**

---

## 四、 上下文压缩：`/context`、`/compact`、`/btw`

不要等终端卡顿才处理，原生的上下文管理指令是：

- **监控：`/context`** 随时查看当前 token 占用与上下文分布。
- **压缩：`/compact`** 把历史对话提炼成精简摘要，清空冗余的 tool-results 日志，瞬间释放上下文，同时保证它不会"失忆"。
- **题外话：`/btw [问题]`** 在写核心逻辑时突然想问个不相干的八卦（"这个配置项在 Webpack 4 里叫什么？"），用 `/btw`。Claude 会在独立的微型沙盒里回答，回答完即丢弃，**绝对不污染主线项目的 token 树**。
- **彻底清空：`/clear`** 适合任务切换时用。

---

## 五、 深度思考：`/effort`

控制 Claude 思维链预算（在 Opus 4.6/4.7 上替代旧的 `budget_tokens`）。

|级别|适用场景|
|---|---|
|`/effort low`|加类型声明、改样式、写单测模板 — 求快求省|
|`/effort medium`|默认平衡|
|`/effort high`|中等复杂度重构|
|`/effort xhigh`|跨模块重构、状态机设计|
|`/effort max`|Web Worker 调度、Canvas 渲染层重构、复杂算法|
|`/effort auto`|恢复默认|

prompt 里写 `ultrathink` 触发最大思考预算（≈31999 tokens），适合架构推演。

---

## 六、 文件噪音与隐私隔离

> 关键：密钥/凭证**必须**用 `.claude/settings.json` 的 `permissions.deny` 守住。`.claudeignore` 目前还有多个未修复 bug（[#16704](https://github.com/anthropics/claude-code/issues/16704)、[#36163](https://github.com/anthropics/claude-code/issues/36163)、[#51105](https://github.com/anthropics/claude-code/issues/51105)），不能依赖它防泄露。

### 隔离编译噪音

`.next/`、`dist/`、`build/`、几兆的 mock JSON 拉进上下文不仅吃满 token，还会污染生成质量。在 `.claude/settings.json` 写：

```json
{
  "permissions": {
    "deny": [
      "Read(./dist/**)",
      "Read(./.next/**)",
      "Read(./out/**)",
      "Read(./build/**)",
      "Read(./public/**/*.png)",
      "Read(./public/**/*.mp4)",
      "Read(./mock/huge-data-*.json)"
    ]
  }
}
```

### 隐私死守线（绝对不能漏）

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./*.pem)",
      "Read(./*.key)",
      "Read(./secrets/**)",
      "Read(./.git/**)",
      "Read(./.cursor/**)"
    ]
  }
}
```

确保 `Anthropic_API_Key`、Webhook 密钥、本地数据库密码对 Claude CLI 彻底隐形。

---

## 七、 工程级权限与监控

Claude Code 拥有直接执行 Bash 和修改本地文件的特权。为防止"大力出奇迹"删错东西或死循环烧 token：

### 1. 权限模式

启动时通过 `--permission-mode` 控制：

|模式|行为|
|---|---|
|`default`|敏感操作弹确认（默认）|
|`acceptEdits`|自动批准文件编辑，其他仍需确认|
|`plan`|只读探索，不修改任何东西|
|`bypassPermissions`|全部放行（危险）|

复杂重构盯盘 → 保持 `default`；批量刷单测模板 → `--permission-mode acceptEdits`，无需频繁敲回车。

### 2. 用量监控

- API Key 用户：`/cost` 看会话累计花销
- Pro / Max 订阅：`/stats` 看用量统计
- 任何用户：`/context` 看 token 占用

长会话警惕"上下文滚雪球" —— 失败尝试会留在上下文里 token 双倍计，该 `/compact` 就 `/compact`。

---

## 八、 应对大文件代码截断

让 Claude 重构几百行的状态机或 Canvas 组件时，可能因单次输出 token 限制断在半空。

- **错误姿势**：慌张地敲"继续"、"接着写" — 容易丢上下文、对不齐括号。
- **正确姿势**：
    
    > "继续，从第 X 行的 `renderFrame()` 函数接着往下写，保持原有缩进，不要缩水。"
    

它会用 Edit 工具精准嵌入补全代码，不会重写整个文件，极大节省输出成本。

---

## 九、 AI 原生 Git 工作流

Claude Code 深度集成 Git。改完代码不要切回常规终端自己 commit。

### 1. 审查：`/diff`

它说"重构好了"时**不要盲信**。直接 `/diff`，原地审查所有增删。

### 2. 语义化自动 commit

> "把当前修改提交到 Git。要求遵循 Angular 规范（如 `perf(canvas): optimize animation frame loop with object pool`），并自动生成清晰的 commit message。"

它会调用本地 git 写好规范提交信息并完成 stage + commit。

### 3. PR 前的双保险

- `/code-review` — 审 diff 找逻辑 bug
- `/security-review` — 安全审查

---

## 十、 无人值守 TDD 闭环

直接在常规终端（Zsh/Bash）下复合单行命令：

```bash
claude "运行 npm run test:unit，如果失败，请查看报错信息并修复 src/components/ 下的对应组件，然后重新运行测试，直到全部通过为止。"
```

- **机制**：Claude CLI 自主接管状态机，跑测试 → 抓报错 → 改代码 → 再跑测试，循环到 exit code 0。
- **爽点**：你切出去喝杯咖啡，回来时终端已经卡在测试通过的状态。

---

## 十一、 多 Agent 协作与批处理

整个 Monorepo 或多模块架构审计，单线程聊天会耗死你。Claude Code 真正恐怖的是**分布式能力**：

|方式|用法|
|---|---|
|**Task 工具**|主 agent 自动调起子 agent 并发执行|
|**`/agents`**|管理自定义 subagent 定义（指定专长、可用工具、模型），主 agent 按需调用|
|**`/batch`**|跨文件大型重构（"把 src/utils 下所有老旧的 axios 封装统一改成原生 fetch"），自动拆任务到独立 worktree 并发|
|**Agent view**|同时调度多个会话（修 bug、审 PR、查 flaky test），状态面板汇总|
|**多 Worktree 会话**|一个分支一个 Claude 会话，互不干扰|

---

## 十二、 复刻 Cookbook：从"写代码"变成"写自动化管线"

用 CLI 之后，看 [anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks) 的视角要从"AI 怎么写代码"切换到"**我怎么用脚本去操纵 AI 做出高级行为**"。

- **Tool-result 清理**：CLI 跑 `git diff`、`npm test` 会产生大量终端噪音，迅速吃 token。学 Cookbook 里压缩、归纳工具结果的 pattern，写进 CLAUDE.md 或定期 `/compact`。
- **Prompt caching 边界**：高频交互时如果每改一行都全量重读几万行组件库，token 费用会爆炸。Cookbook 详细讲了如何设计长文本结构最大化触发 prompt caching，让 Claude 只对变动部分付费。

---

## 十三、 本地脚本通往 MCP

当本地 `scripts/` 需要跨项目复用或保持状态时，封成 MCP 让 Claude 在任何路径下开箱即用。

### 推荐：用命令配置（Claude Code ≥ 2.1.1）

```bash
# 用户级（全局可用）
claude mcp add my-perf-auditor --scope user -- node /absolute/path/to/scripts/mcp-perf-bridge.js

# 项目级（写到 .mcp.json，会被 git 提交）
claude mcp add my-perf-auditor --scope project -- node ./scripts/mcp-perf-bridge.js

# 列出已配置
claude mcp list

# 从 Claude Desktop 导入
claude mcp add-from-claude-desktop
```

### 手写 JSON 的实际路径

|范围|文件|
|---|---|
|用户级|`~/.claude.json`|
|项目级|`<project>/.mcp.json`|

`.mcp.json` 格式：

```json
{
  "mcpServers": {
    "my-perf-auditor": {
      "command": "node",
      "args": ["./scripts/mcp-perf-bridge.js"]
    }
  }
}
```

配好后用 `/mcp` 查看连接状态与每个 server 的工具数量。

---

## 十四、 Academy：API 限制与特权说明书

对 CLI 用户，Academy 是用来理解 Claude Code 在幕后调什么接口、突破黑盒限制。

- **理解底层限流**：`/agents` 派发并发 subagent 重构 Monorepo 时，频繁遇到 `Rate Limit` 报错，去 Academy 查 `Message Batches API` 机制。不紧急的批量重构其实不该在 CLI 里死等，写十几行 Python 走 Batch API 扔后台，打折又省心。

---

## 出门前 Checklist

- [ ] 项目根有 `CLAUDE.md`，写明 build/test 命令和硬约束
- [ ] `.claude/settings.json` 配好 `permissions.deny`，密钥/`.env` 全部封死
- [ ] 自己的性能/审计脚本沉在 `scripts/`
- [ ] 频繁复用的脚本封成 MCP，用 `claude mcp add --scope user` 挂全局
- [ ] 长会话定期 `/context` → `/compact`
- [ ] 复杂重构前 `/effort high` 或 `xhigh`
- [ ] 重要 commit 前先 `/diff`，再让 Claude 写 Angular 规范 message
- [ ] PR 前跑 `/code-review` 和 `/security-review`
