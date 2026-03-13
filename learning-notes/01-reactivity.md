# Vue 3 响应式系统源码解析

> 源码路径：`packages/reactivity/src/`

---

## 核心文件一览

| 文件 | 职责 |
|------|------|
| `constants.ts` | 枚举：TrackOpTypes / TriggerOpTypes / ReactiveFlags |
| `dep.ts` | 依赖节点 Dep / Link，全局 track() / trigger() |
| `effect.ts` | ReactiveEffect 类，批处理，调度器 |
| `ref.ts` | ref() / RefImpl |
| `reactive.ts` | reactive()，Proxy 创建与缓存 |
| `baseHandlers.ts` | Proxy 的 get / set 拦截器 |
| `computed.ts` | computed() / ComputedRefImpl |
| `watch.ts` | watch / watchEffect |

---

## 推荐阅读顺序

```
constants.ts   → 认识标志位（词汇表）
    ↓
dep.ts         → 理解"依赖关系怎么存储"（数据结构）
    ↓
effect.ts      → 理解"副作用怎么运行、追踪、重跑"（执行引擎）
    ↓
baseHandlers.ts → 理解"Proxy 怎么对接 track/trigger"（接入层）
    ↓
ref.ts         → 最简单的响应式单元
    ↓
computed.ts    → 带缓存的派生值
    ↓
watch.ts       → 带回调的异步副作用
```

---

## 一、核心数据结构

### 1. 枚举常量（`constants.ts`）

```typescript
// 读操作类型
TrackOpTypes { GET, HAS, ITERATE }

// 写操作类型
TriggerOpTypes { SET, ADD, DELETE, CLEAR }

// 响应式对象内部标记
ReactiveFlags { SKIP, IS_REACTIVE, IS_READONLY, IS_SHALLOW, RAW, IS_REF }
```

### 2. Link（`dep.ts:32`）

一条"effect 订阅了 dep"的连接，用双向链表组织：

```typescript
class Link {
  version: number      // 与 dep.version 对齐；-1 表示本次 run 未访问（待清理）
  nextDep/prevDep      // effect 的 dep 链表（横向：一个 effect 的所有依赖）
  nextSub/prevSub      // dep 的 subscriber 链表（纵向：一个 dep 的所有订阅者）
  sub: Subscriber      // 所属 effect / computed
  dep: Dep             // 所属 dep
}
```

### 3. Dep（`dep.ts:67`）

代表一个响应式依赖点（一个 reactive 对象的某个 key，或一个 ref）：

```typescript
class Dep {
  version = 0          // 每次触发自增，订阅者靠版本号判断是否需要更新
  subs/subsHead        // subscriber 双向链表
  map/key              // 反查自己属于哪个 reactive 对象的哪个 key
  sc: number           // subscriber 计数
}
```

### 4. globalVersion（`dep.ts:19`）

```typescript
export let globalVersion = 0
```

- 全局单调递增，任意响应式数据变化时 +1
- computed 用它做**快速跳过缓存**的优化：若 `computed.globalVersion === globalVersion`，说明没有任何响应式变动，直接返回缓存值，不需要检查具体依赖

### 5. ReactiveEffect（`effect.ts:87`）

```typescript
class ReactiveEffect<T> implements Subscriber {
  deps?: Link          // 依赖链表头
  depsTail?: Link      // 依赖链表尾
  flags: EffectFlags   // ACTIVE | RUNNING | TRACKING | DIRTY | NOTIFIED ...
  scheduler?           // 自定义调度（watch 用这个实现异步刷新）
  fn: () => T          // 用户的副作用函数
}
```

### 6. EffectFlags（`effect.ts:41`）

```typescript
enum EffectFlags {
  ACTIVE      = 1 << 0,  // effect 存活
  RUNNING     = 1 << 1,  // 正在执行
  TRACKING    = 1 << 2,  // 正在追踪依赖
  NOTIFIED    = 1 << 3,  // 已被加入批处理队列
  DIRTY       = 1 << 4,  // 需要重新求值
  ALLOW_RECURSE = 1 << 5,
  PAUSED      = 1 << 6,
  EVALUATED   = 1 << 7,  // computed 已执行过一次
}
```

---

## 二、reactive() 工作原理

```
reactive(obj)                              reactive.ts:91
  └─ createReactiveObject(target, mutableHandlers)   reactive.ts:261
       └─ WeakMap 缓存检查（避免重复代理）
       └─ new Proxy(target, mutableHandlers)
```

### Proxy 拦截器（`baseHandlers.ts`）

**读操作（get）**`baseHandlers.ts:55`
```
proxy.foo 被读取
  └─ Reflect.get(target, key)        取原始值
  └─ track(target, GET, key)         dep.ts:262  追踪依赖
  └─ 若值是对象 → reactive(res)       深层响应化（懒递归）
```

**写操作（set）**`baseHandlers.ts:142`
```
proxy.foo = val
  └─ Reflect.set(target, key, value) 真正写入
  └─ trigger(target, SET, key)       dep.ts:294  触发更新
```

---

## 三、ref() 工作原理

`ref.ts:113` — RefImpl

```typescript
class RefImpl {
  dep = new Dep()          // 每个 ref 有自己专属的 dep

  get value() {
    this.dep.track()       // 追踪：谁在读我
    return this._value
  }

  set value(newVal) {
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = toReactive(newVal)   // 若是对象，自动 reactive 化
      this.dep.trigger()   // 触发：通知所有订阅者
    }
  }
}
```

**对比 reactive**：
- `reactive` → Proxy 拦截任意 key 的读写
- `ref` → 只有 `.value` 一个入口，class 直接处理，不需要 Proxy

---

