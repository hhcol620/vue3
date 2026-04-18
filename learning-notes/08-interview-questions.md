# Vue 3 高频面试题

> 面试题按模块分类，每题都给出核心考点和简洁答案，适合快速复习。

---

## 一、响应式系统

### 1. Vue 3 响应式原理是什么？和 Vue 2 有什么区别？

**核心考点**：Proxy vs Object.defineProperty

| | Vue 2 | Vue 3 |
|---|---|---|
| 实现 | `Object.defineProperty` | `Proxy` |
| 新增属性 | 需要 `Vue.set` | 自动响应 |
| 数组变化 | 重写了 7 个方法 | 自动响应 |
| 性能 | 初始化递归遍历全部属性 | 惰性（按需代理） |
| 嵌套对象 | 初始化时深度遍历 | 访问时才深度代理 |

---

#### 高分回答模板（分层递进）

**第一层（60分）：说出核心替换**

> Vue 3 基于 `Proxy` 实现响应式，通过 `get` 拦截做依赖收集（track），通过 `set` 拦截做触发更新（trigger），解决了 Vue 2 新增属性和数组下标无法响应的问题。

**第二层（80分）：说清数据结构 + 完整链路**

依赖存储结构：

```
WeakMap<target, Map<key, Set<ReactiveEffect>>>
  │
  └── { count: 0, name: 'Vue' }   ← target（被代理的原始对象）
        ├── 'count' → Set{ renderEffect }   ← key = 访问的属性名
        └── 'name'  → Set{}
```

- **target**：被 `reactive()` 包裹的**原始对象**（Proxy handler 第一个参数天然就是它）
- **key**：被访问的**属性名**，如 `'count'`
- **effect**：对"需要重新执行的函数"的封装（见下方详解）

完整链路：

```
组件渲染 → 创建 ReactiveEffect(renderFn)
  → effect.run() 设置 activeEffect = 当前 effect
  → renderFn 执行，访问响应式属性
  → get 拦截 → track(target, key)
  → 将 activeEffect 存入 targetMap[target][key] 的 dep 集合

属性变化 → set 拦截 → trigger(target, key)
  → 取出 dep 集合，遍历执行所有 effect
  → 组件 effect 的 scheduler 将更新推入异步队列
  → nextTick 后批量刷新，重新执行 renderFn
```

**第三层（95分）：说出惰性代理 + computed 推拉模型**

```ts
// 惰性深层代理：不初始化时递归，访问时才深度代理
get(target, key) {
  const res = Reflect.get(target, key)
  track(target, key)
  if (isObject(res)) return reactive(res) // 按需深代理
  return res
}
```

computed 推拉结合：依赖变化时只标脏（`_dirty = true`），不立即重算；访问 `.value` 时才真正执行 getter。

---

#### effect 是什么？

effect = **对"需要重新执行的函数"的包装对象**，不是函数本身，而是带依赖追踪能力的容器：

```ts
class ReactiveEffect {
  fn          // 被包装的函数（renderFn / getter / watchCallback）
  deps        // 该 effect 订阅了哪些 dep（反向引用）
  scheduler   // 触发时走调度而非直接重跑（组件更新推队列、computed 标脏）
  active      // stop() 后为 false，不再响应
}
```

三种常见 effect：

| 场景 | fn 是什么 | scheduler 行为 |
|---|---|---|
| 组件渲染 | renderFn | 推入异步更新队列 |
| `computed` | getter | 只标脏，不立即重跑 |
| `watchEffect` | 用户回调 | 推入异步队列 |

一句话：**dep 存 effect，effect 存 fn，trigger 找到 effect，scheduler 决定怎么重跑 fn。**

---

### 2. ref 和 reactive 的区别？

| | `ref` | `reactive` |
|---|---|---|
| 适用类型 | 任意类型（基本类型/对象） | 只能是对象/数组 |
| 访问方式 | `.value` | 直接访问 |
| 解构 | 解构会丢失响应性（需 `toRefs`） | 解构会丢失响应性（需 `toRefs`） |
| 原理 | 对象包裹 + getter/setter | Proxy |

