### Q1：这个项目是干什么的？

PMOne 是一个企业级可以对 Server/Web/Mobile 三端进行监控，请求链路追踪、拓扑分析、告警配置这些能力我在里面主要做两块事情：一是负责大屏编辑器的 Schema 化重构，把原来硬编码的组件配置改成声明式的 schema 驱动；二是参与画布交互能力建设，处理多选、编组、缩放这些编辑器核心交互。项目用 Vue 3 + Vite 构建，可视化用 ECharts 和 AntV G6。

### Q2：你负责什么？
我主要负责大屏编辑器的 [[Schema]] 化重构，从方案设计到落地都是我做的。另外深度参与了[[画布交互能力]]建设和[[Webpack 到 Vite]]的构建迁移。

### Q3：最难/最有挑战的地方是什么？
建议主讲 [[Schema]] 化重构
最难的是设计 schema 的抽象层次。太细了——每个组件的 schema 写起来比原来硬编码还麻烦；太粗了——覆盖不了组件之间的差异。当时我们有 30 多种图表组件，每种的配置项差异很大，有的需要嵌套 schema，有的需要数组类型的 schema。最终我选择了支持 schema 和 schema[] 两种嵌套类型，加上类型校验和路径寻址更新，让属性面板能递归渲染任意深度的配置结构。

### Q4：如果重做会怎么优化？

有两个方向。第一，[[Schema]] 系统现在是运行时注册的，用 import.meta.globEager 批量加载，如果组件数量继续增长，首屏加载会有压力，可以改成按需加载。第二，属性面板的渲染现在是完全由 schema 驱动的，但缺少可视化的 schema 编辑工具，新同事上手成本还是偏高，如果有时间可以做一个 schema playground。

### Q5： 大屏编辑器数据流
#### 完整数据流（面试时用口述，不用画图）
```
启动阶段：
initSchemas()
→ import.meta.globEager 批量加载所有 schema 文件

→ 每个 schema 调用 registerSchema() 注册到 dashboardSchemas 字典

→ 校验：name 必须存在、每个字段必须有 type、type 只能是 schema/schema[]/构造函数

用户拖拽组件到画布：

getDefaultProperties(schemaName)

→ 读取该组件的 schema 定义

→ 遍历所有字段，按 type 生成默认值

→ 遇到嵌套 schema 类型 → 递归调用自身

→ 遇到 schema[] 类型 → 初始化为空数组

→ 返回 { properties, variables }

→ properties 挂载到组件实例上，带 _meta 路径信息

属性面板渲染：

getSchemaRecursion(schemaName, viewStyle)

→ 递归展开 schema 树

→ 根据 viewStyle 过滤不需要展示的字段

→ 属性面板根据字段的 type 渲染对应控件（Number → 数字输入框，String → 文本框，schema → 嵌套面板）

用户编辑属性：

schema.updateProperty(path, value)

→ 通过 _meta.path 定位到具体字段

→ 类型校验 + options 校验 + 自定义 validator

→ 校验通过 → 直接赋值（Vue 3 响应式自动触发视图更新）

数据源绑定：

组件 schema 中的数据源字段 → 配置 API 地址和参数

→ 运行时调用 Service 层 → helper.js fetch → /api/v1|v2/

→ 返回数据填充到组件 properties → 图表重新渲染
```
####   精简讲法
整个编辑器的数据流分四个阶段。

**第一步是 schema 注册。** 应用启动时通过 Vite 的 globEager 批量加载所有 schema 文件，每个组件类型对应一份 schema，注册到一个全局字典里。注册时会做校验——每个字段必须声明类型，支持基础类型和嵌套 schema 两种。

**第二步是组件实例化。** 用户拖一个组件到画布上时，调用 getDefaultProperties，根据 schema 定义递归生成默认属性值，每个属性节点带一个 _meta 记录它在属性树中的路径，后续更新靠这个路径寻址。

**第三步是属性面板渲染。** 面板拿到组件的 schema 后递归展开，每个字段根据 type 渲染对应的编辑控件。如果是嵌套 schema 就渲染一个折叠面板，里面再递归。

**第四步是数据源绑定。** 组件可以配置数据源，运行时从后端 API 拉数据，填充到 properties 里，ECharts 或其他渲染器响应式更新。



