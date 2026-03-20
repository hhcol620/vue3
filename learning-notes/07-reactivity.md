# Vue 3 响应式系统源码解析

> 源码路径：`packages/reactivity/src/`
> 学习方式：从使用场景反推源码

---

## 核心文件一览

| 文件 | 职责 |
|------|------|
| `ref.ts` | `ref()` / `RefImpl` |
| `reactive.ts` | `reactive()`，Proxy 创建与缓存 |
| `dep.ts` | `Dep` / `Link` 数据结构，`track()` / `trigger()` |
| `effect.ts` | `ReactiveEffect` 类，`activeSub`，批处理 |
| `computed.ts` | `computed()` / `ComputedRefImpl` |
| `watch.ts` | `watch` / `watchEffect` |

---

## 一、`const count = ref(0)` 背后发生了什么

### 调用链（ref.ts:63）

```
ref(0)
  → createRef(0, false)      ref.ts:103
    → new RefImpl(0, false)  ref.ts:107
```

### `RefImpl` 的结构

```typescript
count = {
  _rawValue: 0,     // 原始值，用于 hasChanged 比较
  _value: 0,        // 实际存储值（若是对象会被 toReactive 包成 reactive）
  dep: new Dep(),   // 依赖桶，存"谁依赖了我"
  [IS_REF]: true    // 身份标记，isRef() 靠它判断
}
```

### `_value` 与 `_rawValue` 的区别

```typescript
this._rawValue = toRaw(value)    // 永远是原始值
this._value    = toReactive(value) // 如果 value 是对象，自动套 reactive
```

所以 `ref({ a: 1 })` 的 `.value` 其实是一个 `reactive` 对象，嵌套属性也能响应式。

---

## 二、读写时发生了什么

### 读：`count.value`（ref.ts:128）

```typescript
get value() {
  this.dep.track()   // 收集依赖
  return this._value
}
```

### 写：`count.value++`（ref.ts:141）

```typescript
set value(newValue) {
  if (hasChanged(newValue, oldValue)) {  // 值真的变了才触发，防无效更新
    this._rawValue = newValue
    this._value = toReactive(newValue)
    this.dep.trigger()                   // 通知所有依赖
  }
}
```

---

## 三、数据结构：`Dep` 和 `Link`（dep.ts）

### `Dep`：依赖桶

每个响应式数据都有一个 `Dep`，内部维护一条**双向链表**，链表节点是 `Link`：

```
Dep（count 的依赖桶）
  subs → Link ←→ Link ←→ Link
          ↓        ↓        ↓
       effect1  effect2  effect3
```

### `Link`：连接桥梁（dep.ts:32）

`Link` 同时存在于**两条双向链表**中：

```typescript
Link {
  sub: effect,      // 指向订阅者（effect）
  dep: Dep,         // 指向数据源
  version: number,  // 用于判断依赖是否还有效

  prevSub / nextSub  // 在 Dep 的链表里（横向）
  prevDep / nextDep  // 在 effect 的链表里（纵向）
}
```

用图表示 dep 和 effect 的多对多关系：

```
        dep1    dep2    dep3
         │       │       │
effectA─Link────Link────Link
         │               │
effectB─Link────────────Link
```

### 为什么用双向链表而不是 Set

Vue 3.4 之前用的是 `Set`，3.4 改为双向链表，原因如下：

**旧方案（Set）的问题：**

```typescript
// 旧实现
class Dep   { subs: Set<ReactiveEffect> }
class ReactiveEffect { deps: Set<Dep> }
```

每次 effect 重跑前必须清空再重建：

```
重跑前：for (dep of effect.deps) dep.subs.delete(effect)
        effect.deps.clear()
重跑中：dep.subs.add(effect) / effect.deps.add(dep)  ← 全部重新收集
```

问题：**每次更新都大量创建/销毁 Set 操作，GC 压力大。**

**新方案（双向链表）的优势：**

1. **`Link` 节点同时挂在两条链表上，零额外内存**

```
effect.deps 链（纵向）  deps → Link1 ↔ Link2 ↔ Link3
dep.subs 链（横向）     subs → Link1 ↔ Link4 ↔ Link5
```

同一个 `Link` 对象既是 effect 的 deps 节点，也是 dep 的 subs 节点。

2. **双向 → O(1) 删除中间节点**

```typescript
// 删除中间节点，单向链表需要从头找前驱 O(n)，双向直接拿 prev O(1)
link.prevDep!.nextDep = link.nextDep
link.nextDep!.prevDep = link.prevDep
```

3. **`version` 机制配合链表，复用节点不重建**

```
重跑前：所有 link.version 置为 -1      O(n) 打标记
重跑中：读到 dep → link.version 恢复   O(1) 复用旧 Link 对象
重跑后：清理 version===-1 的 link      O(1) 双向链表删除
```

不销毁重建 Link 对象，GC 友好。

**一句话：** 双向链表 O(1) 删除 + Link 双挂零内存 + version 复用不重建 = 比 Set 方案在频繁更新场景下性能更优。

---

## 四、`track()` 内部实现（dep.ts:108）

```typescript
track() {
  if (!activeSub || !shouldTrack) return  // 没有 effect 在跑，不收集

  let link = this.activeLink
  if (link === undefined || link.sub !== activeSub) {
    // 第一次读：新建 Link，建立 dep ↔ effect 双向关联
    link = this.activeLink = new Link(activeSub, this)

    // 追加到 effect 的 dep 链表尾部
    if (!activeSub.deps) {
      activeSub.deps = activeSub.depsTail = link
    } else {
      link.prevDep = activeSub.depsTail
      activeSub.depsTail!.nextDep = link
      activeSub.depsTail = link
    }

    addSub(link)  // 见下方 addSub 详解

  } else if (link.version === -1) {
    // 复用：上次运行就收集过，恢复版本号即可，不重建
    link.version = this.version
  }
}
```

