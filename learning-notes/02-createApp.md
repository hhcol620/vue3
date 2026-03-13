# Vue 3 createApp 启动流程源码解析

> 核心文件：
> - `packages/runtime-dom/src/index.ts`
> - `packages/runtime-core/src/apiCreateApp.ts`
> - `packages/runtime-core/src/renderer.ts`
> - `packages/runtime-core/src/component.ts`

---

## 一、入口：`createApp`（`runtime-dom/index.ts:104`）

```typescript
export const createApp = ((...args) => {
  const app = ensureRenderer().createApp(...args)
  // ...
  const { mount } = app
  app.mount = (containerOrSelector) => { /* 包装后的 mount */ }
  return app
})
```

这里做了两件事：
1. 通过 `ensureRenderer()` 拿到渲染器，再调其 `createApp` 创建 app 对象
2. 对 `app.mount` 进行**二次包装**，补充 DOM 专属逻辑

---

## 二、`ensureRenderer()`（`runtime-dom/index.ts:80`）

```typescript
let renderer  // 模块级变量，初始 undefined

function ensureRenderer() {
  return (
    renderer || (renderer = createRenderer(rendererOptions))
  )
}
```

### 懒创建
模块加载时不创建渲染器，**第一次调用 `createApp()` 时才创建**。

### 单例
`renderer` 是模块级变量，`||=` 逻辑保证无论调用多少次，**渲染器只有一个实例**。

### 为什么要懒创建（tree-shake）
如果项目只用了响应式 API，没有调 `createApp`：
- 懒创建 → `ensureRenderer` 从未执行 → 打包工具 tree-shake 掉渲染器代码，bundle 更小
- 非懒创建 → 模块加载就执行，打包工具无法 tree-shake

### `rendererOptions`（`runtime-dom/index.ts:72`）
```typescript
const rendererOptions = extend({ patchProp }, nodeOps)
//                               ↑属性处理     ↑DOM 操作（insert/remove/createElement...）
```
这是传给渲染器的"浏览器 DOM 能力"，让核心渲染器与平台解耦。

---

## 三、`app.mount` 为什么要重新包装

**原始 `mount`**（`runtime-core/apiCreateApp.ts:358`）是平台无关的，只认 `HostElement` 对象，不处理任何 DOM 细节。

**包装后的 `mount`**（`runtime-dom/index.ts:113`）在调原始 `mount` 之前，做了四件 **DOM 专属的事**：

```typescript
app.mount = (containerOrSelector) => {

  // 1. 把字符串选择器转成真实 DOM 元素
  //    '#app' → document.querySelector('#app')
  const container = normalizeContainer(containerOrSelector)

  // 2. 没有 render/template 时，把容器的 innerHTML 当模板
  //    对应 in-DOM 模板写法：<div id="app"><h1>{{ msg }}</h1></div>
  if (!component.render && !component.template) {
    component.template = container.innerHTML
  }

  // 3. 挂载前清空容器内容（避免模板和 Vue 渲染内容重叠）
  container.textContent = ''

  // 4. 调原始 mount，挂完后做 DOM 收尾
  const proxy = mount(container, false, resolveRootNamespace(container))
  container.removeAttribute('v-cloak')      // 去掉 v-cloak
  container.setAttribute('data-v-app', '')  // 打上 data-v-app 标记
  return proxy
}
```

### `component.template` 赋值后在哪里被用

赋值在 `runtime-dom`，读取在 `runtime-core` 更深处：

```
runtime-dom/index.ts:123
  component.template = container.innerHTML    ← 写入

  mount(container, ...)                        ← line 143，调原始 mount
      ↓
runtime-core/apiCreateApp.ts:372
  createVNode(rootComponent)
      ↓
runtime-core/component.ts:1013
  const template = Component.template         ← 读取
  if (template) {
    compile(template)                          ← 编译成 render 函数
  }
```

**必须在 `mount()` 之前赋值**，否则组件初始化时读不到模板。

---

## 四、`createRenderer` 内部：`baseCreateRenderer`（`renderer.ts:342`）

`createRenderer` 本身只是一个转发：

```typescript
export function createRenderer(options) {
  return baseCreateRenderer(options)  // 直接透传
}
```

真正干活的是 `baseCreateRenderer`，它是一个**巨型闭包工厂函数**，做三件事：

### 第一件事：解构 DOM 操作能力（`renderer.ts:357`）

```typescript
const {
  insert:           hostInsert,        // 把节点插入 DOM
  remove:           hostRemove,        // 从 DOM 删除节点
  patchProp:        hostPatchProp,     // 处理 class/style/事件等属性
  createElement:    hostCreateElement, // 创建 DOM 元素
  createText:       hostCreateText,    // 创建文本节点
  setElementText:   hostSetElementText,
  parentNode:       hostParentNode,
  nextSibling:      hostNextSibling,
  ...
} = options   // ← runtime-dom 传进来的 rendererOptions
```

core 只认 `hostInsert` 这些名字，不知道底层是浏览器 DOM 还是其他平台——**跨平台的关键**。

### 第二件事：在闭包内定义所有渲染函数（互相引用，不对外暴露）

```
patch()                ← 所有更新的入口，按 vnode 类型分发
  ├─ processText()
  ├─ processFragment()
  ├─ processElement()      ← 处理普通 DOM 元素
  └─ processComponent()    ← 处理组件
       ├─ mountComponent()     ← 首次挂载
       │    ├─ createComponentInstance()
       │    ├─ setupComponent()    ← 执行 setup()
       │    └─ setupRenderEffect() ← 响应式与渲染的连接点
       └─ updateComponent()    ← 更新
```

`patch` 的分发逻辑（`renderer.ts:374`）：