```ts
const count = ref(0)
count.value++  // 需要 .value

const state = reactive({ count: 0 })
state.count++  // 直接访问
```

**面试追问**：模板里 ref 为什么不需要 `.value`？

> 模板编译时会自动进行 ref 解包（`proxyRefs`）。

---

### 3. computed 和 watch 的区别？

| | `computed` | `watch` |
|---|---|---|
| 目的 | 派生状态 | 执行副作用 |
| 缓存 | 有（依赖不变不重算） | 无 |
| 返回值 | 有 | 无 |
| 懒执行 | 默认是（首次访问才计算） | 默认否（依赖变化即执行） |
| 异步 | 不适合 | 适合 |

**computed 缓存原理**：通过 `_dirty` 标志位，依赖未变时直接返回缓存值。

---

### 4. watchEffect 和 watch 的区别？

```ts
// watch：显式声明依赖，有 oldValue/newValue
watch(source, (newVal, oldVal) => { ... })

// watchEffect：自动收集依赖，立即执行
watchEffect(() => {
  console.log(count.value) // 自动追踪 count
})
```

- `watchEffect` 立即执行，`watch` 默认懒执行（`immediate: true` 可改）
- `watchEffect` 无法获取旧值
- `watchEffect` 更适合有多个响应式依赖的副作用

---

### 5. Vue 3 如何实现依赖收集？

**核心链路**：`effect → track → dep → trigger`

```
访问响应式属性
  → get 拦截器
    → track(target, key)
      → 当前 activeEffect 订阅到 dep
        → 属性变化时 set 拦截
          → trigger → 执行所有订阅的 effect
```

Vue 3.4+ 使用 `WeakMap<target, Map<key, Set<effect>>>` 存储依赖关系。

---

## 二、组件系统

### 6. Vue 3 生命周期有哪些变化？

| Vue 2 | Vue 3 Composition API |
|---|---|
| `beforeCreate` / `created` | ❌（setup 本身替代） |
| `beforeMount` | `onBeforeMount` |
| `mounted` | `onMounted` |
| `beforeUpdate` | `onBeforeUpdate` |
| `updated` | `onUpdated` |
| `beforeDestroy` | `onBeforeUnmount` |
| `destroyed` | `onUnmounted` |

**执行顺序**：`setup` → `onBeforeMount` → 子组件挂载 → `onMounted`

---

### 7. setup 函数的执行时机和作用？

- 执行时机：组件实例创建后，`beforeCreate` / `created` 之前
- 没有 `this`（组件实例尚未创建完成）
- 接收 `(props, context)` 参数，`context` 包含 `attrs`、`slots`、`emit`
- 返回的对象会暴露给模板

```ts
setup(props, { attrs, slots, emit, expose }) {
  const count = ref(0)
  return { count } // 暴露给模板
}
```

---

### 8. defineProps / defineEmits 是什么？

编译器宏（Compiler Macros），只能在 `<script setup>` 中使用，**无需 import**。

```vue
<script setup>
const props = defineProps<{ title: string; count?: number }>()
const emit = defineEmits<{ (e: 'update', val: number): void }>()
</script>
```

编译后会转换为对应的 `props` / `emits` 选项，运行时不存在这些函数。

---

### 9. Vue 3 中父子组件通信方式？

| 方式 | 适用场景 |
|---|---|
| `props` / `emit` | 父传子 / 子传父 |
| `v-model` | 双向绑定（本质是 props + emit） |
| `provide` / `inject` | 跨层级祖孙传递 |
| `expose` / `ref` | 父调用子方法 |
| `attrs` | 未声明的 props 透传 |
| 全局状态（Pinia） | 兄弟/复杂场景 |

---

### 10. v-model 在 Vue 3 中的变化？

```vue
<!-- Vue 3：默认 modelValue + update:modelValue -->
<MyInput v-model="msg" />
<!-- 等价于 -->
<MyInput :modelValue="msg" @update:modelValue="msg = $event" />

<!-- Vue 3 支持多个 v-model -->
<MyForm v-model:title="title" v-model:content="content" />
```

