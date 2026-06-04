
> 一份面向前端开发者的详细技术教程，涵盖核心概念、API 语法、实战场景与踩坑指南。

---

## 一、核心概念：为什么需要 IntersectionObserver？ 

### 1.1 什么是交叉观察器？

`IntersectionObserver`（交叉观察器）是浏览器提供的一个原生 API，它能够**异步地**观察一个目标元素与其祖先元素或视口（viewport）的交叉状态变化。简单来说，它可以告诉你：「某个元素是否进入了可视区域，以及进入了多少」。

它的回调由浏览器在主线程**空闲时**调用（基于 `requestIdleCallback` 类似机制），不会阻塞渲染。

### 1.2 传统 scroll 事件的痛点

在 `IntersectionObserver` 出现之前，要判断元素是否进入可视区域，通常使用：

```js
window.addEventListener('scroll', () => {
  const rect = el.getBoundingClientRect();
  if (rect.top < window.innerHeight && rect.bottom > 0) {
    // 元素可见
  }
});
```

这种方式存在以下问题：

| 问题                           | 说明                                                |
| ---------------------------- | ------------------------------------------------- |
| **性能开销大**                    | `scroll` 事件触发频率极高（每次滚动可能触发数十次），每次都在主线程同步执行        |
| **强制同步布局（Layout Thrashing）** | `getBoundingClientRect()` 会触发浏览器重新计算布局，长列表场景下卡顿明显 |
| **需要手动节流防抖**                 | 必须配合 `throttle` / `debounce` 才能勉强可用，但又会牺牲精度       |
| **代码冗长**                     | 多元素、多阈值的场景下逻辑复杂，可维护性差                             |

而 `IntersectionObserver`：

- **异步执行**，回调在浏览器空闲时调度，不阻塞滚动
- **由浏览器原生优化**，无需手动节流
- **声明式 API**，配置阈值即可，逻辑清晰
- **天然支持多元素观察**，性能远优于手写 scroll 监听

---

## 二、核心语法详解

### 2.1 基本用法

```js
const observer = new IntersectionObserver(callback, options);
observer.observe(targetElement);
```

### 2.2 callback 回调

```js
const callback = (entries, observer) => {
  entries.forEach(entry => {
    // entry 是 IntersectionObserverEntry 实例
    console.log(entry.isIntersecting);      // 是否相交（进入可视区）
    console.log(entry.intersectionRatio);   // 相交比例 0~1
    console.log(entry.target);              // 被观察的目标元素
    console.log(entry.boundingClientRect);  // 目标元素的位置矩形
    console.log(entry.intersectionRect);    // 相交区域矩形
    console.log(entry.rootBounds);          // 根元素的矩形
    console.log(entry.time);                // 触发时间戳
  });
};
```

注意 `entries` 是数组，因为一个 observer 可观察多个目标。

### 2.3 options 配置三剑客

#### root —— 参照的根元素

- 默认 `null`，表示以浏览器**视口（viewport）**作为参照
- 可以指定一个**祖先元素**（必须是目标的祖先，且需要有滚动容器特性，如 `overflow: auto`）

```js
{ root: document.querySelector('.scroll-container') }
```

```
┌───────────── viewport (root=null) ─────────────┐
│                                                │
│  ┌──── .scroll-container (root) ───┐           │
│  │                                  │           │
│  │   [target]  ←—— 观察这个元素      │           │
│  │                                  │           │
│  └──────────────────────────────────┘           │
└────────────────────────────────────────────────┘
```

#### rootMargin —— 根元素的「外扩/内缩」边距

- 类似 CSS 的 `margin` 写法：`"10px 20px 30px 40px"`（上右下左）
- 支持 `px` 和 `%`（**不支持** `em`/`rem`）
- **正值表示扩大根的判定范围**（提前触发），**负值表示缩小**（延后触发）

```js
{ rootMargin: '200px 0px 0px 0px' }  // 根元素顶部"虚拟"上扩 200px
```

懒加载场景中，常常设置 `rootMargin: '200px'` 让图片在距离视口 200px 时就提前加载，体验更顺滑。

```
                    ┌─ rootMargin (虚拟扩大区) ─┐
                    │                            │
                    │    ┌──── 视口 ────┐        │
                    │    │              │        │
                    │    │              │        │
                    │    └──────────────┘        │
                    │                            │
                    └────────────────────────────┘
                              ↑
                    元素只要进入这个虚拟区就触发
```

#### threshold —— 触发阈值

决定**相交比例达到多少时**触发回调：

- `0`（默认）：目标刚露出 1 像素就触发
- `1`：目标完全进入根时才触发
- `0.5`：目标一半可见时触发
- 数组 `[0, 0.25, 0.5, 0.75, 1]`：每个比例点都触发一次（用于实现进度跟踪、动画）

```js
{ threshold: [0, 0.25, 0.5, 0.75, 1] }
```

### 2.4 实例方法