## 四、依赖追踪全链路（track 方向）

**时机**：`effect.run()` 执行期间，`activeSub` 被设为当前 effect。

```
effect.run()                           effect.ts:151
  activeSub = this                     标记"当前正在运行的 effect"
  prepareDeps(this)                    把所有 link.version 置为 -1（标记待验证）
  this.fn()                            执行用户代码（内部触发 get）
    └─ proxy.foo 被读取
         └─ baseHandlers.get()
              └─ track(target, GET, key)       dep.ts:262
                   └─ 找到/创建 targetMap[target][key] = dep
                   └─ dep.track()             dep.ts:108
                        └─ link = new Link(activeSub, dep)
                        └─ 插入 effect.deps 链表尾部
                        └─ addSub(link) → 插入 dep.subs 链表
  cleanupDeps(this)                    删除 version 仍为 -1 的 Link（本次未访问的旧依赖）
  activeSub = prevEffect
```

**关键设计**：每次 run 前先把所有 `link.version` 置 -1，run 后扫描清理，**自动处理动态依赖变化**（如条件分支中的响应式访问）。

---

## 五、依赖触发全链路（trigger 方向）

```
ref.value = newVal  /  proxy.foo = newVal
  └─ dep.trigger()                       dep.ts:167
       dep.version++
       globalVersion++
       dep.notify()                      dep.ts:173
         └─ startBatch()                 batchDepth++，开始批处理
         └─ for each link in dep.subs:
              link.sub.notify()
                └─ ReactiveEffect.notify()     effect.ts:139
                     batch(this)               加入 batchedSub 队列
                └─ ComputedRefImpl.notify()    computed.ts:117
                     flags |= DIRTY            标记脏，不立即重算
                     batch(this, isComputed=true)
                     return true              通知上游 dep 也需触发
         └─ endBatch()
              先处理 batchedComputed（重置 NOTIFIED 标志）
              再处理 batchedSub：
                effect.trigger()              effect.ts:195
                  有 scheduler → scheduler()
                  无 scheduler → runIfDirty()
                       isDirty() → run()      fn() 重新执行
```

---

## 六、computed() 懒求值与缓存

computed 同时扮演两个角色：
- 作为 **Subscriber**：订阅它依赖的 reactive/ref
- 作为 **Dep 持有者**：被读取它的 effect 订阅

`computed.ts:47` — ComputedRefImpl

```
computed.value 被访问
  └─ get value()                         computed.ts:131
       this.dep.track()                  让外层 effect 订阅 computed
       refreshComputed(this)             effect.ts:365
         ↓
         快速跳过：globalVersion 未变 → 直接返回缓存
         ↓
         isDirty 检查：遍历 computed 的 deps，看版本是否落后
         ↓
         若需重算：
           activeSub = computed           computed 本身成为 activeSub
           value = computed.fn()          执行 getter，内部的读操作追踪到 computed
           cleanupDeps(computed)
           若值变化：dep.version++        通知依赖这个 computed 的外层 effect

computed 依赖变化时：
  dep.trigger() → computed.notify()
    flags |= DIRTY                        标记脏，不立即重算
    等有人读 computed.value 才真正重算（Pull 模式）
```

---

## 七、watch / watchEffect

`watch.ts:120`

```
watch(source, callback, options)
  1. 构造 getter：
     - ref         →  () => source.value
     - reactive    →  () => traverse(source)   递归访问所有属性以追踪
     - function    →  source 本身

  2. new ReactiveEffect(getter)

  3. effect.scheduler = job
     job = () => {
       newVal = effect.run()                   重新求值（同时更新依赖）
       if (hasChanged(newVal, oldVal)) {
         callback(newVal, oldVal, onCleanup)
       }
     }

  4. 初次运行：
     - immediate → job()            立即触发回调
     - 否则      → effect.run()     只追踪依赖，不触发回调

触发时：
  响应式数据变 → dep.trigger() → effect.notify() → endBatch()
    → effect.trigger() → effect.scheduler() → job()
    → effect.run() 重新执行 getter → callback(newVal, oldVal)
```

**本质**：watch = 带自定义 scheduler 的 effect。普通 effect 变了立刻重跑，watch 变了只把 job 推进队列，等时机合适再调 callback。

---

## 八、完整调用关系图

```
┌──────────────── 读（追踪阶段）─────────────────┐
│                                                │
│  effect.run() / refreshComputed()             │
│    activeSub = effect/computed                 │
│    this.fn() ──► proxy.get / ref.value.get     │
│                      │                        │
│             baseHandlers.get / RefImpl.get     │
│                      │                        │
│                 track(target, key)  dep.ts:262 │
│                      │                        │
│                 dep.track()         dep.ts:108 │
│                 new Link(sub, dep)             │
│                 加入 sub.deps 链表              │
│                 加入 dep.subs 链表              │
└────────────────────────────────────────────────┘

┌──────────────── 写（触发阶段）─────────────────┐
│                                                │
│  proxy.set / ref.value.set                     │
│    baseHandlers.set / RefImpl.set              │
│       │                                       │
│  trigger() / dep.trigger()  dep.ts:294/167     │
│       │                                       │
│  dep.notify()  dep.ts:173                      │
│    startBatch()                                │
│    for link in dep.subs: link.sub.notify()     │
│      ReactiveEffect → batch(this)              │
│      ComputedRefImpl → DIRTY + batch(isComp)   │
│    endBatch()                                  │
│      computed 重置状态                          │
│      effect.trigger()                          │
│        → scheduler()  (watch/组件渲染)          │
│        → runIfDirty() (watchEffect/同步effect) │
│             isDirty() → run() → fn() 重新执行   │
└────────────────────────────────────────────────┘
```
