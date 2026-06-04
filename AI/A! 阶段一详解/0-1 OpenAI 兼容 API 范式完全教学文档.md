
> 定位：面向 AI 工程实践的核心基础知识。读完本文，你将彻底理解为什么”换模型不换代码”成为可能，以及这套协议的每一个细节。

-----

## 一、为什么需要”统一协议”？

在 AI 大爆发之前，每个模型服务商都有自己的接口格式，开发者每换一个模型就要重写一套对接代码，成本极高。

OpenAI 在发布 GPT 系列 API 时，设计了一套清晰、完整的 HTTP RESTful 接口规范。由于 GPT 系列模型极度流行，这套规范事实上成为了行业标准——即 **OpenAI 兼容 API 范式**。

今天，无论是：

- 闭源云服务：Anthropic Claude、Google Gemini、Mistral、DeepSeek
- 开源推理引擎：vLLM、SGLang、Ollama、LM Studio
- 应用框架：OpenAI SDK、LangChain、LlamaIndex、Cherry Studio

它们全都对外暴露同一套接口。这就是”换模型不换代码”的底层基础。

-----

## 二、三大核心端点

### 2.1 `/v1/chat/completions` — 对话补全（最重要）

这是整个生态的核心端点，所有聊天类、Agent 类应用都走这里。

**请求结构：**

```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system",    "content": "你是一个专业的代码助手。" },
    { "role": "user",      "content": "帮我写一个快速排序。" }
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": false
}
```

**响应结构：**

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1718000000,
  "model": "gpt-4o-2024-05-13",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "```python\ndef quick_sort(arr): ..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 28,
    "completion_tokens": 156,
    "total_tokens": 184
  }
}
```

**关键字段解释：**

| 字段                           | 含义                                                    |
| ---------------------------- | ----------------------------------------------------- |
| `choices[0].message.content` | 模型实际回复的文本内容                                           |
| `finish_reason`              | 停止原因：`stop`（正常结束）/ `length`（超长截断）/ `tool_calls`（调用工具） |
| `id`                         | 本次请求的唯一标识，用于追踪和计费                                     |


-----

### 2.2 `/v1/completions` — 文本补全（Legacy）

这是更早期的端点，不使用 messages 数组，直接传入一段 `prompt` 文本让模型续写。

```json
{
  "model": "gpt-3.5-turbo-instruct",
  "prompt": "快速排序的核心思想是",
  "max_tokens": 200
}
```

**现状：** 新模型几乎不支持此端点，了解即可。生产中请始终使用 `/v1/chat/completions`。

-----

### 2.3 `/v1/embeddings` — 向量嵌入

将文本转换为高维向量，用于语义检索、RAG（检索增强生成）、相似度计算等场景。

**请求：**

```json
{
  "model": "text-embedding-3-small",
  "input": "OpenAI 兼容 API 是什么？",
  "encoding_format": "float"
}
```

**响应：**

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [0.0023, -0.0091, 0.0412, ...]  // 维度通常为 1536 或 3072
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "total_tokens": 10
  }
}
```

> **用途：** 把文档库全部转成向量存入向量数据库（如 Chroma、Milvus），用户提问时也转成向量，做相似度检索，找到最相关的文档片段喂给 LLM，这就是 RAG 的核心流程。

-----

## 三、messages 数组详解

messages 是对话历史的完整快照。每个元素是一条消息，拥有 `role` 和 `content` 两个核心字段。

### 3.1 四种 role

#### `system` — 系统提示词

在对话最开始设定模型的”人格”和”行为规则”。对模型影响力最强。

```json
{
  "role": "system",
  "content": "你是一名严谨的法律顾问，回答必须引用具体法条，不得猜测。"
}
```

#### `user` — 用户消息

真实用户发送的内容，也可以是程序自动构造的输入。

```json
{
  "role": "user",
  "content": "劳动合同到期后，公司不续签需要赔偿吗？"
}
```

#### `assistant` — 模型回复

把模型之前的回复也放入 messages，构成多轮对话上下文。**模型本身无记忆，多轮对话全靠客户端把历史消息拼接传入。**

```json
{
  "role": "assistant",
  "content": "根据《劳动合同法》第46条，劳动合同期满，用人单位不续签的，应支付经济补偿金..."
}
```

#### `tool` — 工具调用结果

当模型发出 `tool_calls` 请求后，客户端执行 `Function Tool（函数工具）` 完工具，把结果以 `"role": "tool"`  回传给模型。（详见第五章）


