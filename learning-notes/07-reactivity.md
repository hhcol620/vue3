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

## 六、`reactive` 与 Proxy

### `ref` vs `reactive` 拦截方式对比

两种响应式 API 用了完全不同的拦截机制：

| | `ref` | `reactive` |
|--|--|--|
| 拦截方式 | class getter/setter | Proxy |
| 适用场景 | 单值（number/string/...） | 对象/数组 |
| dep 存储 | RefImpl 实例上直接有 `dep` | `targetMap[target][key]` |
| 访问方式 | `.value` | 直接读属性 |

### `ref(对象/数组)` 内部也用了 `reactive`

`RefImpl` 构造时调用 `toReactive`（reactive.ts:430）：

```typescript
export const toReactive = <T>(value: T): T =>
  isObject(value) ? reactive(value) : value   // 是对象就包 reactive
```

所以：
- `ref(0)` → `_value = 0`（原始值）
- `ref({ a: 1 })` → `_value = reactive({ a: 1 })`（Proxy）
- `ref([1,2,3])` → `_value = reactive([1,2,3])`（Proxy）

**两层追踪（以 `ref([1,2,3])` 为例）：**

```
arr.value         → ref 的 dep.track()    追踪 .value 整体替换
arr.value[0]      → Proxy get 拦截        追踪数组第 0 项被读
arr.value[0] = 99 → Proxy set 拦截        触发数组第 0 项变化
arr.value = [4,5] → ref 的 dep.trigger()  触发 .value 被替换
```

### `reactive` 如何创建 Proxy（reactive.ts:296）

```typescript
const proxy = new Proxy(
  target,
  targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers,
)
proxyMap.set(target, proxy)   // 缓存，同一对象不重复创建
return proxy
```

`baseHandlers` 分两个实现类：

```
BaseReactiveHandler           公共 get 逻辑
  ├── MutableReactiveHandler  → reactive()  可读可写
  └── ReadonlyReactiveHandler → readonly()  只读
```

### `baseHandlers` 的 get 拦截（baseHandlers.ts:55）

```typescript
get(target, key, receiver) {
  // ...处理特殊 key（IS_REACTIVE / IS_READONLY / RAW 等）

  const res = Reflect.get(target, key, receiver)

  if (!isReadonly) {
    track(target, TrackOpTypes.GET, key)   // 收集依赖
  }

  if (isObject(res)) {
    return isReadonly ? readonly(res) : reactive(res)  // 懒代理嵌套对象
  }

  return res
}
```

**懒代理**：不在初始化时递归处理所有属性，只有真正访问到嵌套对象时才包 Proxy，性能更优。

---

## 七、为什么 Vue 3 用 Proxy 替代 `Object.defineProperty`

### `Object.defineProperty` 的三个根本缺陷

**1. 新增属性监听不到**
```javascript
const obj = reactive({ a: 1 })
obj.b = 2   // ❌ 监听不到，Vue 2 必须用 Vue.set(obj, 'b', 2)
```

**2. 数组下标和 length 监听不到**
```javascript
arr[0] = 99      // ❌ 监听不到
arr.length = 0   // ❌ 监听不到
// Vue 2 为此重写了 push/pop/splice 等 7 个方法，是 hack 而非原生支持
```

**3. 初始化必须递归全量处理**
```javascript
function observe(obj) {
  Object.keys(obj).forEach(key => {
    defineReactive(obj, key, obj[key])  // 每个 key 都要处理
    if (isObject(obj[key])) observe(obj[key])  // 深层递归，哪怕从不访问
  })
}
```

### Proxy 如何解决这三个问题

| 问题 | Proxy 方案 |
|------|-----------|
| 新增属性 | 拦截对象所有操作，新 key 的 set 也能捕获 |
| 数组下标/length | set 拦截原生支持，无需 hack |
| 初始化性能 | 懒代理，get 时才递归包子对象，不访问不处理 |

Proxy 还额外支持 `deleteProperty`、`has`、`ownKeys` 等拦截，能力远超 `defineProperty`。

**唯一代价**：不支持 IE。

---

## 八、`readonly` 的实现

### 核心：只换 handler，set/delete 拦截直接拒绝

`readonly()` 和 `reactive()` 走同一套 `createReactiveObject`，只是传入的 handler 不同：

```typescript
// ReadonlyReactiveHandler（baseHandlers.ts:225）
class ReadonlyReactiveHandler extends BaseReactiveHandler {
  constructor(isShallow = false) {
    super(true, isShallow)   // _isReadonly = true
  }

  set(target, key) {
    if (__DEV__) warn(`Set operation on key "${key}" failed: target is readonly.`)
    return true   // 拦截写入，静默失败（不实际执行，不 trigger）
  }

  deleteProperty(target, key) {
    if (__DEV__) warn(`Delete operation on key "${key}" failed: target is readonly.`)
    return true   // 拦截删除，同上
  }
}
```