Vue 2 中 `.sync` 修饰符被废弃，统一用 `v-model:propName` 替代。

---

## 三、渲染机制

### 11. 虚拟 DOM 是什么？Vue 3 有哪些优化？

虚拟 DOM 是用 JS 对象描述真实 DOM 的树形结构，通过 diff 算法最小化真实 DOM 操作。

**Vue 3 编译优化（静态提升 + 补丁标志）**：

```ts
// 静态节点提升（hoistStatic）：静态节点只创建一次
const _hoisted_1 = createElementVNode("div", null, "静态内容")

// 补丁标志（PatchFlag）：只对动态内容做 diff
createElementVNode("p", null, count.value, 1 /* TEXT */)
//                                           ↑ 只更新文本，跳过属性比较
```

| 优化手段 | 作用 |
|---|---|
| 静态提升 | 静态节点不参与更新 diff |
| 补丁标志 | 精确标记动态类型，跳过无关比较 |
| 块树（Block Tree） | 扁平化动态子节点，O(n) 变 O(动态节点数) |
| 静态属性提升 | 静态 props 对象复用 |

---

### 12. Vue 3 diff 算法和 Vue 2 有什么区别？

| | Vue 2 | Vue 3 |
|---|---|---|
| 算法 | 双端对比 | 双端对比 + 最长递增子序列 |
| 优化 | 无 | 结合 PatchFlag 跳过静态节点 |
| 复杂度 | O(n) | O(n)，但常数更小 |

**最长递增子序列（LIS）** 的作用：确定哪些节点不需要移动，减少 DOM 操作。

---

### 13. key 的作用是什么？为什么不推荐用 index 作 key？

- `key` 是 diff 算法识别节点身份的标识
- 相同 key 的节点会复用，不同 key 的节点会销毁重建
- 用 `index` 作 key：列表增删时 key 与节点对应关系会错位，导致错误复用（输入框内容错乱等）

---

## 四、编译器

### 14. 模板编译的流程是什么？

```
template 字符串
  → parse（词法 + 语法分析）→ AST
  → transform（AST 转换，应用各种插件）→ 优化后的 AST
  → generate（代码生成）→ render 函数字符串
```

最终 render 函数在运行时执行，生成 VNode 树。

---

### 15. `<script setup>` 相比 `<script>` 有什么优势？

- 顶层变量/函数自动暴露给模板，无需 `return`
- 可以使用编译器宏（`defineProps`、`defineEmits` 等）
- 性能更好：编译后更小的运行时代码
- 更好的 TypeScript 支持

---

## 五、性能与最佳实践

### 16. Vue 3 性能比 Vue 2 好在哪里？

1. **响应式**：Proxy 惰性代理，不再初始化时深度遍历
2. **编译优化**：PatchFlag + Block Tree，精准 diff
3. **Tree-shaking**：API 按需引入，打包体积更小
4. **Fragment**：组件支持多根节点，减少无意义包裹元素
5. **Teleport / Suspense**：原生支持，避免额外封装

---

### 17. 如何减少不必要的组件重渲染？

```ts
// 1. 用 v-memo 缓存列表项
<div v-memo="[item.id, item.selected]">...</div>

// 2. 用 shallowRef / shallowReactive 避免深层响应
const list = shallowRef([...])

// 3. 子组件用 defineProps 声明精确 props，避免接收多余数据

// 4. 使用 computed 缓存派生数据

// 5. 大列表用虚拟滚动（vue-virtual-scroller）
```

---

### 18. Teleport 是什么？使用场景？

将组件的 DOM 渲染到指定的 DOM 节点（脱离组件树位置）。

```vue
<Teleport to="body">
  <Modal />  <!-- Modal 的 DOM 挂到 body 下，但逻辑仍在组件树中 -->
</Teleport>
```

**典型场景**：弹窗、全屏 loading、Toast 通知（避免被父级 `overflow:hidden` 或 `z-index` 截断）。

---

### 19. Suspense 是什么？

处理异步组件/异步 setup 的加载状态：