```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "{\"weather\": \"晴\", \"temperature\": 28}"
}
```

### 3.2 多轮对话的完整示例

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system",    "content": "你是一个旅游规划助手。" },
    { "role": "user",      "content": "我想去日本旅游，有什么建议？" },
    { "role": "assistant", "content": "日本旅游推荐东京、京都、大阪三城路线..." },
    { "role": "user",      "content": "京都有哪些必去的寺庙？" }
  ]
}
```

> **关键理解：** API 是无状态的。每次请求你都要把完整对话历史发过去。这是 token 消耗快速增长的根本原因，也是为什么需要”上下文窗口管理”。

-----
## 五、工具调用（Function Calling）

这是 LLM 从”聊天机器人”进化为”Agent”的关键能力。

### 5.1 核心思想

模型本身无法执行代码、查数据库、调接口。但你可以告诉模型”有哪些工具可用”，当模型判断需要某个工具时，它不会直接回复文字，而是输出一段结构化的”工具调用指令”，由你的代码去真正执行，再把结果回传给模型。

### 5.2 三个关键字段

#### 1. `tools` — 定义可用工具列表

随每次请求发送，告诉模型"有哪些工具可以用"。

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取指定城市的当前天气",  // ⭐ 模型靠这个判断要不要调用它
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "城市名称，如：北京、上海"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"]
            }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

---

#### 2. `tool_choice` — 控制模型是否必须调用工具

| 值 | 含义 |
|---|---|
| `"auto"` | 模型自己决定是否调用（默认） |
| `"none"` | 禁止调用任何工具 |
| `"required"` | 强制必须调用至少一个工具 |
| `{"type":"function","function":{"name":"get_weather"}}` | 强制调用特定工具 |

---

#### 3. `tool_calls` — 模型返回的调用指令

当模型决定调用工具时，响应长这样：

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"北京\", \"unit\": \"celsius\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

两个注意点：

- `content: null` — 模型没有文字输出，具体原因要看 finish_reason - `finish_reason = "tool_calls"` → 在等工具结果，用户看不到任何内容 - `finish_reason = "content_filter"` → 内容被拦截，用户看不到任何内容
- `arguments` 是**字符串**，不是对象，使用前需要 `JSON.parse()` / `json.loads()` 解析

---

#### 完整调用循环

```
前端：发送用户消息
       ↓
后端：带上 tools 定义，调用大模型 API
       ↓
大模型：返回 tool_calls，content = null，finish_reason = "tool_calls"
       ↓
后端：解析 tool_calls，执行对应函数       ← 用户感知不到
       ↓
后端：把结果塞回，再次调用大模型 API
       ↓
大模型：返回最终 content，finish_reason = "stop"
       ↓
前端：拿到结果，展示给用户               ← 用户才感知到
```


##### `finish_reason` 是判断流程是否结束的关键信号：

| 值                | 含义       | 常见场景                 |
| ---------------- | -------- | -------------------- |
| `stop`           | 模型正常输出完毕 | 正常回答完成               |
| `tool_calls`     | 模型要调用工具  | Function Calling 流程中 |
| `length`         | 输出被截断    | 达到 `max_tokens` 限制   |
| `content_filter` | 被安全过滤器拦截 | 触发内容审核               |
| `null`           | 还在流式输出中  | 用 stream 模式时的中间状态    |

---

### 代码示例（完整循环）

```python
import json
from openai import OpenAI

client = OpenAI(api_key="YOUR_KEY")

def get_weather(city: str, unit: str = "celsius") -> dict:
    # 模拟真实天气 API 调用
    return {"city": city, "temperature": 28, "condition": "晴", "unit": unit}

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "获取城市天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
        }
    }
}]

messages = [{"role": "user", "content": "北京今天天气怎么样？"}]

# 第 1 轮：发送请求
response = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=tools
)
msg = response.choices[0].message

# 检测是否需要调用工具
if response.choices[0].finish_reason == "tool_calls":
    messages.append(msg)  # 把 assistant 的 tool_calls 加入历史

    for tool_call in msg.tool_calls:
        args = json.loads(tool_call.function.arguments)
        result = get_weather(**args)

        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,        # 对应 tool_calls 里的 id
            "content": json.dumps(result, ensure_ascii=False)
        })

    # 第 2 轮：带工具结果再次请求
    final = client.chat.completions.create(
        model="gpt-4o", messages=messages, tools=tools
    )
    print(final.choices[0].message.content)
    # 输出：北京今天28°C，晴天 ☀️