```typescript
const patch = (n1, n2, container, ...) => {
  if (n1 === n2) return                   // 完全相同，跳过

  if (n1 && !isSameVNodeType(n1, n2)) {  // 类型变了，先卸载旧的
    unmount(n1, ...)
    n1 = null
  }

  switch (n2.type) {
    case Text:     processText()
    case Comment:  processCommentNode()
    case Static:   mountStaticNode()
    case Fragment: processFragment()
    default:
      if (ELEMENT)    processElement()    // <div> 等普通元素
      if (COMPONENT)  processComponent()  // 组件
      if (TELEPORT)   Teleport.process()
      if (SUSPENSE)   Suspense.process()
  }
}
```

### 第三件事：返回 3 个东西（`renderer.ts:2424`）

```typescript
return {
  render,                              // vnode → 真实 DOM（内部私有版本）
  hydrate,                             // SSR 水合用
  createApp: createAppAPI(render),     // ← render 预注入
}
```

---

## 五、为什么 `createAppAPI` 要预注入 `render`，而不用同层的那个 `render`

`runtime-dom/index.ts` 里有两个 `render`，**它们是完全不同的东西**：

### 公开的 `render`（`runtime-dom/index.ts:96`）—— 用户 API

```typescript
export const render = ((...args) => {
  ensureRenderer().render(...args)   // 包装了一层 ensureRenderer
})
```

给用户直接调用的，使用场景：
```typescript
import { render, h } from 'vue'
render(h('div', 'hello'), document.body)   // 不用 createApp，直接渲染
```

### 闭包内的 `render`（`renderer.ts:2376`）—— 内部实现

```typescript
const render: RootRenderFunction = (vnode, container, namespace) => {
  if (vnode == null) {
    if (container._vnode) unmount(...)     // 卸载
  } else {
    patch(container._vnode || null, vnode, container, ...)  // 挂载/更新
  }
  container._vnode = vnode
  flushPostFlushCbs()                      // 执行 post 钩子
}
```

可以直接访问闭包里的 `patch`、`unmount` 等**私有函数**，外部拿不到。

### 为什么要注入而不是直接调用

`createAppAPI` 在 `runtime-core` 里，它**不能 import runtime-dom**（否则循环依赖，core 反依赖 dom）。

解决方案：**依赖注入**

```
runtime-dom 创建渲染器
  → 拿到闭包内的私有 render
  → createAppAPI(render)   注入进去
  → app.mount() 内部就能调 render(vnode, container)
```

```
runtime-core                         runtime-dom
────────────                         ───────────
createAppAPI(render) ←── 注入 ──── baseCreateRenderer 闭包内的 render
     ↓                                    ↑ 可访问 patch/unmount 等私有函数
app.mount()
  render(vnode, container) ──────────────┘

export const render = (...) =>       ← 另一个，给用户直接用的公开 API
  ensureRenderer().render(...)          本质是对上面那个 render 的包装
```

---

## 七、平台分层设计思路

```
runtime-core   → 不知道 DOM，只管 vnode / 组件逻辑   （平台无关）
runtime-dom    → 知道 DOM，负责处理 DOM 专属细节      （浏览器平台）
```

`runtime-dom` 在外面包一层，补上浏览器才需要的逻辑，而不是把 DOM 代码写进 core 里。同一套 core 可以用在其他平台（`runtime-test`、NativeScript 等）。

---

## 八、`createApp` 完整调用链

```
你写的：createApp(App).mount('#app')
           ↓
runtime-dom/index.ts:104
  ensureRenderer()                     懒创建渲染器（单例）
      └─ createRenderer(rendererOptions)   runtime-core/renderer.ts
           rendererOptions 包含：
             patchProp                 处理 class/style/event 等属性
             insert/remove/create...  DOM 操作

  .createApp(App)                      runtime-core/apiCreateApp.ts
      └─ 返回 app 对象
           app.use() / app.component() / app.directive()
           app.mount()                 原始 mount（平台无关）

  重写 app.mount                       补充 DOM 专属逻辑
           ↓
app.mount('#app')
  normalizeContainer('#app')           '#app' → DOM 元素
  component.template = innerHTML       兜底 in-DOM 模板
  container.textContent = ''           清空容器
  mount(container)                     调原始 mount
    ↓
  apiCreateApp.ts:372
    createVNode(rootComponent)         包成 vnode
    render(vnode, container)           调渲染器 render
      ↓
  renderer.ts: patch(null, vnode)
    processComponent()
      mountComponent()
        createComponentInstance()      创建组件实例
        setupComponent()
          setup()                      执行用户的 setup()
                                       ← ref/reactive/computed 在这里建立
        setupRenderEffect()            ← 关键！响应式系统与渲染连接处
          new ReactiveEffect(componentUpdateFn, scheduler)
                                       组件渲染函数包进 ReactiveEffect
                                       渲染时读响应式数据 → dep.track()
                                       数据变化 → scheduler → 重新渲染
```

---

## 九、响应式系统与渲染系统的汇合点

**`setupRenderEffect()`**（`runtime-core/src/renderer.ts`）是两个系统的连接处：

```
reactivity 包                      runtime-core 包
────────────                       ────────────────
ref / reactive                     组件渲染函数
dep.track / dep.trigger    ←──── new ReactiveEffect(renderFn, queueJob)
                                   ↑
                                   renderFn 执行时读响应式数据 → track
                                   数据变化 → trigger → queueJob → 重渲染
```

```
数据变化后的完整流程：
  count.value++
    → dep.trigger()
    → effect.notify()
    → endBatch() → effect.scheduler() → queueJob(componentUpdateFn)
    → nextTick 后执行 componentUpdateFn
    → 重新执行 render()，生成新 vnode
    → patch(oldVNode, newVNode)
    → DOM 更新
```
