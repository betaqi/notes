
## 什么是 KV Cache

**KV = Key-Value**，是 Transformer 模型内部的一个计算结构。

模型每次读一段文字，内部会把每个 token 转换成两个向量：

```
token "北京"  →  Key 向量 + Value 向量
token "天气"  →  Key 向量 + Value 向量
token "今天"  →  Key 向量 + Value 向量
```

这些 Key/Value 向量计算很耗时，**但如果这段文字没变，结果每次都一样**。Cache 的作用就是把算过的结果存起来，下次直接复用。

---

## Cache 的作用

```
第 1 次请求：
system prompt（500 token）→ 计算 KV，存起来
用户消息（10 token）      → 计算 KV
                              耗时：510 token

第 2 次请求：
system prompt（500 token）→ 命中缓存，直接用！✅
用户消息（10 token）      → 计算 KV
                              耗时：只有 10 token
```

类比理解：就像做数学题，把算过的中间结果存起来，下次直接用，不用从头再算。

---

## 实际影响

||没命中缓存|命中缓存|
|---|---|---|
|计算量|全部重新算|只算新增部分|
|速度|慢|快|
|费用|正常价|更低或免费|

---

## 怎么让命中率更高

```
// ✅ 好的顺序：不变的放前面
[system prompt]    ← 每次一样，容易命中
[历史对话]         ← 前面的不变，容易命中
[最新用户消息]     ← 每次不同，放最后

// ❌ 坏的习惯：每次改 system prompt
每次请求都修改 system prompt → 缓存永远失效
```

---

## 在 usage 里怎么看

json

```json
"prompt_tokens_details": {
  "cached_tokens": 800   // 命中缓存的 token 数，费用更低或免费
}
```

`cached_tokens` 越多，说明缓存命中率越高，速度越快、费用越低。