### `version === -1` 机制（动态依赖收集）

每次 effect 重新运行前，把所有旧 link 的 `version` 置为 `-1`（标记"待确认"）。运行过程中读到哪个 dep 就把对应 link 恢复。运行结束后，version 还是 `-1` 的 link 说明这次没用到 → 自动清除。

```
// 场景：条件渲染中依赖会动态变化
if (flag.value) {
  console.log(a.value)  // 只有 flag=true 时才依赖 a
}
```

这保证了 effect 的依赖列表始终与实际访问一致，不会收集用不到的依赖。

### `addSub(link)` 详解（dep.ts:207）

`track()` 建立了 effect → dep 方向（effect.deps 链表），`addSub` 负责补上 dep → effect 方向（dep.subs 链表），合起来才是完整的双向关联。

**第一件事：订阅计数（第 208 行）**

```typescript
link.dep.sc++   // sc = subscriber count，记录 dep 当前有多少订阅者
```

**第二件事：处理 computed 懒激活（第 210-218 行）**

```typescript
const computed = link.dep.computed
if (computed && !link.dep.subs) {
  // 这个 dep 属于某个 computed，且刚获得第一个订阅者
  computed.flags |= EffectFlags.TRACKING | EffectFlags.DIRTY
  for (let l = computed.deps; l; l = l.nextDep) {
    addSub(l)   // 递归激活 computed 的所有上游 dep
  }
}
```

`computed` 是懒执行的，没有订阅者时不追踪。当第一个 effect 订阅了某个 computed，才递归激活其上游依赖链（computed 章节再深入）。

**第三件事：维护 dep.subs 双向链表（第 220-230 行）**

```typescript
const currentTail = link.dep.subs    // 当前链表尾节点
link.prevSub = currentTail           // 新节点 prev 指向旧尾
if (currentTail) currentTail.nextSub = link  // 旧尾 next 指向新节点
link.dep.subs = link                 // dep.subs 始终指向最新的尾节点
```

```
追加前：dep.subs → LinkA
追加后：dep.subs → LinkB ↔ LinkA
```

**两个方向的分工：**

```
track()  负责：effect ──deps──► Link   effect 知道自己依赖了哪些 dep
                                         用途：effect 重跑前清理旧依赖

addSub() 负责：dep ────subs──► Link   dep 知道谁订阅了自己
                                         用途：数据变化时通知所有订阅的 effect
```

同一个 `Link` 对象被两条链表共享，`link.sub = effect`，`link.dep = dep`，不是循环引用，是双向关联。

---

## 五、`trigger()` 内部实现（dep.ts:167）

```typescript
trigger() {
  this.version++      // dep 版本号自增
  globalVersion++     // 全局版本号自增（computed 快速判断是否过期用）
  this.notify()
}

notify() {
  startBatch()        // 开启批处理
  try {
    for (let link = this.subs; link; link = link.prevSub) {
      link.sub.notify()  // 通知每个订阅的 effect
    }
  } finally {
    endBatch()        // 批处理结束，统一调度执行
  }
}
```

### `startBatch` / `endBatch` 批处理

同一个同步任务里多次数据变化，不会立刻每次都重跑 effect，而是收集完所有变化后统一执行一次：

```typescript
count.value++   // trigger → startBatch，effect 标记 dirty，入队
name.value = 'x' // trigger → effect 已在队列，跳过重复入队
// 同步代码结束 → endBatch → 统一执行队列里的 effect
```

---

## 六、两套 `track/trigger` 的区别

| | 位置 | 调用方 | 原因 |
|--|--|--|--|
| `dep.track()` | Dep 类方法 | `ref` 直接调 | ref 每个实例有自己的 dep |
| `track(target, key)` | dep.ts 底部函数 | `reactive` 调 | 对象有多个 key，需从 targetMap 查找对应 dep |

`reactive` 用 `targetMap` 存储结构：

```
targetMap（WeakMap）
  └─ obj → Map {
               'name' → Dep,
               'age'  → Dep,
               ...
            }
```

`WeakMap` 的好处：对象被 GC 回收后，对应的依赖数据自动释放，不会内存泄漏。

---

## 七、完整响应式链路

```
const count = ref(0)
  → RefImpl { _value: 0, dep: Dep }

─── 渲染时（effect.run() 执行中）───

count.value  （读）
  → get value()
    → dep.track()
      → activeSub 不为空（effect 正在跑）
      → new Link(effect, dep)
      → effect.deps 链表追加
      → dep.subs 链表追加

─── 数据变化 ───

count.value++  （写）
  → set value()
    → hasChanged() ✓
    → dep.trigger()
      → dep.version++  globalVersion++
      → dep.notify()
        → startBatch()
        → 遍历 dep.subs，link.sub.notify()
          → effect 标记 dirty，scheduler() → queueJob(job)
        → endBatch() → 执行队列 → 重新渲染
```

---

## 暂跳过的部分

- `reactive()` / Proxy 的 get/set 拦截器（`baseHandlers.ts`）
- `computed()` 的懒计算与缓存机制
- `watch` / `watchEffect` 的实现
- `effect.ts` 中 `prepareDeps` / `cleanupDeps` 的完整逻辑
