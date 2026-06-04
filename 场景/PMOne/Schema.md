### Q1: 改造之前是什么样的
旧版的 schema 是一个巨大的扁平对象，数据定义和 UI 配置全部混在一起——type、default、label、ctrl、options 写在同一个字段里。更麻烦的是嵌套：用 `mix: true` + `items` 做分组，字段散落在各层级，还有 `oldFormeSchema`、`oldFormePropKey` 这类兼容 hack。新增一个组件需要理解这套嵌套规则，改错一层整个面板就渲染不出来。

改造后分成三层：DataSchema 只管数据结构（类型、默认值、校验），UISchema 只管渲染（控件类型、分组、布局顺序），共享 Key 常量保证两边编译时对齐。新增组件只需要写两个文件 + 注册一行代码。

### Q2: 属性面板是怎么根据 schema 渲染的
属性面板的渲染是纯声明式的——UISchema 的 layout 数组定义了渲染顺序和结构，StylePanel 组件遍历 layout，根据每个节点的 type 决定渲染方式：普通字段直接渲染 DynamicCtrl，row 类型横向排列多个控件，group 类型渲染成折叠面板。DynamicCtrl 通过 `resolveCtrl(field.ctrl)` 从控件注册表拿到对应的 Vue 组件，用 `<component :is>` 动态渲染。


### Q3: 旧版本为什么满足不了支持 row(一行多控件) grop(折叠面板 switch 控制显隐)功能？在技术角度上应该可以实现

#### 看旧版代码里的实现方式：
```
// 旧版的"折叠面板"——用一个特殊的 ctrl 类型

axisLineGroup: {

type: Object,

ctrl: 'expand-item', // 布局当成控件来处理

expandType: 'axis-line', // 还要指定子类型

mix: true, // 标记需要把 items 拍平到上层

items: { ... }

}

// 旧版的"一行多控件"——又是另一个特殊 ctrl

axisNameStyle: {

type: Object,

ctrl: 'font-mix-group', // 又一种布局控件

mix: true,

items: { ... }

}
```

**问题不是"能不能做"，而是布局和数据定义耦合在一起：**

1. **每种布局方式都要发明一个 ctrl 类型。** 折叠面板是 `expand-item`，一行多控件是 `font-mix-group`，轴线样式是 `axis-line-group`。渲染侧每种都要写一个 v-else-if 分支。想加一种新布局（比如 Tab 切换、条件显隐面板），就要新增 ctrl 类型 + 新增渲染分支。
2. **嵌套规则不统一。** 有的用 `items` 嵌套，有的用 `schema` 引用，有的用 `mix: true` 把子字段拍平到父级。开发者每次写新 schema 都要翻旧代码看"这种布局应该怎么嵌套"。
3. **字段的"数据身份"和"布局身份"混在一起。** `axisLineGroup` 这个 key 在 schema 里既是一个字段定义（type: Object），又是一个布局容器（ctrl: 'expand-item'）。它的 `default` 该写什么？它的值在 properties 里长什么样？实际上它不存储数据，只是为了渲染用——但 schema 系统不区分这两种角色。
4. **`mix: true` 是一个 hack。** 注册时 `index.js` 里有一段逻辑：检测到 `mix && items` 就把子字段拍平到 schema 顶层。这意味着 schema 的声明结构和实际运行时结构不一致——你看到的嵌套不是真正的嵌套。
#### 新版怎么解决的：
layout 是独立的有序数组，只描述"怎么摆"，不参与数据定义：

```
// 布局就是布局，和数据定义完全分开
layout: [
	{ key: 'show', ctrl: 'switch', label: '显示' }, // 普通字段
	{ type: 'row', fields: [{ key: 'width' }, { key: 'type' }] }, // 行布局
	{ type: 'group', switch: { key: 'axisLineShow' }, layout: [...] }, // 折叠面板
]
```

