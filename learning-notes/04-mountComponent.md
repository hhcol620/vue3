# mountComponent 完整流程

> 源码路径：`packages/runtime-core/src/renderer.ts` / `component.ts`
> 学习方式：从使用场景反推源码

---

## 一、mountComponent 做了哪三件事

```typescript
// renderer.ts:1183
const mountComponent = (initialVNode, container, ...) => {
  // 第一步
  const instance = createComponentInstance(initialVNode, parentComponent, parentSuspense)

  // 第二步
  setupComponent(instance)

  // 第三步
  setupRenderEffect(instance, initialVNode, container, ...)
}
```

| 步骤 | 函数 | 职责 |
|------|------|------|
| 1 | `createComponentInstance` | 开辟空白的 instance 对象（画布） |
| 2 | `setupComponent` | 往 instance 里填数据（props、slots、setupState） |
| 3 | `setupRenderEffect` | 启动渲染机器，消费 instance 里的数据 |

---

## 二、createComponentInstance：开辟画布

`component.ts:630`，只是初始化一个对象，重要属性分三类：

### 渲染核心（setupRenderEffect 后才有值）

```typescript
subTree: null   // 上次渲染出的 vnode 树，patch diff 时的"旧树"
effect: null    // ReactiveEffect 实例
update: null    // effect.run.bind → 强制立即重渲染
job: null       // effect.runIfDirty.bind → 进 queueJob 的函数
render: null    // 编译后的 render 函数
```

### 状态数据（setupComponent 后才有值）

```typescript
props: EMPTY_OBJ       // 响应式的 props
attrs: EMPTY_OBJ       // 非 prop 的透传属性
slots: EMPTY_OBJ       // 插槽
setupState: EMPTY_OBJ  // setup() 返回值，模板里的变量从这里读
ctx: { _: instance }   // 模板访问变量的代理上下文
proxy: null            // 暴露给模板/用户的公共代理（this）
```

### 生命周期钩子（缩写形式）

```typescript
bm: null  // beforeMount
m:  null  // mounted
bu: null  // beforeUpdate
u:  null  // updated
bum: null // beforeUnmount
um: null  // unmounted
```

都是数组，调用 `onMounted(fn)` 就是往 `instance.m` 里 push。

> **关键理解**：`createComponentInstance` 只是开辟画布，所有重要属性的值都在后续步骤填入。

---

## 三、setupComponent：填充画布

```typescript
// component.ts
function setupComponent(instance) {
  initProps(instance, rawProps)         // 处理 props
  initSlots(instance, children)         // 处理插槽
  setupStatefulComponent(instance)      // 核心：执行 setup()
}
```

### setupStatefulComponent 做了什么

对应你日常写的组件：

```vue
<script setup>
const props = defineProps({ title: String })
const count = ref(0)
function increment() { count.value++ }
</script>

<template>
  <h1>{{ title }}</h1>
  <button @click="increment">{{ count }}</button>
</template>
```

**第一步：创建 proxy（line 862）**

```typescript
instance.proxy = new Proxy(instance.ctx, PublicInstanceProxyHandlers)
```

模板里写 `{{ count }}` 时，Vue 通过这个 proxy 按顺序查找：
`setupState` → `props` → `data`，找到就返回。

**第二步：调用 setup() 函数（line 873）**

```typescript
callWithErrorHandling(setup, instance, ..., [props, setupContext])
```

真正执行你 `<script setup>` 里的代码，`ref(0)`、`defineProps` 在这一刻运行。

注意：调用前执行了 `pauseTracking()`，调用后 `resetTracking()`——setup 执行期间不收集依赖，避免污染。

**第三步：存储返回值**

`setup()` 返回的 `{ count, increment }` 存入 `instance.setupState`，之后模板通过 proxy 访问。

### 一句话总结

> `setupComponent` = **初始化 props/slots，执行 setup()，把返回值挂到 instance.setupState，让模板通过 proxy 能访问到数据**

---

## 四、完整流程图

```
mountComponent
  │
  ├─① createComponentInstance
  │     └─ 创建空白 instance（subTree/effect/proxy/setupState 全是 null/EMPTY_OBJ）
  │
  ├─② setupComponent
  │     ├─ initProps       → instance.props 填充（响应式）
  │     ├─ initSlots       → instance.slots 填充
  │     └─ setupStatefulComponent
  │           ├─ instance.proxy = new Proxy(ctx)   模板变量查找代理
  │           ├─ callWithErrorHandling(setup, ...)  执行用户的 setup()
  │           └─ instance.setupState = setup 返回值
  │
  └─③ setupRenderEffect                            （见 03-setupRenderEffect.md）
        └─ new ReactiveEffect(componentUpdateFn) → 首次渲染 → 后续响应式更新
```

---

## 五、面试常考

**Q：Vue 3 组件挂载经历了哪些步骤？**

> `mountComponent` 分三步：① `createComponentInstance` 创建组件实例（空白画布）；② `setupComponent` 初始化 props/slots 并执行 `setup()`，把返回值存入 `instance.setupState`；③ `setupRenderEffect` 创建 `ReactiveEffect(componentUpdateFn)`，触发首次渲染并建立响应式更新机制。

---

**Q：setup() 是在哪里被调用的？**

> `setupStatefulComponent` → `callWithErrorHandling(setup, instance, [props, setupContext])`。调用前后分别执行 `pauseTracking` / `resetTracking`，防止 setup 执行期间污染响应式依赖收集。

---

**Q：模板里的变量（如 `{{ count }}`）是怎么找到的？**

> 通过 `instance.proxy`（`new Proxy(instance.ctx, PublicInstanceProxyHandlers)`）。proxy 的 get 拦截按顺序查找 `setupState` → `props` → `data`，找到即返回。所以 setup 返回的变量、props、data 都能在模板里直接使用。

---

**Q：`createComponentInstance` 和 `setupComponent` 有什么区别？**

> `createComponentInstance` 只负责分配内存、创建空白 instance 对象；`setupComponent` 才真正填充数据（执行 setup、处理 props/slots）。职责分离，前者是"开辟画布"，后者是"在画布上作画"。

---

**Q：生命周期钩子（如 onMounted）是怎么注册的？**

> 调用 `onMounted(fn)` 时，Vue 取当前 `currentInstance`，把 `fn` push 进 `instance.m` 数组。`componentUpdateFn` 在首次渲染完成后遍历 `instance.m` 执行。这也是为什么生命周期钩子必须在 `setup()` 同步执行阶段调用——此时 `currentInstance` 才有值。
