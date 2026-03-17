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

## 下一步深入

- [ ] `mountComponent` → `setupRenderEffect` 详细分析
- [ ] `patchKeyedChildren` diff 算法（最长递增子序列）