```vue
<Suspense>
  <template #default><AsyncComponent /></template>
  <template #fallback><Loading /></template>
</Suspense>
```

当 `AsyncComponent` 的 `setup` 是 `async` 时，`fallback` 内容会先展示。

---

## 六、Composition API 设计

### 20. Composition API 相比 Options API 的优势？

| | Options API | Composition API |
|---|---|---|
| 代码组织 | 按选项类型（data/methods/computed...） | 按功能/关注点 |
| 逻辑复用 | Mixin（有命名冲突和来源不清问题） | Composable 函数（清晰） |
| TypeScript | 不友好 | 原生支持 |
| 代码复用 | 弱 | 强（composable 任意组合） |

---

### 21. 什么是 Composable（组合式函数）？如何设计？

可复用的有状态逻辑单元，约定以 `use` 开头：

```ts
// useCounter.ts
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const increment = () => count.value++
  const decrement = () => count.value--
  return { count, increment, decrement }
}

// 组件中使用
const { count, increment } = useCounter(10)
```

**设计原则**：
- 只在 `setup` 或 Composable 中调用
- 返回 ref（保持响应性），而不是解包后的原始值
- 命名以 `use` 开头

---

### 22. provide / inject 的响应性如何保证？

```ts
// 父组件
const count = ref(0)
provide('count', count)  // 传递 ref，保持响应性

// 子组件
const count = inject('count')  // 接收的是同一个 ref
```

**注意**：传递 reactive 对象的属性时需要用 `toRef` 包裹，否则会丢失响应性。

---

## 七、路由与状态（概要）

### 23. Vue Router 4 的变化？

- `createRouter` 代替 `new VueRouter`
- `history` 选项：`createWebHistory` / `createWebHashHistory`
- 组合式 API：`useRouter()` / `useRoute()`
- 动态路由增删：`router.addRoute()` / `router.removeRoute()`

### 24. Pinia 相比 Vuex 的优势？

- 无需 mutations（直接修改 state）
- 原生 TypeScript 支持
- 无嵌套模块（每个 store 独立）
- 支持 Composition API 风格定义
- 体积更小（~1KB）

---

## 快速记忆清单

```
响应式：Proxy > defineProperty，惰性代理，track/trigger
ref vs reactive：.value / Proxy，toRefs 解构
computed：缓存 + _dirty，watch：副作用 + oldVal
生命周期：setup 最早，无 this
diff：PatchFlag 跳静态，LIS 减移动
编译：parse → transform → generate
性能：Tree-shaking + Block Tree + 静态提升
```

---

# Vue 3 高级面试题（Senior 级别）

> 考察对源码机制、架构设计、工程实践的深度理解。

---

## 一、响应式系统深度

### A1. effect / ReactiveEffect 的内部机制？

Vue 3 响应式的最小单元是 `ReactiveEffect`，`watch`、`computed`、`watchEffect`、组件渲染都基于它。

```ts
class ReactiveEffect {
  deps: Dep[] = []          // 当前 effect 订阅的所有依赖
  _trackId = 0              // 去重 track 用（Vue 3.4+）
  dirty = true              // computed 的缓存标志

  run() {
    activeEffect = this     // 设置全局当前 effect
    return this.fn()        // 执行 fn，触发 get → track → 收集 this
  }

  notify() {
    // 依赖变化时被调用
    this.scheduler ? this.scheduler() : this.run()
  }
}
```

**关键**：`activeEffect` 是全局单例，`track` 时将它记录到 dep，`trigger` 时遍历 dep 通知所有 effect。

---

### A2. computed 的懒执行和缓存如何实现？

```ts
class ComputedRefImpl {
  _dirty = true   // true = 需要重新计算
  _value: T

  constructor(getter) {
    this.effect = new ReactiveEffect(getter, () => {
      // scheduler：依赖变化时不立即重算，只标脏
      if (!this._dirty) {
        this._dirty = true
        triggerRefValue(this) // 通知订阅了 computed 的 effect
      }
    })
  }

  get value() {
    trackRefValue(this)     // computed 本身也是依赖
    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()  // 重新计算
    }
    return this._value
  }
}
```

