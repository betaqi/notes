### Q1: SSE 和 WebSocket 的区别
**通信方向：** SSE 是单向的，只能服务端向客户端推送；WebSocket 是全双工的，客户端和服务端可以同时互发消息。
**协议层面：** SSE 基于 HTTP/1.1（普通的长连接 + `Content-Type: text/event-stream`），天然兼容代理、CDN 和防火墙。WebSocket 通过 HTTP Upgrade 握手后切换到 `ws://` / `wss://` 独立协议，有些企业代理或网关对它支持不好。
**数据格式：** SSE 只能传文本（UTF-8），如果要传二进制得 Base64 编码。WebSocket 原生支持文本和二进制帧（ArrayBuffer / Blob）。
**典型场景：**
- SSE 适合"服务端单向推流"——股票行情、新闻 feed、ChatGPT 流式输出
- WebSocket 适合"双向实时交互"——在线聊天、协同编辑、多人游戏、实时交易

#### Q2：ChatGPT 流式输出的"停止生成"实现
1. **前端**：`AbortController` + `signal` 传给 fetch，调用 `.abort()` 中断请求
2. **已渲染内容保留**：abort 只是停止读流，DOM 上已经 append 的文字不会消失
3. **后端配合**：监听连接关闭事件，及时终止模型推理，节省资源
4. **用户体验细节**：停止后把按钮切回"发送"状态，可能还要在消息末尾加个"已停止生成"的标记、、

### Q3：Fetch API vs XHR 处理 AI 流式输出
AI 聊天的流式输出通常使用 SSE（Server-Sent Events）或者直接返回一个 chunked 的 HTTP 响应体，服务端逐步发送 token。这个场景下 Fetch API 和 XHR 的差异非常明显。
**Fetch API 的优势在于它原生支持 ReadableStream。** 通过 `response.body.getReader()` 可以拿到一个流式 reader，配合 `TextDecoder` 逐块读取服务器推送过来的内容：

### Q4: TypeScript 中 interface 与 type 的区别
**声明合并（Declaration Merging）**：interface 支持，type 不支持。同名 interface 会自动合并属性，这在扩展第三方库的类型定义时非常有用：
**表达能力**：type 更强。type 可以表示联合类型、交叉类型、条件类型、映射类型、元组等，interface 做不到：
**继承方式**：interface 用 `extends`，type 用交叉类型 `&`。二者也可以互相继承——interface 可以 extends 一个 type，type 也可以 `&` 一个 interface。
**实际选择原则**：定义对象结构（组件 Props、API 响应）优先用 interface；需要联合类型、条件类型、映射类型等高级类型运算时用 type；基本类型别名用 type。

### Q5: 实现 RAG 前端流程涉及的技术点
用户上传文档 → 前端分块 → 存入内存/IndexedDB ↓ 用户提问 → 前端检索相关 chunks → 拼装 prompt → 调用 LLM API → 流式输出

**分块的关键考虑：**
- chunk 太大，注入 token 数多、相关性被稀释；太小，语义不完整
- overlap 让边界处的信息不会丢失
- 按段落/标题等语义边界切割优于硬切字符数
- 对于结构化文档（Markdown 有标题层级），可以按 heading 切分，保留层级路径作为 chunk 的前缀