|方法|作用|
|---|---|
|`observer.observe(target)`|开始观察一个目标|
|`observer.unobserve(target)`|停止观察某个目标（其他目标继续观察）|
|`observer.disconnect()`|断开所有观察，释放 observer|
|`observer.takeRecords()`|手动获取尚未派发的 entries（罕用）|

---

## 三、实战场景

### 3.1 图片懒加载（Lazy Loading）

#### JavaScript 版本

```html
<img data-src="/images/photo-1.jpg" alt="" class="lazy" />
<img data-src="/images/photo-2.jpg" alt="" class="lazy" />
```

```js
const lazyImages = document.querySelectorAll('img.lazy');

const imageObserver = new IntersectionObserver(
  (entries, observer) => {
    entries.forEach(entry => {
      if (!entry.isIntersecting) return;

      const img = entry.target;
      const src = img.dataset.src;
      if (src) {
        img.src = src;
        img.removeAttribute('data-src');
      }

      // 加载后立刻取消观察，避免无谓回调
      observer.unobserve(img);
    });
  },
  {
    root: null,
    rootMargin: '200px 0px',  // 提前 200px 开始加载
    threshold: 0.01,
  }
);

lazyImages.forEach(img => imageObserver.observe(img));
```

#### TypeScript 版本

```ts
function setupLazyLoad(selector: string = 'img.lazy'): IntersectionObserver {
  const images = document.querySelectorAll<HTMLImageElement>(selector);

  const observer = new IntersectionObserver(
    (entries, obs) => {
      for (const entry of entries) {
        if (!entry.isIntersecting) continue;
        const img = entry.target as HTMLImageElement;
        const src = img.dataset.src;
        if (src) {
          img.src = src;
          delete img.dataset.src;
        }
        obs.unobserve(img);
      }
    },
    { rootMargin: '200px 0px', threshold: 0.01 }
  );

  images.forEach(img => observer.observe(img));
  return observer;
}
```

> 现代浏览器原生支持 `<img loading="lazy">`，简单场景下可直接使用。但 `IntersectionObserver` 提供更精细的控制（自定义阈值、占位图切换、上报曝光等）。

### 3.2 无限滚动加载（Infinite Scroll）

核心思路：在列表底部放一个**哨兵元素（sentinel）**，观察它何时进入视口。

```html
<ul id="list">
  <!-- 已加载的列表项 -->
</ul>
<div id="sentinel">加载中...</div>
```

```ts
interface Item {
  id: number;
  title: string;
}

let page = 1;
let loading = false;
let hasMore = true;

async function fetchItems(p: number): Promise<Item[]> {
  const res = await fetch(`/api/items?page=${p}`);
  return res.json();
}

function appendItems(items: Item[]): void {
  const list = document.getElementById('list')!;
  const frag = document.createDocumentFragment();
  items.forEach(item => {
    const li = document.createElement('li');
    li.textContent = item.title;
    frag.appendChild(li);
  });
  list.appendChild(frag);
}

const sentinel = document.getElementById('sentinel')!;

const scrollObserver = new IntersectionObserver(
  async entries => {
    const entry = entries[0];
    if (!entry.isIntersecting || loading || !hasMore) return;

    loading = true;
    try {
      const items = await fetchItems(page);
      if (items.length === 0) {
        hasMore = false;
        scrollObserver.disconnect();
        sentinel.textContent = '没有更多了';
        return;
      }
      appendItems(items);
      page += 1;
    } finally {
      loading = false;
    }
  },
  { rootMargin: '300px 0px' }  // 距离底部 300px 时提前加载
);

scrollObserver.observe(sentinel);
```

---

## 四、注意踩坑

### 4.1 内存泄漏：什么时候该 unobserve / disconnect？

`IntersectionObserver` 持有目标元素的强引用。如果元素已从 DOM 移除但 observer 仍在观察，会导致：

- 元素无法被 GC 回收
- 回调可能继续触发，引发逻辑错误

**正确做法：**

|场景|处理方式|
|---|---|
|一次性触发（如懒加载）|触发后立刻 `observer.unobserve(target)`|
|单页面退出 / 组件卸载|调用 `observer.disconnect()` 释放所有观察|
|动态增删列表项|移除 DOM 前先 `unobserve` 对应元素|

### 4.2 React 中的正确姿势

使用 `useEffect` 在挂载时建立观察，卸载时销毁：

```tsx
import { useEffect, useRef, useState } from 'react';

function LazyImage({ src, alt }: { src: string; alt: string }) {
  const imgRef = useRef<HTMLImageElement>(null);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    const el = imgRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            setLoaded(true);
            observer.unobserve(entry.target);
          }
        });
      },
      { rootMargin: '200px' }
    );

    observer.observe(el);

    // 关键：清理函数中 disconnect
    return () => observer.disconnect();
  }, []);

  return <img ref={imgRef} src={loaded ? src : undefined} alt={alt} />;
}
```

**封装通用 Hook：**

