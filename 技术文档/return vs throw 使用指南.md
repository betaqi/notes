

## 核心原则

> **`return`** — 预期内的情况，我处理完了，继续或退出  
> **`throw`** — 异常情况，我处理不了，交给调用方决定

---

## `return` 适用场景

### 1. 输入校验 / 边界条件

```ts
function processUser(user: User | null) {
  if (!user) return  // 没有 user 是正常情况，直接退出
  sendEmail(user.email)
}
```

### 2. Guard Clause（提前退出，避免嵌套）

```ts
// ❌ 嵌套地狱
function save(data) {
  if (data) {
    if (data.name) {
      if (data.name.length > 0) {
        db.save(data)
      }
    }
  }
}

// ✅ 提前 return
function save(data) {
  if (!data) return
  if (!data.name) return
  if (data.name.length === 0) return
  db.save(data)
}
```

### 3. 事件回调（你不是调用方）

```ts
onmessage(event) {
  let payload
  try {
    payload = JSON.parse(event.data)
  } catch {
    console.error('Parse failed')
    return  // 跳过这帧，等下一条，不影响后续触发
  }
  handlers.onMessage(type, payload)
}
```

---

## `throw` 适用场景

### 1. 调用方需要知道失败了

```ts
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`)
  if (!res.ok) throw new Error(`Failed: ${res.status}`)
  return res.json()
}

// 调用方自己决定如何处理
try {
  const user = await fetchUser('123')
} catch (e) {
  showToast('加载失败，请重试')
}
```

### 2. 不可能发生的状态（防御性编程）

```ts
function getLabel(status: 'active' | 'inactive' | 'banned') {
  switch (status) {
    case 'active':   return '正常'
    case 'inactive': return '未激活'
    case 'banned':   return '已封禁'
    default:
      throw new Error(`Unexpected status: ${status}`)  // 走到这里说明有 bug
  }
}
```

### 3. 构造函数 / 初始化失败

```ts
class ApiClient {
  constructor(apiKey: string) {
    if (!apiKey) throw new Error('apiKey is required')
    this.apiKey = apiKey
  }
}
```

### 4. 自定义错误类型，让上层精准处理

```ts
class NotFoundError extends Error {}
class PermissionError extends Error {}

async function getPost(id: string, user: User) {
  const post = await db.find(id)
  if (!post) throw new NotFoundError('Post not found')
  if (post.authorId !== user.id) throw new PermissionError()
  return post
}

// 调用方区分处理
try {
  const post = await getPost(id, currentUser)
} catch (e) {
  if (e instanceof NotFoundError) redirect('/404')
  else if (e instanceof PermissionError) redirect('/403')
  else throw e  // 其他错误继续往上抛
}
```

---

## 进阶：`assertRes` 模式

统一处理业务接口错误，避免每个调用方都写 catch + toast：

```ts
class HandledError extends Error {}

function assertRes(res: Result<unknown>) {
  if (!res?.success && !res?.data) {
    ElMessage.error(res?.msg ?? '请检查网络状态')
    throw new HandledError(res?.msg ?? '请求失败')
  }
}

// 调用方干净清晰
async function loadUser() {
  const res = await fetchUser('123')
  assertRes(res)        // 失败自动 toast + 中断
  renderUser(res.data)  // 只关心成功逻辑
}

// 全局兜底，避免已处理的错误重复提示
app.config.errorHandler = (err) => {
  if (err instanceof HandledError) return  // 已 toast，忽略
  ElMessage.error('未知错误')
}
```

---

## 事件回调中 `return` vs `throw` 对比

||`return`|`throw`|
|---|---|---|
|后续帧继续触发|✅|✅|
|当前帧 handler 执行|❌ 跳过|❌ 跳过|
|控制台表现|只有主动 `console.error`|红色报错 + 调用栈|
|全局 error 监听器|❌ 捕获不到|✅ 能捕获|
|**推荐**|✅ 事件回调首选|需上报时手动处理|

---

## 一句话判断

```
这个失败，调用方需要知道吗？
├── 不需要（边界/跳过/正常分支） → return
└── 需要（失败/异常/bug）        → throw
```