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

### `new Link(activeSub, dep)` 只是造节点，不接链表

```typescript
constructor(public sub: Subscriber, public dep: Dep) {
  this.version = dep.version
  this.nextDep = this.prevDep = this.nextSub = this.prevSub = undefined
  // 所有指针全是 undefined，没有接入任何链表
}
```

`public sub / dep` 只是存了两个引用，**`new Link` 本身不做任何链表操作**。
真正的链表接入分两步，在创建之后手动完成：

```
new Link()          → 造砖：空节点，指针全 undefined
track() 后续代码    → 砌墙①：接入 activeSub.deps 链表
addSub()            → 砌墙②：接入 dep.subs 链表
```

### 完整代码

```typescript
track() {
  if (!activeSub || !shouldTrack) return  // 没有 effect 在跑，不收集

  let link = this.activeLink
  if (link === undefined || link.sub !== activeSub) {

    // ① 造节点（指针全 undefined，只存 sub 和 dep 引用）
    link = this.activeLink = new Link(activeSub, this)

    // ② 手动接入 activeSub.deps 链表（effect 记录自己依赖了哪些 dep）
    if (!activeSub.deps) {
      activeSub.deps = activeSub.depsTail = link   // 第一个节点
    } else {
      link.prevDep = activeSub.depsTail            // 手动接指针
      activeSub.depsTail!.nextDep = link
      activeSub.depsTail = link                    // 更新尾指针
    }

    // ③ 手动接入 dep.subs 链表（dep 记录谁订阅了自己）
    addSub(link)

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

## 五、`trigger()` → `notify()` 完整通知链路

### `trigger()` / `notify()`（dep.ts:167）

```typescript
trigger() {
  this.version++      // dep 版本号自增
  globalVersion++     // 全局版本号自增（computed 快速判断是否过期用）
  this.notify()
}

notify() {
  startBatch()
  try {
    for (let link = this.subs; link; link = link.prevSub) {
      if (link.sub.notify()) {
        // 返回 true = 这是个 computed，需额外通知它自己的 dep
        ;(link.sub as ComputedRefImpl).dep.notify()
      }
    }
  } finally {
    endBatch()
  }
}
```

注意遍历方向是**从尾到头**（`link.prevSub`），`endBatch` 执行时按正序触发，保证顺序一致。

### `effect.notify()`（effect.ts:139）

```typescript
notify(): void {
  if (this.flags & EffectFlags.RUNNING && !(this.flags & ALLOW_RECURSE)) {
    return   // effect 正在运行中，防止自触发死循环
  }
  if (!(this.flags & EffectFlags.NOTIFIED)) {
    batch(this)  // 还没入队，加进去
  }
  // 已是 NOTIFIED 则跳过，防止同一 effect 重复入队
}
```

### `batch()`（effect.ts:240）

```typescript
function batch(sub: Subscriber): void {
  sub.flags |= EffectFlags.NOTIFIED   // 打标记：已在队列
  sub.next = batchedSub               // 用 next 串成单链表
  batchedSub = sub                    // batchedSub 始终指向链表头
}
// 结果：batchedSub → effect3 → effect2 → effect1 → undefined
```

### `endBatch()` 统一执行（effect.ts:262）

```typescript
export function endBatch(): void {
  if (--batchDepth > 0) return   // 还有外层 batch，不执行

  while (batchedSub) {
    let e = batchedSub
    batchedSub = undefined
    while (e) {
      const next = e.next
      e.next = undefined
      e.flags &= ~EffectFlags.NOTIFIED   // 清除标记
      if (e.flags & EffectFlags.ACTIVE) {
        e.trigger()   // → scheduler() → queueJob() → 下一帧渲染
      }
      e = next
    }
  }
}
```

`batchDepth` 是引用计数，只有减到 0 才真正执行，不需要额外判断开始和结束。

### `dep.notify()` 自带 `startBatch/endBatch` 的原因

不是为了合并多次赋值，而是为了处理 **computed 级联通知**：

```
count 变化 → dep1.notify()
  startBatch()   batchDepth=1
  通知 effectA 入队
  通知 computed C → C.dep.notify()   ← 嵌套触发
    startBatch()   batchDepth=2
    通知 effectB 入队
    endBatch()     batchDepth=1  ← 不执行，等外层
  endBatch()     batchDepth=0  ← 统一执行 effectA + effectB