### get 里的两处 readonly 判断

```typescript
// 1. readonly 不收集依赖（数据不会变，track 没意义）
if (!isReadonly) {
  track(target, TrackOpTypes.GET, key)
}

// 2. 嵌套对象也递归包成 readonly（深度只读）
if (isObject(res)) {
  return isReadonly ? readonly(res) : reactive(res)
}
```

### 对比

```
reactive(obj)  →  MutableReactiveHandler
  get: track()           读时收集依赖
  set: trigger()         写时通知更新

readonly(obj)  →  ReadonlyReactiveHandler
  get: 不 track          数据不会变，收集没意义
  set: 警告 + 拒绝       静默失败，不触发更新
  嵌套对象: readonly()   深层也只读
```

---

## 九、两套 `track/trigger` 的区别

| | 位置 | 调用方 | 原因 |
|--|--|--|--|
| `dep.track()` | Dep 类方法 | `ref` 直接调 | ref 每个实例有自己的 dep |
| `track(target, key)` | dep.ts 底部函数 | `reactive` 调 | 对象有多个 key，需从 targetMap 查找对应 dep |

`reactive` 用 `targetMap` 存储结构：

```
targetMap（WeakMap）
  └─ obj → Map { 'name' → Dep, 'age' → Dep, ... }
```

`WeakMap` 的好处：对象被 GC 回收后，对应的依赖数据自动释放，不会内存泄漏。

---

## 十、完整响应式链路

```
const count = ref(0)
  → RefImpl { _value: 0, dep: Dep }

─── 渲染时（effect.run() 执行中）───

count.value  （读）
  → get value() → dep.track()
      → activeSub 不为空 → new Link(effect, dep)
      → effect.deps 链表追加；dep.subs 链表追加

─── 数据变化 ───

count.value++  （写）
  → set value() → hasChanged() ✓
    → dep.trigger()
      → dep.version++  globalVersion++
      → dep.notify()
        → startBatch()
        → 遍历 dep.subs，link.sub.notify() → effect 入队
        → endBatch() → effect.trigger() → queueJob → 下一帧渲染
```

**Q：Vue 3 依赖收集为什么从 Set 改成双向链表？（Vue 3.4）**

> Set 方案每次 effect 重跑都要清空重建，GC 压力大。双向链表方案用 `version=-1` 标记旧依赖，重跑时原地复用 `Link` 对象，只清理真正失效的节点（O(1) 删除）。同一个 `Link` 节点同时挂在 effect.deps 和 dep.subs 两条链表上，零额外内存开销。

**Q：`watch` 和 `watchEffect` 的区别？底层如何实现？**

> 两者都基于 `ReactiveEffect`，走同一个 `baseWatch` 函数。区别在于：`watchEffect` 把用户函数直接作为 getter，立即执行并自动收集依赖，数据变化后直接重跑；`watch` 把 source 包装成 getter，首次只运行 getter 收集依赖而不触发 cb，数据变化后运行 getter 得到新值，通过 `hasChanged` 对比新旧值，真的变化才调用 cb 并传入 `(newVal, oldVal)`。

**Q：`startBatch/endBatch` 和 `queueJob` 分别解决什么问题？**

> `startBatch/endBatch` 用 `batchDepth` 引用计数解决 **computed 级联通知的顺序问题**——防止内层 endBatch 提前执行导致顺序错乱。`queueJob` 解决**多次赋值只渲染一次的问题**——调度器按 job.id 去重，同一组件无论触发多少次更新，渲染任务只在队列里存一份。

---

## 十一、`toRef` / `toRefs`

### `toRef` 的四种形态（ref.ts:482）

```typescript
toRef(existingRef)           // 已是 ref → 直接返回，不重复包装
toRef(() => props.foo)       // getter 函数 → GetterRefImpl（只读 ref）
toRef(reactiveObj, 'key')    // reactive 对象 + key → ObjectRefImpl（最常用）
toRef(1)                     // 普通值 → 等价于 ref(1)
```

### `ObjectRefImpl`：穿透代理（ref.ts:356）

```typescript
class ObjectRefImpl {
  get value() {
    return this._object[this._key]    // 读：穿透到原 reactive 对象
  }
  set value(newVal) {
    this._object[this._key] = newVal  // 写：穿透到原 reactive 对象
  }
  get dep() {
    return getDepFromReactive(this._raw, this._key)  // dep 也是原对象的
  }
}
```