**关键**：依赖变化 → `scheduler` 标脏（不重算）→ 访问 `.value` 时才重算。这是"推拉结合"模型。

---

### A3. Vue 3.4 的响应式重构做了什么优化？

1. **双向链表依赖追踪**（`version` + `Link` 节点）替代原来的 `Set<effect>`，减少内存分配
2. **`_trackId` 去重**：同一 effect 内多次访问同一属性只 track 一次，避免重复订阅
3. **`version` 计数**：trigger 时递增 version，effect 通过比较 version 判断是否需要重算，减少无效执行

---

### A4. effectScope 是什么？解决什么问题？

管理一组 effect 的生命周期，统一停止。

```ts
const scope = effectScope()
scope.run(() => {
  const count = ref(0)
  watchEffect(() => console.log(count.value))
  watch(count, () => { ... })
  // 以上所有 effect 都属于这个 scope
})

scope.stop() // 一次性停止所有 effect，防止内存泄漏
```

**使用场景**：Pinia store、可复用逻辑库（如 VueUse）内部都用它管理 effect 生命周期。

---

### A5. shallowRef / shallowReactive / markRaw / toRaw 的使用场景？

| API | 作用 | 场景 |
|---|---|---|
| `shallowRef` | 只有 `.value` 本身是响应式，内部不深度代理 | 大数组/图表数据，手动控制更新 |
| `shallowReactive` | 只有第一层是响应式 | 确定只用第一层的对象 |
| `markRaw` | 标记对象永不被代理 | 第三方类实例（地图/echarts）、大型不变数据 |
| `toRaw` | 获取 Proxy 背后的原始对象 | 传给非响应式库、做深比较 |

```ts
// markRaw 避免误代理第三方实例
const chart = markRaw(new ECharts(el))
state.chart = chart  // 不会被 reactive 代理
```

---

### A6. Composable 中的内存泄漏如何避免？

```ts
// 错误：在 composable 里监听全局事件，但不清理
export function useBadScroll() {
  window.addEventListener('scroll', handler) // 组件卸载后还在监听！
}

// 正确：在 onUnmounted 中清理
export function useScroll() {
  onMounted(() => window.addEventListener('scroll', handler))
  onUnmounted(() => window.removeEventListener('scroll', handler))
}

// 更好：用 watch/watchEffect 的返回值
export function useCount() {
  const stop = watchEffect(() => { ... })
  onUnmounted(stop) // 或者依赖 effectScope 自动管理
}
```

---

## 二、渲染器与 VNode 深度

### A7. VNode 的 patchFlag 有哪些？各自含义？

```ts
export const enum PatchFlags {
  TEXT           = 1,       // 动态文本内容
  CLASS          = 2,       // 动态 class
  STYLE          = 4,       // 动态 style
  PROPS          = 8,       // 动态 props（需配合 dynamicProps）
  FULL_PROPS     = 16,      // 有 key 变化的 props（如 v-bind 无参）
  NEED_HYDRATION = 32,      // 需要 SSR hydration
  STABLE_FRAGMENT = 64,     // 稳定 fragment（子节点顺序不变）
  KEYED_FRAGMENT  = 128,    // 带 key 的 fragment
  UNKEYED_FRAGMENT = 256,   // 不带 key 的 fragment
  NEED_PATCH      = 512,    // 需要 patch（ref、生命周期等）
  DYNAMIC_SLOTS   = 1024,   // 动态 slot
  HOISTED         = -1,     // 静态提升节点，永不 diff
  BAIL            = -2,     // 退出优化模式，完整 diff
}
```

**意义**：patch 时只处理对应的动态部分，其余跳过。

---

### A8. Block Tree 的核心思想？

**问题**：传统 vdom diff 需要遍历整棵树，大部分是静态节点，浪费性能。

**解法**：Block 节点（如根节点、`v-if`/`v-for` 容器）收集其内部所有**动态子孙节点**到 `dynamicChildren` 数组。

