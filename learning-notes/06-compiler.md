# 模板编译：template → render 函数

> 源码路径：`packages/compiler-core/src/`、`packages/compiler-dom/src/`
> 学习目标：建立编译流程轮廓，应对面试

---

## 一、核心目标

> **`<template>` 字符串 → `render` 函数**

编译器和运行时完全独立：compiler 不依赖 runtime，输出的 render 函数才被 runtime 消费。

```
开发者写的 template
       ↓  compiler
   render 函数
       ↓  runtime
     真实 DOM
```

---

## 二、三个阶段

```
template 字符串
      ↓
① parse（解析）        compiler-core/src/parse.ts
      → AST（抽象语法树）
      ↓
② transform（转换）    compiler-core/src/compile.ts
      → 优化 AST（静态提升、指令转换、PatchFlag 标记）
      ↓
③ generate（生成）     compiler-core/src/codegen.ts
      → render 函数字符串 → new Function() → 真正的函数
```

---

## 三、第一步：parse → AST

把模板字符串解析成树形结构：

```html
<div class="app">
  <h1>{{ title }}</h1>
  <button @click="onClick">click</button>
</div>
```

解析结果：

```javascript
{
  type: 'Element', tag: 'div',
  props: [{ name: 'class', value: 'app' }],
  children: [
    {
      type: 'Element', tag: 'h1',
      children: [{ type: 'Interpolation', content: 'title' }]
    },
    {
      type: 'Element', tag: 'button',
      props: [{ name: 'onClick', ... }],
      children: [{ type: 'Text', content: 'click' }]
    }
  ]
}
```

---

## 四、第二步：transform → 优化 AST

这步做两件最重要的事：

### 静态提升（hoistStatic）

```html
<div>
  <p>纯静态，永远不变</p>   <!-- 提升到 render 函数外面，只创建一次 -->
  <p>{{ dynamic }}</p>       <!-- 每次渲染重新创建 -->
</div>
```

静态节点提升到外部变量，diff 时直接跳过，不参与比对。

### PatchFlag 标记

编译期分析出节点哪部分会变，打上标记：

```javascript
createVNode('p', null, title, PatchFlag.TEXT)
//                             ↑ 只有文本会变
```

常见 PatchFlag：

| Flag | 含义 |
|------|------|
| `TEXT = 1` | 动态文本 |
| `CLASS = 2` | 动态 class |
| `STYLE = 4` | 动态 style |
| `PROPS = 8` | 动态 props |
| `FULL_PROPS = 16` | 有动态 key 的 props，需要全量 diff |

运行时 diff 时只比对有 PatchFlag 的部分，不全量比对所有属性。

---

## 五、第三步：generate → render 函数

最终生成的 render 函数：

```javascript
// 静态节点提升到外部，只创建一次
const _hoisted = createVNode('p', null, '纯静态')

function render(_ctx) {
  return createVNode('div', { class: 'app' }, [
    _hoisted,                                            // 直接复用，不参与 diff
    createVNode('h1', null, _ctx.title, 1 /* TEXT */),  // PatchFlag=1，只 diff 文本
    createVNode('button', { onClick: _ctx.onClick }, 'click')
  ])
}
```

生成字符串后通过 `new Function(code)` 变成真正可执行的函数，挂到 `instance.render`。

---

## 六、编译发生在什么时候

| 场景 | 时机 | 使用的包 |
|------|------|---------|
| 使用 vue-loader / vite-plugin-vue | **构建时**编译，用户拿到的已经是 render 函数 | `@vue/compiler-sfc` |
| 使用完整版 Vue（含 compiler） | **运行时**编译，浏览器里实时编译 template | `@vue/compiler-dom` |
| `<script setup>` | 构建时，由 SFC 编译器处理 | `@vue/compiler-sfc` |

生产环境推荐构建时编译，体积更小（不需要打包 compiler）。

---

## 七、面试常考

**Q：Vue 3 模板编译分几步？**

> 三步：① `parse` 把 template 字符串解析成 AST；② `transform` 优化 AST（静态提升、PatchFlag 标记、指令转换）；③ `generate` 把 AST 生成 render 函数字符串，再通过 `new Function()` 变成可执行函数。

---

**Q：什么是静态提升？有什么用？**

> 编译阶段识别出永远不会变化的节点，将其提升到 render 函数外部作为常量。每次组件重渲染时直接复用这个常量，不重新创建 vnode，diff 时也直接跳过，减少了不必要的内存分配和比对开销。

---

**Q：什么是 PatchFlag？**

> 编译期给动态节点打的标记，标明该节点哪部分是动态的（文本、class、style、props 等）。运行时 diff 时只比对有 PatchFlag 对应的部分，跳过静态属性的比对，实现精准更新而非全量 diff。

---

**Q：compiler 和 runtime 是什么关系？**

> 完全独立，compiler 包不依赖 runtime 包。compiler 的职责是把 template 编译成 render 函数（字符串形式），runtime 的职责是执行 render 函数生成 vnode 并操作 DOM。两者通过 render 函数解耦，生产环境可以只打包 runtime，不包含 compiler。

---

**Q：`<script setup>` 是在什么时候编译的？**

> 构建时由 `@vue/compiler-sfc` 处理，编译成标准的 `setup()` 函数形式，浏览器拿到的已经是编译后的 JS，没有任何 template 字符串，所以运行时不需要 compiler。

---

## 八、暂跳过的部分

- `parse` 内部的有限状态机解析细节
- `transform` 中各个 nodeTransforms 插件（v-if、v-for、v-model 等指令的转换逻辑）
- `compiler-dom` 相对于 `compiler-core` 扩展了哪些平台特定转换
- Block Tree 优化（配合 PatchFlag 的动态节点收集机制）