```
-----

## 六、结构化输出（response_format）

在自动化流水线中，你往往需要模型输出严格的 JSON 而不是随意的文字。

### 6.1 JSON Mode

最简单的方式，告诉模型”必须输出 JSON”：

```json
{
  "model": "gpt-4o",
  "response_format": { "type": "json_object" },
  "messages": [
    {
      "role": "user",
      "content": "提取以下文本中的人名和地点，以 JSON 格式返回：\n张三昨天在北京签了合同。"
    }
  ]
}
```

模型输出：

```json
{"people": ["张三"], "locations": ["北京"]}
```

> ⚠️ **注意：** 使用 JSON mode 时，system 或 user 消息中必须明确要求输出 JSON，否则模型可能陷入无限生成循环。

### 6.2 JSON Schema 严格模式（Structured Outputs）

更强大的方式，后端提供精确的 Schema，模型输出**严格符合**该结构：

```json
{
  "model": "gpt-4o",
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "person_extraction",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "people": {
            "type": "array",
            "items": { "type": "string" }
          },
          "locations": {
            "type": "array",
            "items": { "type": "string" }
          }
        },
        "required": ["people", "locations"],
        "additionalProperties": false
      }
    }
  },
  "messages": [...]
}
```

**两种模式对比：**

|特性       |JSON Mode|JSON Schema Strict|
|---------|---------|------------------|
|保证输出 JSON|✅        |✅                 |
|保证字段完整   |❌（可能缺字段） |✅（严格校验）           |
|支持嵌套对象   |✅        |✅（有限制）            |
|适用场景     |简单结构     |生产级数据提取           |

-----

## 七、usage 字段详解

每次 API 响应都携带 `usage`，记录本次请求的 token 消耗情况。这是成本管控和性能优化的核心数据。

### 7.1 基础字段

```json
{
  "usage": {
    "prompt_tokens": 128,
    "completion_tokens": 256,
    "total_tokens": 384
  }
}
```

| 字段                  | 含义                                 |
| ------------------- | ---------------------------------- |
| `prompt_tokens`     | 输入消耗的 token 数（messages + tools 定义） |
| `completion_tokens` | 模型生成回复消耗的 token 数                  |
| `total_tokens`      | 两者之和，计费依据                          |

### 7.2 扩展字段

高版本 API 中，`usage` 变得更丰富：

```json
{
  "usage": {
    "prompt_tokens": 1024,
    "completion_tokens": 512,
    "total_tokens": 1536,
    "prompt_tokens_details": {
      "cached_tokens": 800,       // 命中  KV Cache 的 token（费用更低或免费）
      "audio_tokens": 0
    },
    "completion_tokens_details": {
      "reasoning_tokens": 200,    // 模型内部"思考"消耗的 token（o1/o3 系列）
      "audio_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
  }
}
```

**重点理解：**

**`cached_tokens`（[[KV Cache ]] 命中）**

- 当你的 system prompt 或历史消息与上次请求高度重合时，服务端可以复用之前计算好的 KV Cache
- 命中缓存的 token 通常有折扣（OpenAI 为 50% 折扣）甚至免费
- 启示：**把不变的 system prompt 放在最前面，把变化的内容放在后面**，可以最大化缓存命中率

**`reasoning_tokens`（推理 token）**

- o1、o3 等推理模型在生成最终回复之前，会先进行内部”思维链”推理
- 这部分思考消耗的 token 计入 `reasoning_tokens`
- 你看不到这部分思考内容（除非模型支持 `reasoning_effort` 参数），但它计入计费

-----

## 八、统一协议的生态全景

### 8.1 为什么”换模型不换代码”成为可能

```
你的业务代码
      ↓
OpenAI Python SDK / LangChain / LlamaIndex
      ↓（统一发送 POST /v1/chat/completions）
┌─────────────────────────────────────────────┐
│          兼容同一套 API 的服务               │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐   │
│  │ OpenAI  │  │ Claude  │  │  Gemini  │   │
│  │ GPT-4o  │  │(OpenAI  │  │(OpenAI   │   │
│  │         │  │ 兼容层) │  │ 兼容层)  │   │
│  └─────────┘  └─────────┘  └──────────┘   │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐   │
│  │  vLLM   │  │ Ollama  │  │  SGLang  │   │
│  │(本地部署)│  │(本地部署)│  │(本地部署) │   │
│  └─────────┘  └─────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
```

切换模型，只需修改 `base_url` 和 `model` 名称：

```python
# 使用 OpenAI
client = OpenAI(api_key="sk-xxx")

# 切换到本地 Ollama（零代码改动，只改初始化）
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 随便填
)