想加新布局类型，只需要在 StylePanel 里加一个 `v-else-if`（渲染侧），不需要动 DataSchema、不需要发明新 ctrl、不需要 mix hack。布局节点不占 schema 字段位，不产生数据。
####  面试时的说法： 
> "旧版技术上能实现 row 和 group，但它把布局当成一种特殊的控件类型来处理——每种布局都要发明一个 ctrl，渲染侧要加分支，还需要 mix: true 这种 hack 把嵌套字段拍平。新版把布局抽成独立的 layout 层，和数据定义彻底分开，加新布局只改渲染侧一处。"


### Q4: 数组模式（如多条坐标轴）是怎么处理的
DataSchema 里有两个字段控制数组模式：`isArray: true` 和 `arrayOptions`。arrayOptions 定义了有哪些数组维度，比如坐标轴就是 `[{ label: 'X轴', key: 'axisX' }, { label: 'Y轴', key: 'axisY' }]`。

SchemaPanel 检测到 `schema.isArray` 为 true 时，渲染一组 Tab 按钮（X轴/Y轴），点击切换 `activeTab`。当前 Tab 对应的数据是 `properties[activeTab]`，它是一个数组——每个元素是一条轴的配置。

每条轴渲染成一个 ExpandItem 折叠面板，标题取 `item.name?.val`，内部复用同一个 StylePanel + 同一份 layout。更新时通过索引定位：`onArrayFieldUpdate(idx, key, val)` 找到 `currentItems[idx][key].val` 直接赋值。

Resolver 那边也对应处理——`resolveProperties` 检测到 `schema.isArray` 时，遍历每个 arrayOption 的 key，对数组中的每个 item 逐个执行迁移链和字段补齐。

### Q6: Vue 的 component :is 动态组件有什么性能问题
在我们这个场景下，性能问题不大，但要知道潜在风险：

1. **组件切换开销**：`<component :is>` 每次切换会销毁旧组件、创建新组件。但我们的场景里 ctrl 类型是固定的（由 schema 决定），不会在运行时频繁切换，所以这个问题不存在。
2. **大量实例**：如果一个面板有 30+ 个字段，就会同时存在 30+ 个 DynamicCtrl 实例。每个实例都有自己的响应式系统和 computed。对于我们的属性面板规模（一个 schema 大概 20-30 个字段），Vue 处理起来没有压力。
3. **如果真的遇到性能瓶颈**，可以做的优化：
    - 折叠面板内的控件用 `v-if` 而不是 `v-show`，折叠时不渲染
    - 对不在视口内的 group 做懒渲染
    - 用 `shallowRef` 减少深层响应式追踪

实际上我们选 `<component :is>` 而不是 40 个 v-else-if，性能反而更好——v-else-if 链越长，Vue 的 patch 过程越慢，因为每次更新都要从头走条件判断。动态组件是 O(1) 查找。
### Q7: 这套 schema 系统和 JSON Schema 的区别是什么？优劣势？
**定位不同：**

- JSON Schema 是一个通用的数据校验规范，核心能力是描述"数据长什么样"——类型、约束、嵌套结构、$ref 引用。它不关心 UI。
- 我们的 schema 系统是一个**领域专用的配置到面板的映射层**，核心能力是描述"这个配置项怎么编辑"——用什么控件、怎么分组、怎么布局。

**为什么没用 JSON Schema：**

1. JSON Schema 没有 UI 语义。要渲染面板还是需要额外的 UI Schema 层（比如 react-jsonschema-form 的 uiSchema），等于多了一层抽象但没减少工作量。
2. 我们的 layout 需求很具体——折叠面板、行布局、switch 联动——这些在 JSON Schema 生态里没有标准化的表达方式，每个库的实现都不一样。
3. JSON Schema 的校验能力（pattern、minLength、enum）我们用不上太多，属性面板的校验主要是类型级别的，控件本身就约束了输入。

**劣势（诚实讲）：**

- 不通用，只能在这个项目内使用，没有社区生态
- 没有 $ref 和 definitions，schema 复用靠手动提取（比如 fontOptions 是手动共享的常量）
- 如果未来要做 schema 的序列化存储或跨系统传输，JSON Schema 有现成的工具链，我们的没有

