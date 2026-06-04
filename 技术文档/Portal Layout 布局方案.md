
## 核心思路

使用 **CSS Grid + Subgrid** 让 Header 与内容区共享同一套列轨道，内容区宽度自动与 Header 中间空白对齐，无需 JS。

## 布局结构

```
┌──────────────────────────────────────────────────────┐
│  Layout (grid 容器)                                    │
│  grid-template-columns: auto  1fr  auto               │
│  grid-template-rows:    auto  1fr                     │
├──────────┬────────────────────────┬──────────────────┤
│  Header (subgrid, col-span-full)                      │
│  col 1   │        col 2          │      col 3        │
│  Title   │       (空白)           │   用户信息        │
├──────────┼────────────────────────┼──────────────────┤
│          │  <main> col-start-2    │                   │
│          │  内容区只占中间列        │                   │
│          │  宽度 = Header空白宽度  │                   │
└──────────┴────────────────────────┴──────────────────┘
```

## 文件说明

### `layout/Layout.vue`

外层容器定义三列 Grid：

```html
<div class="w-screen h-screen grid overflow-hidden"
  style="grid-template-columns: auto 1fr auto; grid-template-rows: auto 1fr;">
  <Header class="col-span-full" />
  <main class="col-start-2 col-end-3 min-h-0 overflow-y-auto">
    <slot />
  </main>
</div>
```

- `auto 1fr auto`：左右列由内容撑开，中间列填满剩余空间
- Header 横跨所有列（`col-span-full`）
- `<main>` 只占第 2 列（`col-start-2 col-end-3`）

### `layout/Header.vue`

Header 使用 subgrid 继承父级列轨道：

```css
.portal-header {
  display: grid;
  grid-template-columns: subgrid;
  grid-column: 1 / -1;
  align-items: center;
}
```

模板内三个子元素自动分配到三列：

```html
<header class="portal-header">
  <!-- col 1: 左侧 Title -->
  <div class="flex items-center gap-3 pl-6">...</div>

  <!-- col 2: 中间占位 -->
  <div></div>

  <!-- col 3: 右侧用户信息 -->
  <div class="flex items-center justify-end pr-6">...</div>
</header>
```

## 关键点

1. **subgrid** 让 Header 的子元素与父 Grid 共享列轨道定义，从而 `<main>` 的宽度精确等于 Header 中间的空白宽度
2. 左右列是 `auto`，由 Title / 用户信息的实际宽度决定
3. 窗口缩放时自动适配，无需响应式断点或 JS 计算
4. 如果需要内容区带左右 padding，直接在 `<main>` 或 slot 内容上加即可