```
Block (root)
  staticDiv              ← 不收集，跳过
  dynamicSpan (TEXT)     ← 收集到 dynamicChildren[0]
  staticSection          ← 不收集
    dynamicP (CLASS)     ← 收集到 dynamicChildren[1]
```

diff 时直接对比 `dynamicChildren`，**完全跳过静态节点**，复杂度从 O(全部节点) 降到 O(动态节点)。

---

### A9. 异步组件 + Suspense 的实现原理？

```ts
const AsyncComp = defineAsyncComponent(() => import('./Comp.vue'))
```

内部机制：
1. 组件 `setup` 返回 Promise 时，触发最近祖先 `Suspense` 的挂起流程
2. Suspense 渲染 `fallback` 插槽内容
3. Promise resolve 后，Suspense 切换为 `default` 插槽内容（真实组件）
4. 通过 `SSuspense` 内部的 `pendingBranch` / `activeBranch` 管理两个分支的切换

---

### A10. 自定义渲染器的实现方式？

```ts
import { createRenderer } from '@vue/runtime-core'

const { createApp } = createRenderer({
  // 宿主环境的 DOM 操作实现
  createElement(type) { return { type, children: [], props: {} } },
  insert(child, parent) { parent.children.push(child) },
  patchProp(el, key, prevVal, nextVal) { el.props[key] = nextVal },
  // ... 其他平台操作
})
```

**意义**：runtime-core 完全平台无关，runtime-dom 只是一种具体实现。小程序、Canvas、Native 渲染都基于此扩展。

---

## 三、编译器深度

### A11. 模板编译的 transform 阶段做了什么？

transform 是编译优化的核心，通过插件机制对 AST 做二次处理：

| Transform | 作用 |
|---|---|
| `transformIf` | 将 `v-if/v-else` 转为条件分支，标记 Block |
| `transformFor` | 将 `v-for` 转为 `renderList`，标记 Block |
| `transformElement` | 分析 props，标记 PatchFlag、收集 dynamicProps |
| `transformText` | 合并相邻文本/插值节点 |
| `hoistStatic` | 识别静态节点，提升到模块顶层 |
| `transformSlot` | 处理 slot 编译 |

---

### A12. v-model 是如何被编译的？

```vue
<input v-model="name" />
```

编译结果：
```ts
createVNode("input", {
  value: _ctx.name,
  "onUpdate:modelValue": $event => (_ctx.name = $event)
}, null, 8 /* PROPS */, ["value", "onUpdate:modelValue"])
```

自定义组件 `v-model`：
```ts
// <MyInput v-model="name" /> 编译为：
createVNode(MyInput, {
  modelValue: name,
  "onUpdate:modelValue": $event => (name = $event)
})
```

---

### A13. slot 的编译与运行时机制？

```vue
<!-- 父组件 -->
<Child>
  <template #header="{ title }">{{ title }}</template>
</Child>
```

编译后，slot 变成父组件传给子组件的**函数**：

```ts
// 父组件 render
createVNode(Child, null, {
  header: ({ title }) => createTextVNode(title),
  _: 1 // STABLE
})

// 子组件 render 时调用
renderSlot($slots, 'header', { title: 'Hello' })
```

**本质**：slot 是延迟执行的函数，在子组件渲染时调用，作用域在父组件（这就是"作用域插槽"的原理）。

---

## 四、架构与工程实践

### A14. Vue 3 Tree-shaking 是如何实现的？

Vue 3 的所有公共 API（`ref`、`computed`、`watch` 等）都是**具名导出**，而非挂在 Vue 实例上。

```ts
// Vue 2（不能 tree-shake）
import Vue from 'vue'
Vue.nextTick(...)

// Vue 3（可以 tree-shake）
import { nextTick, ref } from 'vue'
nextTick(...)
```

打包工具（Rollup/webpack）静态分析 import，未使用的导出被标记为 dead code 并移除。

---

### A15. 大型 Vue 3 项目的性能优化策略？

**编译时**：
- 避免在模板里写复杂表达式（会影响静态提升）
- 合理使用 `v-memo` 缓存稳定列表

