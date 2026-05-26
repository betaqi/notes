
> 场景：把 `result.scalars().all()` 改成 `result.all()`，请求 `/articles` 报 500。 真正的错误在 uvicorn 终端：
> 
> ```
> AttributeError: 'Row' object has no attribute 'id'
> ```

## 一、核心概念：execute() 返回的是二维表格

SQLAlchemy 的 `session.execute(select(...))` 返回的东西，**永远是一个"二维表格"的游标**，不管你 select 的是什么。

```
       列1        列2        列3 ...
行1  [ ... ]    [ ... ]    [ ... ]
行2  [ ... ]    [ ... ]    [ ... ]
```

即使你只 select 一列、甚至 select 一整个 ORM 对象，它也坚持用"表格"这个抽象包起来。

## 二、三个对象搞清楚

### 1. `Article`（你自己定义的 ORM 类）

```python
class Article(Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
```

一个 `Article` 实例 = 数据库里一行的"对象化表示"。

类比 Prisma：

```ts
const article: Article = { id: 1, title: "文章 1" }
article.id       // 1
article.title    // "文章 1"
```

Python 里一模一样：`a.id` / `a.title` 能用。

### 2. `Row`（SQLAlchemy 的"一行"容器）

`Row` 是 SQLAlchemy 内置的类，**像一个增强版的元组**，用来表达"查询结果的一行"。一行里**可能有多列**，每列是一个值。

类比 Node 的 `pg` 驱动：

```js
// node-postgres 返回:
rows = [
  { id: 1, title: '文章 1' },   // 一"行"是一个普通对象
  ...
]
```

SQLAlchemy 不用普通字典，它用自己的 `Row` 类型，但思路一样：**一行 = 一个多列的容器**。

### 3. `.scalars()` 的作用 —— 把"一行一列"的壳剥掉

假设你执行 `select(Article).limit(2)`，在底层 SQLAlchemy 其实是这么想的：

```
SELECT (整个 Article 对象作为 1 列) FROM articles LIMIT 2
```

返回的结果表格：

```
        列0 (类型: Article)
行1   Article(id=1, title="文章 1")
行2   Article(id=2, title="文章 2")
```

所以：

```python
result = await session.execute(select(Article).limit(2))

# 情况 A：不剥壳
result.all()
# → [Row(Article(id=1,...)), Row(Article(id=2,...))]
#   ^^^                      ^^^  外层是 Row！

# 情况 B：剥壳
result.scalars().all()
# → [Article(id=1,...), Article(id=2,...)]
#   已经把"行"的壳去掉，直接拿到里面那一列的值
```

`.scalars()` 的语义："**我知道每行只有一列有用，请把外面的 Row 壳去掉**"。名字来自数据库术语 **scalar = 单值**。

### 为啥 `a.id` 会炸

删掉 `.scalars()` 后，`items` 是这样的：

```python
items = [Row(Article(id=1,...)), Row(Article(id=2,...))]

for a in items:
    a.id         # ❌ Row 对象没有 .id 属性
                 #    真正的 id 藏在 a[0].id 里
    a.title      # ❌ 同理
```

`Row` 是个"容器"，**容器本身没有 `id`**，里面装的那个 `Article` 才有。就像：

```js
const box = [article]   // 数组包着 article
box.id                  // undefined，数组没有 id
box[0].id               // 1
```

## 三、为什么需要 Row —— select() 的各种形态

因为 `select()` 里可以塞任何东西，不一定是一整个 ORM 对象：

```python
# 场景 A：查整个对象
select(Article)
# 结果每行 1 列：Article 对象

# 场景 B：只查两个字段（不想查 body 这种大字段时常用）
select(Article.id, Article.title)
# 结果每行 2 列：(int, str)

# 场景 C：JOIN 两张表，两边都要
select(Article, User).join(User, Article.author_id == User.id)
# 结果每行 2 列：(Article 对象, User 对象)

# 场景 D：聚合 + 对象混着来
select(User, func.count(Article.id)).join(Article).group_by(User.id)
# 结果每行 2 列：(User 对象, int)

# 场景 E：纯聚合
select(func.count()).select_from(Article)
# 结果每行 1 列：int
```

如果 `execute()` 只能返回 `[Article, ...]`，那场景 B/C/D/E 就没法表达了。`Row` 就是 SQLAlchemy 给出的**统一答案**：不管你 select 了几列、都是什么类型，**每一行永远包成一个 `Row`**。

### Row 的"双面"访问

```python
# 场景 B：只查两个字段
result = await session.execute(select(Article.id, Article.title))
for row in result.all():
    row[0]            # id      → 按位置
    row[1]            # title
    row.id            # 按列名也行
    row.title
    id, title = row   # 解构，像元组

# 场景 C：JOIN
result = await session.execute(select(Article, User).join(User))
for row in result.all():
    row.Article       # Article 对象
    row.User          # User 对象
    article, user = row
```

JS 类比：

```ts
// SQLAlchemy 的 Row 类似这种联合抽象：
type Row<T extends readonly unknown[]> = T & { [K: string]: T[number] }
// 既能 row[0] 当元组用，又能 row.Article 当对象用
```

## 四、为什么不"智能"一点：单列时自动剥壳？

SQLAlchemy 的哲学：`**execute()` 行为必须稳定可预测**——无论你 select 几列，返回类型都一致（永远是 `Result[Row]`）。这样你写类型注解、写通用工具函数才好写。

它把"剥壳"做成**一个显式的动作**，就是 `.scalars()`。等于在说："我**知道**我这次只 select 了一列，请帮我省掉那层 Row 的壳。"

类比 JS 里的 `[].flat()`：数组方法不会"智能地"在只有一层嵌套时自动拍平，你必须显式告诉它。