**没有自己的 dep**，响应式完全由原 reactive 对象的 Proxy 保证，`ObjectRefImpl` 只是一层薄包装。

### `toRefs`：批量 toRef（ref.ts:345）

```typescript
function toRefs(object) {
  const ret = isArray(object) ? new Array(object.length) : {}
  for (const key in object) {
    ret[key] = propertyToRef(object, key)  // 每个 key 都 toRef 一遍
  }
  return ret
}
```

### 为什么需要 `toRefs`

```javascript
const state = reactive({ x: 1, y: 2 })

const { x, y } = state           // ❌ 解构丢失响应式，拿到普通值
const { x, y } = toRefs(state)   // ✅ 每个值都是 ref，响应式保留
// x.value === 1，修改 x.value 会同步到 state.x，反之亦然
```

在 `setup()` 中把 reactive 对象解构返回时必须用 `toRefs`，否则模板中失去响应式。

---

## 十二、`shallowReactive` vs `reactive`

### 区别的本质：一行代码

两者走完全相同的流程，只是 handler 构造时 `_isShallow` 不同。

get 拦截器里的这个判断决定了全部差异（baseHandlers.ts:116）：

```typescript
if (isShallow) {
  return res          // shallow：直接返回，不再递归包 Proxy
}
if (isObject(res)) {
  return reactive(res) // 普通 reactive：嵌套对象继续包 Proxy（深度响应式）
}
```

### 具体表现

```javascript
const state = shallowReactive({
  count: 1,           // ✅ 响应式（根层级）
  nested: { a: 1 },  // ❌ 非响应式（嵌套对象是原始对象，非 Proxy）
})

state.count = 2           // ✅ 触发更新
state.nested.a = 2        // ❌ 不触发更新
state.nested = { a: 99 }  // ✅ 触发更新（替换整个 nested 是根层级操作）
```

### 使用场景

大型列表、嵌套层级深但只需要追踪根层级属性时用 `shallowReactive`，避免深层递归代理的性能开销。

---

## 十三、面试常考问题

**Q：Vue 3 响应式原理是什么？**

> `ref` 通过 getter/setter 拦截读写。读时调 `dep.track()`，若当前有 `activeSub`（正在运行的 effect）就建立 `Link` 关联；写时调 `dep.trigger()`，遍历所有订阅的 effect 通知更新，最终通过调度器 `queueJob` 异步批量执行渲染。

**Q：Vue 3 为什么修改数据后不会立即触发多次渲染？**

> 两层去重：① `effect.notify()` 里的 `NOTIFIED` 标志位防止同一 effect 在一次 `endBatch` 执行前重复入队；② 调度器 `queueJob` 按 job.id 去重，同一组件的渲染任务只会在微任务队列里存在一份，下一个 tick 统一执行一次。

**Q：`ref` 和 `reactive` 的依赖收集有什么区别？**

> `ref` 每个实例自带一个 `dep`，直接调 `dep.track()`；`reactive` 用全局 `targetMap（WeakMap<对象, Map<key, Dep>>）` 存储，按对象和 key 分别维护各自的 dep，通过 Proxy 拦截 get/set 触发 `track/trigger`。`WeakMap` 保证对象被 GC 时依赖数据自动释放。

**Q：为什么 Vue 3 用 Proxy 替代 `Object.defineProperty`？**

> `Object.defineProperty` 只能拦截已有属性的读写，新增属性、数组下标/length 变更都无能为力，且初始化时必须递归全量处理所有属性。Proxy 代理整个对象的所有操作，天然支持新增属性和数组变更，采用懒代理策略（get 时才递归包子对象），初始化性能更优。唯一代价是不支持 IE。

**Q：`readonly` 是如何实现的？**

> 本质是换了一套 Proxy handler（`ReadonlyReactiveHandler`），`set` 和 `deleteProperty` 被覆盖为"开发环境警告 + 静默拒绝"，不调用 `trigger`；`get` 里不调 `track`（数据不会变，收集没意义）；访问嵌套对象时递归包 `readonly`，实现深度只读。

**Q：`toRef` 和 `toRefs` 的作用？**

> `toRef(reactiveObj, 'key')` 创建一个 `ObjectRefImpl`，读写都穿透到原 reactive 对象，自身没有独立 dep，响应式由原对象的 Proxy 保证。`toRefs` 是批量版本，对 reactive 对象每个 key 执行 `toRef`，返回普通对象。主要用途是解构 reactive 对象时保持响应式连接。

**Q：`shallowReactive` 和 `reactive` 的区别？**

