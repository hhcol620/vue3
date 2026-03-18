# Vue 3 Renderer 源码导读：baseCreateRenderer 学习策略

> 核心文件：`packages/runtime-core/src/renderer.ts`
> 核心函数：`baseCreateRenderer`（约 2000 行，定义 ~30 个内部函数）

---

## 核心认知

`baseCreateRenderer` 里所有 `const xxx = ...` 的函数构成一个**封闭函数族**，共享同一闭包（`rendererOptions`）。

**不要横向全读，要按纵向主线深入。**

贯穿整个 renderer 的核心判断：
- `n1 === null` → 首次挂载
- `n1 !== null` → 更新

`patch` 是总入口，所有渲染最终都回到 patch，形成**递归树**而非链条。

---

## 学习主线顺序

### 第一条主线：首次渲染（最先学）

```
render()                         ← 入口，第 2376 行
  └─► patch(null, vnode, ...)    ← n1=null 表示首次挂载，第 374 行
        └─► processComponent()   ← 处理组件类型，第 1146 行
              └─► mountComponent() ← 首次挂载组件，第 1183 行
                    └─► setupRenderEffect() ← 建立响应式渲染，第 ~1300 行
                          └─► renderComponentRoot() → patch() ← 递归
```

### 第二条主线：元素节点（看完组件再看）

```
patch()
  └─► processElement()           ← 第 595 行
        ├─► mountElement()       ← 首次挂载 DOM，第 649 行
        └─► patchElement()       ← 更新 DOM，第 815 行
```

### 第三条主线：更新流程（响应式触发后）

```
effect 触发 → update()
  └─► patch(n1, n2, ...)         ← n1≠null 表示更新
        └─► patchElement() / updateComponent()
              └─► patchChildren()         ← 第 1623 行
                    └─► patchKeyedChildren() ← diff 算法，第 1785 行
```

---

## 暂时可跳过的函数

| 函数 | 跳过理由 |
|------|---------|
| `processText` / `processCommentNode`（493, 508）| 文本/注释节点，边缘场景 |
| `mountStaticNode` / `patchStaticNode`（526, 547）| v-once 静态节点优化 |
| `processFragment`（1038）| Fragment 特殊情况，理解组件后再看 |
| `patchUnkeyedChildren`（1725）| 次要路径，先看 keyed diff |
| `unmount` / `unmountComponent`（2108, 2290）| 卸载流程，理解挂载后自然懂 |
| `patchBlockChildren`（956）| 编译器优化（Block Tree），后期看 |

---

## 防陷阱原则

1. **遇到不认识的函数，先看函数名 + 参数猜它做什么，再决定要不要进去**
2. **`patch` 是递归的，不是线性链**，子组件的渲染也是通过 patch 重入
3. **先跑通最简单 case**：`<div>hello</div>` 的渲染路径，再扩展到组件
4. `setupRenderEffect` 是连接响应式系统和渲染的桥梁，是整个 renderer 最核心的地方

---

## render 与 container._vnode（第 2376 行）

`render(vnode, container)` 是整个渲染树的根入口：

```typescript
const render: RootRenderFunction = (vnode, container, namespace) => {
  if (vnode == null) {
    if (container._vnode) unmount(container._vnode, ...)  // 卸载
  } else {
    patch(container._vnode || null, vnode, container, ...)  // 挂载或更新
  }
  container._vnode = vnode  // 渲染完成后保存，作为下次的"老 vnode"
}
```

`container._vnode` 是挂在 DOM 节点上的自定义属性，作用是**保存上一次的渲染结果**：

```
首次渲染：patch(null, vnode, ...)               → container._vnode = vnode
数据更新：patch(container._vnode, newVnode, ...) → container._vnode = newVnode
卸载：    vnode 传 null → unmount(container._vnode)  → container._vnode = null
```

### render 完成后的 flush 回调（第 2397-2402 行）

```typescript
if (!isFlushing) {
  isFlushing = true
  flushPreFlushCbs(instance)   // 执行 pre 队列：watch(flush:'pre') 的回调
  flushPostFlushCbs()           // 执行 post 队列：onMounted / onUpdated 等
  isFlushing = false
}
```