```

没有这个机制，effectB 会在 effectA 入队前就执行，顺序乱掉。

### `count.value = 1; count.value = 2` 的完整执行过程

两次赋值各自触发独立的 `startBatch/endBatch`，**每次 `endBatch` 都会执行**，但最终只渲染一次：

```
count.value = 1
  dep.notify()
    startBatch()   batchDepth=1
    effect.notify() → batch(effect)  NOTIFIED=true，入队
    endBatch()     batchDepth=0 → 执行
      effect.flags &= ~NOTIFIED      ← 清除标记
      effect.trigger() → queueJob(job)  ✓ job 加入调度队列

count.value = 2
  dep.notify()
    startBatch()   batchDepth=1
    effect.notify() → NOTIFIED 已清除 → 再次 batch(effect) 入队
    endBatch()     batchDepth=0 → 执行
      effect.trigger() → queueJob(job)  ✗ job 已在队列，去重跳过

下一个微任务 tick：
  queue flush → job() 执行一次（count 此时 = 2）→ 渲染
```

### 批量合并的两层机制

| 层 | 机制 | 作用 |
|---|---|---|
| `startBatch/endBatch` | `batchDepth` 计数 | 合并 computed 级联通知，保证顺序 |
| `queueJob` 去重 | job.id 检查 | 合并多次数据变化为一次渲染 |

单次赋值时第一层没效果（batchDepth 自己 1→0），防止多次渲染靠的是第二层 `queueJob` 去重。

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

## 八、面试常考问题

**Q：Vue 3 响应式原理是什么？**

> `ref` 通过 getter/setter 拦截读写。读时调 `dep.track()`，若当前有 `activeSub`（正在运行的 effect）就建立 `Link` 关联；写时调 `dep.trigger()`，遍历所有订阅的 effect 通知更新，最终通过调度器 `queueJob` 异步批量执行渲染。

**Q：Vue 3 为什么修改数据后不会立即触发多次渲染？**

> 两层去重：① `effect.notify()` 里的 `NOTIFIED` 标志位防止同一 effect 在一次 `endBatch` 执行前重复入队；② 调度器 `queueJob` 按 job.id 去重，同一组件的渲染任务只会在微任务队列里存在一份，下一个 tick 统一执行一次。

**Q：`ref` 和 `reactive` 的依赖收集有什么区别？**

> `ref` 每个实例自带一个 `dep`，直接调 `dep.track()`；`reactive` 用全局 `targetMap（WeakMap<对象, Map<key, Dep>>）` 存储，按对象和 key 分别维护各自的 dep，通过 Proxy 拦截 get/set 触发 `track/trigger`。`WeakMap` 保证对象被 GC 时依赖数据自动释放。

**Q：Vue 3 依赖收集为什么从 Set 改成双向链表？（Vue 3.4）**

> Set 方案每次 effect 重跑都要清空重建，GC 压力大。双向链表方案用 `version=-1` 标记旧依赖，重跑时原地复用 `Link` 对象，只清理真正失效的节点（O(1) 删除）。同一个 `Link` 节点同时挂在 effect.deps 和 dep.subs 两条链表上，零额外内存开销。

**Q：`startBatch/endBatch` 和 `queueJob` 分别解决什么问题？**

> `startBatch/endBatch` 用 `batchDepth` 引用计数解决 **computed 级联通知的顺序问题**——防止内层 endBatch 提前执行导致顺序错乱。`queueJob` 解决**多次赋值只渲染一次的问题**——调度器按 job.id 去重，同一组件无论触发多少次更新，渲染任务只在队列里存一份。

---

## 暂跳过的部分

- `reactive()` / Proxy 的 get/set 拦截器（`baseHandlers.ts`）
- `computed()` 的懒计算与缓存机制
- `watch` / `watchEffect` 的实现
- `effect.ts` 中 `prepareDeps` / `cleanupDeps` 的完整逻辑