> 区别只在 get 拦截器里的一行判断：`reactive` 访问嵌套对象时递归包 Proxy（深度响应式）；`shallowReactive` 设置 `_isShallow=true`，get 时直接返回原始值，嵌套对象不再代理。根层级属性变化可以响应，嵌套属性变化不响应。适合大型列表等只需追踪根层级的场景。

**Q：Vue 3 依赖收集为什么从 Set 改成双向链表？（Vue 3.4）**

> Set 方案每次 effect 重跑都要清空重建，GC 压力大。双向链表方案用 `version=-1` 标记旧依赖，重跑时原地复用 `Link` 对象，只清理真正失效的节点（O(1) 删除）。同一个 `Link` 节点同时挂在 effect.deps 和 dep.subs 两条链表上，零额外内存开销。

**Q：`startBatch/endBatch` 和 `queueJob` 分别解决什么问题？**

> `startBatch/endBatch` 用 `batchDepth` 引用计数解决 **computed 级联通知的顺序问题**——防止内层 endBatch 提前执行导致顺序错乱。`queueJob` 解决**多次赋值只渲染一次的问题**——调度器按 job.id 去重，同一组件无论触发多少次更新，渲染任务只在队列里存一份。

---

## 十四、`watch` / `watchEffect`

### 作用区别

```
watchEffect(() => { ... })
  → 立即执行，自动收集内部依赖，依赖变化重新执行
  → 不关心具体哪个值变了，只关心"要执行这段逻辑"

watch(source, (newVal, oldVal) => { ... })
  → 明确指定监听源，懒执行（默认不立即运行）
  → 能拿到新旧值，关心"具体哪个值变了、变成了什么"
```

### 底层实现：两者都走 `doWatch` → `baseWatch`（reactivity/watch.ts）

**第一步：把 source 统一包成 getter 函数（watch.ts:153）**

```typescript
if (isRef(source)) {
  getter = () => source.value          // ref → 读 .value
} else if (isReactive(source)) {
  getter = () => traverse(source)      // reactive → 深度遍历收集所有依赖
} else if (isFunction(source)) {
  if (cb) {
    getter = source                    // watch(fn, cb) → fn 是 getter
  } else {
    getter = () => source(cleanup)     // watchEffect → source 本身就是 getter
  }
}
```

**第二步：创建 `ReactiveEffect(getter)`（watch.ts:286）**

```typescript
effect = new ReactiveEffect(getter)
effect.scheduler = () => scheduler(job, false)
```

与组件渲染 effect 完全相同的机制，区别只在 getter 和 job 的内容。

**第三步：`job` 函数处理新旧值（watch.ts:233）**

```typescript
const job = () => {
  if (cb) {
    // watch：运行 getter 得到新值，值真的变了才触发 cb
    const newValue = effect.run()
    if (hasChanged(newValue, oldValue)) {
      cb(newValue, oldValue, onCleanup)
      oldValue = newValue              // 更新 oldValue 供下次对比
    }
  } else {
    // watchEffect：直接运行 getter（用户传入的函数）
    effect.run()
  }
}
```

**第四步：首次运行（watch.ts:312）**

```typescript
if (cb) {
  if (immediate) {
    job(true)              // watch + immediate：立即触发 cb
  } else {
    oldValue = effect.run()  // watch 默认：只跑 getter 收集依赖，不触发 cb
  }
} else {
  effect.run()             // watchEffect：立即运行
}
```

### flush 调度时机（apiWatch.ts）

```
flush: 'pre'（默认）  → DOM 更新前执行，用 queueJob
flush: 'post'        → DOM 更新后执行，用 queuePostRenderEffect
flush: 'sync'        → 数据变化立即同步执行，不进队列
```

`watchPostEffect` 和 `watchSyncEffect` 就是预设了 flush 的 `watchEffect` 快捷方式。

### 完整对比

| | `watchEffect` | `watch` |
|--|--|--|
| 依赖收集 | 自动（运行时收集） | 手动指定 source |
| 首次执行 | 立即 | 默认不执行（`immediate:true` 才执行）|
| 新旧值 | 无 | 有 `(newVal, oldVal)` |
| getter | source 本身 | 根据 source 类型包装成 getter |
| 底层 | `ReactiveEffect(source)` | `ReactiveEffect(getter)` + job 比较新旧值 |
| 适用场景 | 副作用同步（DOM 操作、网络请求） | 监听特定值变化，需要新旧值对比 |

---

## 暂跳过的部分

- `computed()` 的懒计算与缓存机制
- `effect.ts` 中 `prepareDeps` / `cleanupDeps` 的完整逻辑
- `collectionHandlers`（Map / Set / WeakMap / WeakSet 的响应式处理）
- 数组的特殊拦截（`arrayInstrumentations`）