Vue 里的副作用不立即执行，而是推入队列批量处理：

| 队列 | 对应 API | 时机 |
|------|---------|------|
| pre flush | `watch(flush:'pre')` | DOM 更新后、生命周期前 |
| post flush | `onMounted` / `onUpdated` | DOM 完全稳定后 |

`isFlushing` 标志位的作用：防止在 flush 过程中（如 `onMounted` 里修改数据）再次触发 `render`，形成递归 flush 死循环。

> `flushPreFlushCbs` / `flushPostFlushCbs` 的队列调度机制在 `scheduler.ts`，暂跳过。

---

## patch 函数详解（第 374 行）

### 参数含义

```typescript
patch(
  n1,              // 老 vnode（首次挂载为 null）
  n2,              // 新 vnode（目标状态）
  container,       // 挂载的父 DOM 节点
  anchor,          // 插入锚点：insertBefore(el, anchor)，null 则 appendChild
  parentComponent, // 父组件实例（用于 provide/inject、错误冒泡）
  parentSuspense,  // 最近的 Suspense 边界
  namespace,       // 'svg' | 'mathml' | undefined
  slotScopeIds,    // scoped CSS slot 相关
  optimized,       // 是否走编译器优化路径（Block Tree）
)
```

重点关注前 4 个，后面的在主流程里基本透传。

### 内部三步逻辑

**第一步：快速剪枝**（385-398 行）

```typescript
if (n1 === n2) return  // 同一对象，直接跳过

if (n1 && !isSameVNodeType(n1, n2)) {
  // 类型变了（div→span，或组件A→组件B）
  // 无法复用，卸载老的，n1 置 null，让后续走"首次挂载"逻辑
  unmount(n1, ...)
  n1 = null
}
```

**第二步：按 vnode 类型分发**（401-483 行）

```
n2.type 的值            走哪个处理函数
────────────────────────────────────────────────────
Symbol(Text)         → processText
Symbol(Comment)      → processCommentNode
Symbol(Static)       → mountStaticNode（v-once 静态节点）
Symbol(Fragment)     → processFragment
'div'/'span'/...     → shapeFlag & ELEMENT    → processElement
组件对象              → shapeFlag & COMPONENT  → processComponent
Teleport             → shapeFlag & TELEPORT   → TeleportImpl.process
Suspense             → shapeFlag & SUSPENSE   → SuspenseImpl.process
```

`type` 是字符串（原生标签） → `processElement`
`type` 是对象（组件定义）   → `processComponent`

**第三步：处理 ref**（486-490 行）

```typescript
if (ref != null && parentComponent) {
  setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2)
} else if (ref == null && n1 && n1.ref != null) {
  setRef(n1.ref, null, parentSuspense, n1, true)
}
```

两种场景含义：

| 场景 | 条件 | 行为 |
|------|------|------|
| n2 有 ref | `ref != null` | 绑定/更新 ref（首次挂载或更新时） |
| n2 没有 ref，n1 有 ref | `ref == null && n1.ref != null` | 清理老 ref，防止残留脏引用 |

典型触发清理的场景：`<div v-if="show" ref="el">` 当 show 变 false，新 vnode 是 Comment 节点没有 ref，必须把 `el` 置为 null。

> ⚠️ `setRef` 内部未深入，暂跳过。

### patch 的作用总结

> **patch 是纯粹的"分发器"**，自身不做任何 DOM 操作，只根据 vnode 类型转发给对应的 `process*` 函数，所有真正的 mount/update 逻辑都在下游。

```
patch(n1, n2)
  ├─ n1 === n2 ──────────────────────→ return
  ├─ 类型不同 → unmount(n1)，n1=null → 走挂载路径
  └─ 按 type/shapeFlag 分发
        ├─ processElement(n1, n2)
        │     ├─ n1==null → mountElement
        │     └─ n1!=null → patchElement
        └─ processComponent(n1, n2)
              ├─ n1==null → mountComponent
              └─ n1!=null → updateComponent
```

---

## 下一步深入

- [ ] `mountComponent` → `setupRenderEffect` 详细分析
- [ ] `patchKeyedChildren` diff 算法（最长递增子序列）