**运行时**：
- 长列表用虚拟滚动（`@tanstack/vue-virtual`）
- 组件懒加载（`defineAsyncComponent` + 路由级 code split）
- `shallowRef` 存大型不变数据，手动 `triggerRef`
- 使用 `markRaw` 隔离第三方实例

**工程**：
- 按需引入 UI 组件库
- 合理拆分路由 chunk，避免首屏过大
- SSR / SSG 改善首屏时间

---

### A16. Vue 3 SSR 的 hydration 过程？

1. 服务端：调用 `renderToString` 生成 HTML 字符串 + 序列化状态
2. 客户端：收到 HTML 后，Vue 不重新创建 DOM，而是**接管已有 DOM**
3. `hydrateNode` 遍历真实 DOM，与 VNode 树对比：
   - 文本节点：直接验证内容
   - 元素节点：挂载事件监听、建立响应式
   - Mismatch：dev 下报警告，prod 下强制重新渲染

**Hydration Mismatch 常见原因**：`Date.now()`、`Math.random()`、浏览器特有 API 在 SSR 时执行结果不一致。

---

### A17. defineComponent 的作用？不用它行吗？

```ts
const Comp = defineComponent({
  props: { title: String },
  setup(props) { ... }
})
```

**运行时**：是个恒等函数，直接返回传入的对象，无任何开销。

**类型层面**：提供完整的 TypeScript 类型推导（props 类型推导、setup 返回值类型等）。

不用它完全可以运行，但 TypeScript 用户会失去类型推导。`<script setup>` 不需要它，因为编译器自动处理类型。

---

### A18. getCurrentInstance 的使用场景与注意事项？

```ts
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
// instance.proxy    → 组件公开代理（等同于模板里的 this）
// instance.emit     → emit 方法
// instance.slots    → slots
// instance.appContext → 应用上下文（全局属性、全局组件等）
```

**注意**：
- 只能在 `setup` 或生命周期钩子中调用（effect 执行期间不可用）
- 不是公共 API，版本间可能变化，尽量避免在业务代码里用
- 插件/库开发时用于访问应用上下文

---

### A19. Vue 3 插件开发的标准模式？

```ts
// 插件必须实现 install 方法
const MyPlugin = {
  install(app: App, options = {}) {
    // 1. 注册全局组件
    app.component('MyButton', MyButton)

    // 2. 注册全局指令
    app.directive('focus', { mounted: el => el.focus() })

    // 3. 全局注入（provide/inject 模式）
    app.provide('pluginOptions', options)

    // 4. 全局属性（谨慎使用，污染实例）
    app.config.globalProperties.$http = axios
  }
}

app.use(MyPlugin, { theme: 'dark' })
```

---

### A20. 如何设计一个可复用的 Modal 组件库？

考察点：Teleport、异步组件、provide/inject、命令式调用

```ts
// 命令式调用（高级用法）
const { open, close } = useModal(MyDialog)
open({ title: '确认删除？' })

// 实现原理：
// 1. 在 app 级别 provide 一个 ModalManager（响应式 stack）
// 2. 根组件用 Teleport 渲染 ModalManager 中的所有 modal
// 3. useModal 通过 inject 获取 ModalManager，push/pop modal
// 4. 支持 Promise 化：open() 返回 Promise，用户操作后 resolve/reject
```

---

## 高级考点速查

```
ReactiveEffect：activeEffect 全局单例，run() 时收集依赖
computed：scheduler 标脏 + _dirty 缓存，推拉结合
effectScope：统一管理一组 effect 的停止，防内存泄漏
PatchFlag：精确标记动态类型，-1=静态提升永不diff
Block Tree：dynamicChildren 扁平收集动态子孙，跳过静态
自定义渲染器：createRenderer(hostOptions) 替换平台操作
slot 本质：父传子的函数，渲染时在父作用域执行
SSR hydration：接管已有DOM，不重建，监听事件+建立响应式
Tree-shaking：具名导出 + 静态分析，Vue 2 prototype 挂载无法摇树
getCurrentInstance：仅 setup/钩子中可用，非公开API慎用
```
