# Vue 3 响应式与渲染的连接点：setupRenderEffect

> 核心文件：`packages/runtime-core/src/renderer.ts`
> 核心函数：`setupRenderEffect`（第 ~1290 行）

---

## 在整个流程中的位置

```
app.mount('#app')
  └─► render(vnode, container)           renderer.ts:2376
        └─► patch(null, vnode, ...)       renderer.ts:374
              └─► processComponent()      renderer.ts:1146
                    └─► mountComponent()  renderer.ts:1183
                          └─► setupRenderEffect()  ← 本文重点
```

`patch` 本身只是分发器，连接响应式和渲染的真正位置是 `setupRenderEffect`。

---

## 核心代码（renderer.ts:1577-1601）

```typescript
// create reactive effect for rendering
instance.scope.on()
const effect = (instance.effect = new ReactiveEffect(componentUpdateFn))
instance.scope.off()

const update = (instance.update = effect.run.bind(effect))
const job: SchedulerJob = (instance.job = effect.runIfDirty.bind(effect))
job.i = instance
job.id = instance.uid
effect.scheduler = () => queueJob(job)   // ← 异步调度，不立即重渲染

toggleRecurse(instance, true)

update()  // ← 首次执行，触发渲染 + 依赖收集
```

---

## 三步理解

### 第一步：把渲染函数包进 ReactiveEffect

```typescript
const effect = new ReactiveEffect(componentUpdateFn)
```

`componentUpdateFn` 是"执行组件渲染 → 生成 vnode → patch 到 DOM"的完整流程：

```typescript
const componentUpdateFn = () => {
  if (!instance.isMounted) {
    // 首次挂载
    const subTree = (instance.subTree = renderComponentRoot(instance))
    patch(null, subTree, container, ...)   // 生成真实 DOM
    instance.isMounted = true
  } else {
    // 更新
    const nextTree = renderComponentRoot(instance)
    patch(instance.subTree, nextTree, ...)  // diff 后更新 DOM
    instance.subTree = nextTree
  }
}
```

包进 `ReactiveEffect` 后，执行时内部读到的所有响应式数据都会自动 **track（收集依赖）**。

### 第二步：配置异步调度器

```typescript
effect.scheduler = () => queueJob(job)
```

默认 effect 数据变化时会立刻同步重新执行。但渲染不能这么暴力——同一帧内多次数据变化会触发多次渲染。

换成 scheduler 后：数据变了 → 把渲染任务推入微任务队列 → 下一个 tick 批量执行一次渲染。

### 第三步：首次执行 update()

```typescript
update()  // 即 effect.run()
```

首次执行 `componentUpdateFn`：
- `renderComponentRoot(instance)` 执行组件的 render 函数
- 过程中读取 `ref.value` / `reactive` 的属性 → 触发 getter → `dep.track(effect)`
- 依赖关系建立完成，之后数据变化就能通知到这个 effect

---

## 完整连接示意

```
setupRenderEffect()
  │
  ├─ new ReactiveEffect(componentUpdateFn)
  │
  ├─ effect.scheduler = () => queueJob(job)
  │
  └─ update()  ← 首次运行
       │
       ├─ renderComponentRoot()    执行组件 render 函数
       │     读取 count.value  →   dep.track(effect)  收集依赖
       │
       └─ patch(null, subTree)     生成真实 DOM

          ↓ 之后数据变化时

  count.value++
    → dep.trigger()
    → effect.scheduler()
    → queueJob(job)
    → nextTick 后执行 componentUpdateFn
    → renderComponentRoot() 生成新 vnode
    → patch(oldTree, newTree)
    → DOM 更新
```

---

## 为什么说 patch 不是连接点

`patch` 只是一个分发器，自身不感知响应式，它只接收新旧 vnode 做 DOM diff。

真正让"数据变化 → 重渲染"成立的，是 `ReactiveEffect(componentUpdateFn)` 这一行——把渲染变成了一个**响应式副作用**：
- 渲染时自动收集依赖（track）
- 数据变化时自动调度重渲染（scheduler → queueJob）

`patch` 只是每次重渲染时被动调用的工具。

---

## 面试角度总结

**问：Vue 3 响应式是怎么驱动视图更新的？**

> 组件挂载时，`setupRenderEffect` 把组件的渲染函数包进 `ReactiveEffect`，首次执行时自动收集所有用到的响应式数据作为依赖。数据变化后触发 `scheduler`，把更新任务推入微任务队列，下一个 tick 批量执行，生成新 vnode 后调 `patch` 做 DOM diff 更新视图。

---

## 暂跳过的部分

- `renderComponentRoot` 内部（如何执行 render 函数、处理 slots 等）
- `queueJob` / `scheduler.ts` 队列调度机制
- `instance.scope` 的作用（effect 作用域，组件卸载时批量停止）