```tsx
function useIntersection<T extends Element>(
  options?: IntersectionObserverInit
): [React.RefObject<T>, boolean] {
  const ref = useRef<T>(null);
  const [isIntersecting, setIntersecting] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const observer = new IntersectionObserver(
      ([entry]) => setIntersecting(entry.isIntersecting),
      options
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, [options?.root, options?.rootMargin, options?.threshold?.toString()]);

  return [ref, isIntersecting];
}
```

**易踩坑点：**

- 不要把 observer 实例放进 state，会触发额外渲染
- 依赖数组中如果有 `options`，需注意引用稳定性，可用 `useMemo` 包裹
- React 严格模式下 `useEffect` 会执行两次，必须正确返回清理函数

### 4.3 Vue 3 中的正确姿势

使用 `onMounted` / `onBeforeUnmount`：

```vue
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';

const target = ref<HTMLElement | null>(null);
const visible = ref(false);
let observer: IntersectionObserver | null = null;

onMounted(() => {
  if (!target.value) return;
  observer = new IntersectionObserver(
    ([entry]) => { visible.value = entry.isIntersecting; },
    { rootMargin: '100px' }
  );
  observer.observe(target.value);
});

onBeforeUnmount(() => {
  observer?.disconnect();
  observer = null;
});
</script>

<template>
  <div ref="target">{{ visible ? '可见' : '不可见' }}</div>
</template>
```

> 如果使用 [VueUse](https://vueuse.org/)，可直接用 `useIntersectionObserver`，它已自动处理生命周期清理。

### 4.4 其他常见坑

1. **root 必须是 target 的祖先**：否则 observer 永远不会触发
2. **隐藏元素（`display: none`）不会触发**：因为没有布局
3. **iframe 跨域**：跨域 iframe 内的元素无法被外部 observer 观察
4. **rootMargin 在跨域 iframe 中失效**：浏览器为安全考虑会忽略
5. **threshold 只触发一次的误解**：每次跨越阈值（双向）都会触发，要靠 `isIntersecting` 自行判断方向
6. **首次 observe 会立即触发一次回调**：用于上报初始可见状态

---

## 五、浏览器兼容性

`IntersectionObserver` 在所有现代浏览器中均已支持（Chrome 51+、Firefox 55+、Safari 12.1+、Edge 15+）。如需兼容更老的浏览器，可使用官方 polyfill：

```bash
npm install intersection-observer
```

```js
import 'intersection-observer';  // 入口顶部引入
```

---
## 六. 阈值（threshold）
就是「触发回调的临界比例」——它告诉浏览器：**目标元素与根（视口/容器）的相交面积达到多少时，应该通知我**。

## 直观理解

想象你在观察一张图片是否「进入屏幕」：

- `threshold: 0` —— 图片**刚露出 1 个像素**就通知你（默认值）
- `threshold: 0.5` —— 图片有**一半**进入视口才通知你
- `threshold: 1` —— 图片**完全**进入视口才通知你

这个比例叫做 `intersectionRatio`，取值 `0 ~ 1`。

## 图示

```
threshold: 0.5  （元素一半可见时触发）

  视口边界 ─────────────────
            ┌──────────┐    ← 元素 50% 在视口内
            │  可见部分 │       此刻触发回调
  ──────────┼──────────┤
            │  不可见   │
            └──────────┘
```

## 单值 vs 数组

**单个值**：只在跨越这个比例时触发

```js
{ threshold: 0.5 }  // 50% 时触发一次
```

**数组**：每跨越一个比例点都触发一次，可以做"渐进式"效果

```js
{ threshold: [0, 0.25, 0.5, 0.75, 1] }
// 元素从不可见到完全可见的过程中，会触发 5 次
// 可用于实现"滚动进度动画"，比如根据 intersectionRatio 改变透明度
```

## 易混淆点

很多人以为 `threshold: 0.5` 意味着"超过 50% 就一直触发"——其实不是。它的真实语义是：**每次穿越这个阈值（无论是进入还是离开方向）触发一次**。所以回调里仍然要用 `entry.isIntersecting` 判断当前是"进"还是"出"。

```js
new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      console.log('元素 50% 可见了，进入');
    } else {
      console.log('元素低于 50% 可见，离开');
    }
  });
}, { threshold: 0.5 });
```

简单一句话总结：**threshold = 触发回调所需的"可见比例"门槛**。
## 知识来源

- [MDN Web Docs — IntersectionObserver API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
- [MDN Web Docs — IntersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver)
- [MDN Web Docs — IntersectionObserverEntry](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserverEntry)
- [W3C Specification — Intersection Observer](https://www.w3.org/TR/intersection-observer/)
- [web.dev — Lazy-loading images using IntersectionObserver](https://web.dev/articles/lazy-loading-images)
- [Chrome for Developers — IntersectionObserver's Coming into View](https://developer.chrome.com/blog/intersectionobserver/)
- [W3C polyfill — w3c/IntersectionObserver](https://github.com/w3c/IntersectionObserver)
- [VueUse — useIntersectionObserver](https://vueuse.org/core/useIntersectionObserver/)

> 上述链接为公开技术文档与规范，本教程中的概念阐述与代码示例参考其内容并结合工程实践改写。