# 切换到 vLLM（同理）
client = OpenAI(
    base_url="http://your-server:8000/v1",
    api_key="your-vllm-key"
)

# 业务代码完全不变
response = client.chat.completions.create(
    model="llama3.1:70b",  # 只改这里
    messages=[{"role": "user", "content": "你好"}]
)
```

### 8.2 各框架/工具与协议的关系

|工具/框架                |与协议的关系                                               |
|---------------------|-----------------------------------------------------|
|**OpenAI Python SDK**|官方封装，直接映射到 API 字段                                    |
|**LangChain**        |抽象层，底层调用 `/v1/chat/completions`，支持所有兼容服务             |
|**LlamaIndex**       |专注 RAG，同时调用 `/v1/chat/completions` 和 `/v1/embeddings`|
|**Cherry Studio**    |GUI 客户端，允许用户配置任意兼容 API 的 base_url                    |
|**vLLM**             |开源推理引擎，对外暴露完整的 OpenAI 兼容接口                           |
|**Ollama**           |本地模型管理，默认在 11434 端口提供兼容接口                            |

### 8.3 Anthropic Claude 的兼容策略

Claude 有自己的原生 API（`/v1/messages`），字段略有不同。但 Anthropic 也提供了 OpenAI 兼容层，让你无需修改代码即可接入。

同时，很多平台（如 AWS Bedrock、Azure AI Foundry）会在托管 Claude 时统一对外提供 OpenAI 兼容接口，进一步消除差异。

-----

## 九、常见参数速查

|参数                 |类型           |作用                       |
|-------------------|-------------|-------------------------|
|`temperature`      |float 0-2    |控制随机性。0=确定性输出，1=正常，>1=更随机|
|`top_p`            |float 0-1    |核采样。与 temperature 二选一使用  |
|`max_tokens`       |int          |限制生成的最大 token 数          |
|`n`                |int          |同时生成 n 个候选回复             |
|`stop`             |string/array |遇到指定字符串立即停止生成            |
|`presence_penalty` |float -2 to 2|正值惩罚已出现的 token，减少重复      |
|`frequency_penalty`|float -2 to 2|正值降低高频 token 的概率，增加多样性   |
|`seed`             |int          |指定随机种子，相同 seed 输出更稳定（可复现）|
|`logprobs`         |bool         |返回 token 的对数概率（用于评估模型置信度）|

-----

## 十、关键概念总结

```
┌─────────────────────────────────────────────────────┐
│                  你需要掌握的核心认知                  │
│                                                     │
│  1. API 无状态 → 多轮对话靠客户端维护完整 messages 历史  │
│                                                     │
│  2. stream=true → SSE 协议，delta 字段携带增量内容     │
│                                                     │
│  3. 工具调用 = 模型输出指令 + 你执行 + 你回传结果        │
│     finish_reason="tool_calls" 是触发信号             │
│                                                     │
│  4. cached_tokens 是降低成本的关键                    │
│     → system prompt 固定化 + 放在最前面               │
│                                                     │
│  5. reasoning_tokens 只有推理模型才有                  │
│     → 内部思考链，计费但不返回给你                     │
│                                                     │
│  6. base_url 可替换 → 这是整个生态统一性的基础           │
└─────────────────────────────────────────────────────┘
```

-----

## 附：快速上手模板

```python
from openai import OpenAI
import json

# 1. 初始化客户端（换 base_url 即可切换模型服务商）
client = OpenAI(
    api_key="YOUR_API_KEY",
    # base_url="http://localhost:11434/v1"  # 切换到 Ollama
)

# 2. 维护对话历史
messages = [
    {"role": "system", "content": "你是一个专业助手，回答简洁准确。"}
]

def chat(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        temperature=0.7,
        max_tokens=1024,
        stream=False
    )
    
    reply = response.choices[0].message.content
    messages.append({"role": "assistant", "content": reply})
    
    # 打印 token 消耗
    usage = response.usage
    print(f"[Token消耗] 输入:{usage.prompt_tokens} 输出:{usage.completion_tokens} 合计:{usage.total_tokens}")
    
    return reply

# 3. 开始对话
print(chat("什么是向量数据库？"))
print(chat("它和传统数据库有什么区别？"))  # 多轮对话自动携带上下文
```

-----

*本文档覆盖 OpenAI 兼容 API 范式的全部核心知识点。建议结合官方文档 [platform.openai.com/docs](https://platform.openai.com/docs) 进行实践验证。*