Python 社区有句信仰：

> **Explicit is better than implicit.** —— The Zen of Python

随时可以在终端看到：

```bash
python3 -c "import this"
```

## 五、三种结果处理 API

|   |   |   |
|---|---|---|
|方法|返回类型|适用场景|
|`.all()`|`List[Row]`（类似具名元组）|`select(*columns)` 查部分字段，支持元组解包|
|`.scalars().all()`|`List[ORM对象]`（如 `Article`）|`select(Article)` 查整个模型，需要操作对象或访问关联|
|`.mappings().all()`|`List[RowMapping]`（类似字典）|两种情况都适用，序列化友好，推荐只读 API 场景|

### `.all()` —— 最原始的 SQL 层 API

支持元组解包：

```python
id, title = row  # 元组解包，.mappings() 不支持
```

### `.scalars().all()` —— 专为 ORM 单对象查询设计

返回"活的"ORM 对象：

```python
article = result.scalars().first()
article.comments   # ✅ 可访问关联关系（自动查 Comment 表）
article.title = "新标题"
await session.commit()  # ✅ 自动生成 UPDATE 语句
```

### `.mappings().all()` —— SQLAlchemy 2.0 新增

统一两种查询方式，序列化友好：

```python
result = await session.execute(select(Article))
items = result.mappings().all()
items[0]["id"]    # ✅ 字典式访问
items[0]["title"] # ✅
```

### "活的 ORM 对象" vs "死的字典"

|   |   |   |
|---|---|---|
||ORM 对象（scalars）|字典（mappings）|
|Session 追踪|✅ 被追踪，能感知变化|❌ Session 不认识它|
|修改自动同步 DB|✅ 改字段 → commit → 自动 UPDATE|❌ 只改内存，数据库无变化|
|访问关联关系|✅ `article.comments` 自动查关联表|❌ key 根本不存在|
|序列化方便|❌ 需要 Pydantic 或 `__dict__`|✅ 直接用，JSON 友好|

## 六、常见坑：await 与 .all() 的顺序

```python
# ❌ 错误：.all() 作用在 coroutine 上，coroutine 没有 .all()
await session.execute(...).all()

# ✅ 正确：await 先拿到 CursorResult，再调 .all()
result = await session.execute(...)
result.all()

# ✅ 也可以加括号强制 await 先执行（可读性差，不推荐）
(await session.execute(...)).all()
```

**原因**：Python 先执行 `session.execute(...).all()`（coroutine 上调 `.all()`），再 `await`，但 coroutine 没有 `.all()` 方法，报错。

## 七、便利快捷方式

因为"select 一个 ORM 对象 + `.scalars()`"是 80% 的场景，SQLAlchemy 2.0 提供了一个快捷入口：

```python
# 这两行等价
articles = (await session.execute(select(Article))).scalars().all()
articles = (await session.scalars(select(Article))).all()   # ← 更短
```

后者本质是"execute + scalars 合一"的语法糖。多列查询时还是得老老实实用 `execute()` + `Row`。

## 八、什么时候用哪种 + 最佳实践

### 选择指南

- **只读 API 查询、返回 JSON 响应** → `.mappings().all()`
    
- **需要修改数据、写回数据库** → `.scalars().all()`
    
- **需要访问关联关系**（如 `article.author`）→ `.scalars().all()`
    
- **需要元组解包** → `.all()`
    

### `dict(row._mapping)` 还需要吗？

用了 `.mappings().all()` 之后**不需要**，`RowMapping` 本身已经支持字典式访问。

```python
# ❌ 多此一举
result = [dict(row._mapping) for row in row_data.mappings().all()]

# ✅ 直接用
result = row_data.mappings().all()
result[0]["id"]  # 直接访问
```

> ⚠️ 注意：`RowMapping` 不是真正的 `dict`，若需要真正序列化（如传给某些库），可以 `[dict(row) for row in result]`

### 最佳实践代码

```python
async with SessionLocal() as session:
    if fields:
        field_list = fields.split(",")
        entities = [getattr(Article, field) for field in field_list]
        query = select(*entities)
    else:
        query = select(Article)

    row_data = await session.execute(
        query.offset((page - 1) * size).limit(size)
    )
    result = row_data.mappings().all()  # 两种情况统一处理

    return {
        "page": page,
        "size": size,
        "items": result,
    }
```

## 九、心智模型速查表

|   |   |   |
|---|---|---|
|你 select 的形状|每行长什么样|怎么拿到里面的值|
|`select(Article)`|`Row(Article(...))`|`.scalars()` 剥壳 → `Article(...)`|
|`select(Article.id, Article.title)`|`Row(id=1, title="...")`|`row.id` / `row[0]` / 元组解包|
|`select(Article, User)`|`Row(Article=..., User=...)`|`row.Article` / `row.User`|
|`select(func.count())`|`Row(1,)`|`.scalar()`（单数！）→ `1`|
|以上任意形式|`RowMapping`|`.mappings()` → `row["id"]` 字典式访问|

## 十、练习总结表

|   |   |   |   |
|---|---|---|---|
|练习|考点|报错类型|报错在哪一刻|
|1|`async/await` 语法规则|`SyntaxError`|启动时（解析源码）|
|2|`lifespan` 生命周期|`OperationalError: no such table`|第一次请求时|
|3|`.scalars()` 的解包语义|`AttributeError: 'Row' has no attribute 'id'`|第一次请求时|

---

**参考文档**

- [SQLAlchemy 2.0 ORM 查询指南](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html)
    
- [Result.mappings() API](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Result.mappings)
    

> ⚠️ 注意：SQLAlchemy 1.x 和 2.0 差异较大，查文档时注意版本。