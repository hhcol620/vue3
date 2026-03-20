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
const job: SchedulerJob = (instance.job = effect.runIfDirty.bind(effect))
job.i = instance
job.id = instance.uid
effect.scheduler = () => queueJob(job)
```

**注意**：这几行全部是赋值，**没有任何依赖收集发生**，只是在"准备工作"：
- `job` 绑定的是 `runIfDirty`（只有 dirty 时才执行，避免重复渲染）
- `job.i` / `job.id` 挂载实例信息，供调度器识别和排序
- `effect.scheduler` 赋值只是告诉 effect："将来你被触发时，用这个方式通知我"

**`scheduler` 何时被调用？** 在 `ReactiveEffect.trigger()` 内部：

```typescript
// reactivity/src/effect.ts
trigger(): void {
  if (this.flags & EffectFlags.PAUSED) {
    pausedQueueEffects.add(this)
  } else if (this.scheduler) {
    this.scheduler()        // ← 有 scheduler 就调用它（异步入队）
  } else {
    this.runIfDirty()       // ← 没有 scheduler 就直接同步执行
  }
}
```

触发链：响应式数据 `.set` → 通知订阅的 effect → `effect.trigger()` → `scheduler()` → `queueJob(job)` → 异步批量执行。

默认 effect 数据变化时会立刻同步重新执行。但渲染不能这么暴力——同一帧内多次数据变化会触发多次渲染。

换成 scheduler 后：数据变了 → 把渲染任务推入微任务队列 → 下一个 tick 批量执行一次渲染。

| | 触发时机 | 执行条件 |
|---|---|---|
| `update` (`effect.run`) | 主动调用，如首次挂载 | 无条件执行 |
| `job` (`effect.runIfDirty`) | 响应式数据变更后经调度器异步执行 | 只有 dirty 才执行 |

### 第三步：首次执行 update()

```typescript
update()  // 即 effect.run()，无条件同步执行
```

**这里才是依赖收集真正发生的时机**。`effect.run()` 执行时：

```
effect.run()
  └── activeEffect = this        ← 把当前 effect 设为"激活状态"
      └── componentUpdateFn()
          └── renderComponentRoot()
              └── 执行组件 render 函数
                  └── 访问 ref/reactive 数据
                      └── 触发 getter → track()
                          └── 把 activeEffect 收集到该数据的订阅列表里
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
