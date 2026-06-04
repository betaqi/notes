### Q1: 构建分包策略是怎么设计的？
> 分包策略用的是 Rollup 的 `manualChunks`，手动把比较大依赖拆出来  用户访问不同页面时，这三个 chunk 可以走浏览器缓存，不会因为业务代码变更而失效。其他依赖体积不大，交给 Vite 自动处理就够了。
### Q2: 迁移过程中遇到的最大问题是什么？
1. **入口配置差异**：Webpack 用 `entry` 字段指定入口，Vite 用 `index.html` 作为入口 + 插件注入 JS。项目用了 `vite-plugin-index-html` 来模拟 Webpack 的入口行为，支持双入口切换（pmone/npm）。证据：`vite.dev.conf.js:10-14`
2. **CommonJS 依赖兼容**：项目依赖了 `jquery`、`codemirror`、`quill` 等 CommonJS 模块，Vite 原生只支持 ESM。生产构建里专门加了 `rollup-plugin-commonjs` 和 `rollup-plugin-node-resolve`。证据：`vite.prod.conf.js:3-4, 56-59`
3. **Vue 运行时编译**：项目用了 `createApp({ template: '<App/>' })` 这种需要运行时编译器的写法，必须把 vue 别名指向 `vue/dist/vue.esm-bundler.js`。证据：`vite.prod.conf.js:17-19`，`src/pmone.js:2`
4. **globEager 替代 require.context**：Webpack 的 `require.context` 在 Vite 里不能用，schema 批量加载改成了 `import.meta.globEager`。证据：`src/views/dashboard/schema/index.js:13`

#### 面试时的讲法： 
> 最大的问题是依赖兼容。我们有 78 万行代码，依赖了不少 CommonJS 模块——jQuery、CodeMirror、Quill 这些。Vite 开发模式走 ESM，这些库直接 import 会报错。解决方案是生产构建里加 rollup-plugin-commonjs 做转换，开发模式靠 Vite 的依赖预构建（pre-bundling）自动处理。另外一个坑是 Webpack 的 require.context 在 Vite 里没有对应 API，我们的 schema 批量加载全部改成了 import.meta.globEager。