**总结：** 这是一个"够用且轻量"的选择。如果项目规模再大一个量级（比如要支持用户自定义组件、跨团队共享 schema），可能需要考虑迁移到更标准化的方案。

### Q8: 迁移出错的兜底策略
先说设计层面的防御，再说运行时的兜底。

**设计层面——让迁移不容易出错：**

1. 迁移函数是幂等的。每一步都有前置条件判断——比如 v2→v3 的拍平，只有 `item.axisName?.val && typeof item.axisName.val === 'object'` 成立才执行。已经迁移过的数据再跑一次不会出错。
2. 新字段赋值用 `??`（nullish coalescing），已有值不会被覆盖。
3. 迁移链是顺序执行的，从 `item._meta.version` 开始逐步往上走，不会跳版本。

**运行时兜底：**

1. 迁移是在前端运行时做的，**不修改后端存储的原始数据**。也就是说，即使迁移逻辑有 bug，刷新页面会从后端重新拉原始数据再跑一遍迁移链。最坏情况是面板显示异常，不会造成数据丢失。
2. `resolveItem` 最后一步是补齐缺失字段——即使迁移函数漏处理了某个 key，只要 DataSchema 里定义了这个字段，就会用默认值补上。面板至少能渲染出来，只是值可能不是用户之前设置的。
3. 注册时的 key 对齐校验（UISchema 引用了 DataSchema 没有的 key → console.error）能在开发阶段提前发现问题。

**如果面试官追问"那用户之前的配置丢了怎么办"：**  
诚实说：如果迁移函数写错了导致某个旧字段没被正确映射到新 key，用户会看到默认值而不是自己之前设的值。这种情况靠测试覆盖——mock 里有 v1、v2、v3 三种格式的测试数据，迁移后和预期结果做断言。上线前必须跑通所有版本的迁移路径。


### Q9: 怎么兼容旧数据？生产环境已经有大屏在跑
#### 回答框架：
核心策略是**运行时迁移 + 不动后端存储**。

生产环境已有的大屏，后端数据库里存的还是旧格式（v1 或 v2）。我没有做批量刷数据的操作，原因是：

1. 大屏数量多，批量迁移有风险，一旦出错影响面大
2. 旧版前端还在线上跑，如果后端数据格式变了，旧版会崩
3. 不同大屏创建时间不同，数据版本不统一，没法一刀切

所以选择了**前端运行时按需迁移**：

1. 每条数据带 `_meta.version` 标记当前版本
2. 前端加载数据后，`resolveProperties` 检查版本号，从当前版本开始逐步执行迁移函数
3. 迁移完成后标记为最新版本，面板正常渲染
4. 用户修改配置后保存，写回后端的就是新格式

这样实现了**渐进式迁移**——用户打开旧大屏编辑并保存后，数据自然升级到新版本；没被打开过的大屏保持原样，不受影响。

#### 具体迁移链（代码实证）：
- 裸值包装成 `{ val: xxx }`。旧数据 `show: true` 变成 `show: { val: true }`。统一数据访问方式，面板组件只需要读 `.val`。
-  嵌套对象拍平为独立 key。旧数据 `axisName: { val: { show: true, color: '#ccc' } }` 拆成 `axisNameShow: { val: true }`、`axisNameFontColor: { val: '#ccc' }`。这一步是为了配合新的 schema 结构——DataSchema 里每个字段都是扁平的独立 key，不再有嵌套对象。
####  如果追问"迁移后保存，旧版前端还能用吗"：
 这是一个取舍。v1 格式的数据在旧版前端里，那些被拍平的字段（如 `axisNameShow`）旧版不认识，会被忽略；旧版认识的嵌套字段（如 `axisName`）已经被 delete 了，旧版会用默认值。所以一旦用新版编辑并保存，这条数据就不建议再用旧版打开了。实际操作中我们是灰度发布——新版上线后旧版入口逐步下掉，不会出现同一个大屏两个版本交替编辑的情况。