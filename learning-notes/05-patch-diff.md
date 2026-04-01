# patch / diff 算法

> 源码路径：`packages/runtime-core/src/renderer.ts`
> 核心函数：`patch`、`patchKeyedChildren`、`getSequence`

---

## 一、patch 是什么

`patch` 是一个**纯分发器**，自身不操作 DOM，只根据 vnode 类型决定交给谁处理。

```typescript
// renderer.ts:374
const patch = (n1, n2, container, ...) => {
  if (n1 === n2) return   // 完全相同直接跳过

  // 类型不同：卸载旧节点，n1 置 null 走挂载逻辑
  if (n1 && !isSameVNodeType(n1, n2)) {
    unmount(n1, ...)
    n1 = null
  }

  switch (n2.type) {
    case Text:      processText(n1, n2, ...)
    case Comment:   processCommentNode(n1, n2, ...)
    case Fragment:  processFragment(n1, n2, ...)
    default:
      if (shapeFlag & ShapeFlags.ELEMENT)
        processElement(n1, n2, ...)       // div / span 等普通元素
      else if (shapeFlag & ShapeFlags.COMPONENT)
        processComponent(n1, n2, ...)     // 组件
  }
}
```

以 Element 为例，进入 `processElement` 后：

```
processElement
  ├── n1 为 null → mountElement（首次挂载）
  └── n1 存在   → patchElement（更新）
        ├── patchProps      对比属性
        └── patchChildren   对比子节点 ← diff 核心在这里
              ├── 有 key → patchKeyedChildren
              └── 无 key → patchUnkeyedChildren
```

---

## 二、patchKeyedChildren：五步 diff 策略

场景：旧 `[a b c d e f g]`，新 `[a b e d c h f g]`

### 第一步：从头对比（line 1804）

```
旧: [a b] c d e f g
新: [a b] e d c h f g
```

头部 `a b` 类型和 key 都相同，直接 patch 复用，`i` 往后推。

### 第二步：从尾对比（line 1830）

```
旧: a b c d e [f g]
新: a b e d c h [f g]
```

尾部 `f g` 相同，直接 patch 复用，`e1` `e2` 往前推。

经过两步，锁定中间未处理部分：旧 `[c d e]`，新 `[e d c h]`

### 第三步：旧完新有 → 新增（line 1861）

```
旧: (a b)
新: (a b) c d    ← 新节点还有剩，全部 mount
```

### 第四步：新完旧有 → 删除（line 1891）

```
旧: (a b) c d    ← 旧节点还有剩，全部 unmount
新: (a b)
```

### 第五步：乱序处理（line 1902）← 最复杂

针对中间乱序部分 旧`[c d e]` → 新`[e d c h]`，分三小步：

**5.1 建立新节点的 `key → index` Map**

```typescript
keyToNewIndexMap = { e: 0, d: 1, c: 2, h: 3 }
```

**5.2 遍历旧节点，找复用 / 删除多余**

```
c → 在 Map 中找到，新位置 2，patch 复用
d → 在 Map 中找到，新位置 1，patch 复用
e → 在 Map 中找到，新位置 0，patch 复用
h → 旧中没有 → 标记为新增（newIndexToOldIndexMap 中值为 0）
```

同时记录 `newIndexToOldIndexMap`（新位置 → 旧位置的映射）：

```
newIndexToOldIndexMap = [3, 2, 1, 0]
// 新位置0的e来自旧位置3，新位置1的d来自旧位置2，以此类推
// 0 表示没有对应旧节点（新增）
```

**5.3 用 LIS 最小化 DOM 移动**

```typescript
const increasingNewIndexSequence = moved ? getSequence(newIndexToOldIndexMap) : []
// getSequence([3, 2, 1, 0]) → [2] 即索引2对应的节点(c)不需要移动
// 实际上会得到最长不需要移动的序列
```

从后往前遍历新节点：
- 在 LIS 中的节点：**不移动**，原地复用
- 不在 LIS 中的节点：**移动** DOM
- `newIndexToOldIndexMap` 为 0 的节点：**新增** mount

---

## 三、LIS 的作用（核心优化）

```
newIndexToOldIndexMap = [3, 2, 1, 0]
```

**没有 LIS**：不知道哪些节点相对顺序未变，把 e d c 全部移动，3 次 DOM 操作

**有 LIS**：算出最长递增子序列 = `[1, 2]`（d、c 相对顺序未变，不需要移动）
→ 只移动 e，1 次 DOM 操作

> LIS 算法回答的问题：**「哪些节点在新旧列表中的相对顺序没有变，不需要移动？」**

---

## 四、为什么需要 key

没有 key 时，Vue 只能按位置（index）比对，顺序变化会导致大量不必要的 patch。

有 key 时，通过 `key → index Map` 精确找到可复用的旧节点，跨位置复用。

```
// 无 key：[a b c] → [c a b]
// 位置0: a vs c → 不同，patch（实际内容不一样却强行更新）
// 位置1: b vs a → 不同，patch
// 位置2: c vs b → 不同，patch
// 共 3 次无效 patch

// 有 key：[a b c] → [c a b]
// 通过 Map 找到 c/a/b 都能复用
// LIS 算出 [a b] 不需要移动，只移动 c
// 共 1 次 DOM 移动
```

---

## 五、完整流程图

```
数据变化
  → componentUpdateFn 重新执行
  → renderComponentRoot() 生成新 vnode 树
  → patch(oldVnode, newVnode)
      └── processElement → patchElement
            ├── patchProps（属性 diff）
            └── patchChildren
                  └── patchKeyedChildren
                        ├── 第1步：头部预处理（跳过相同前缀）
                        ├── 第2步：尾部预处理（跳过相同后缀）
                        ├── 第3步：旧完新有 → mount 新增节点
                        ├── 第4步：新完旧有 → unmount 多余节点
                        └── 第5步：乱序处理
                              ├── 建 key→index Map
                              ├── 遍历旧节点找复用/删除
                              └── LIS 算法最小化 DOM 移动
```

---

## 六、面试常考

**Q：Vue 3 的 diff 算法是什么？**

> Vue 3 diff 核心在 `patchKeyedChildren`，分五步：① 从头对比跳过相同前缀；② 从尾对比跳过相同后缀；③ 旧节点处理完但新节点还有，直接 mount；④ 新节点处理完但旧节点还有，直接 unmount；⑤ 中间乱序部分，建 `key→index Map` 找可复用节点，再用 LIS 算法计算最少需要移动的节点，最小化 DOM 操作次数。

---

**Q：Vue 3 diff 和 Vue 2 diff 有什么区别？**

> Vue 2 使用双端对比（头头、尾尾、头尾、尾头四种比较），Vue 3 用头尾双端预处理 + 中间乱序用 LIS 算法处理。Vue 3 的优化点在于 LIS 能精确算出不需要移动的最大节点集合，比 Vue 2 减少了更多不必要的 DOM 移动。

---

**Q：为什么列表渲染需要加 key？**

> 没有 key 时只能按 index 位置比对，顺序变化会导致大量错误 patch（类型相同但内容不同的节点被复用）。有 key 后，`patchKeyedChildren` 通过 `key→index Map` 精确找到可复用的旧节点，跨位置复用 DOM，再配合 LIS 最小化移动次数，性能更优且结果正确。

---

**Q：LIS（最长递增子序列）在 diff 中解决了什么问题？**

> 解决「哪些节点不需要移动」的问题。旧新节点映射关系记录在 `newIndexToOldIndexMap` 中，对它求 LIS 得到相对顺序未变的最大节点集合，这些节点原地复用，只移动不在 LIS 中的节点，将 DOM 移动次数降到最低。

---

## 七、暂跳过的部分

- `patchUnkeyedChildren`（无 key 时的简单按位置对比）
- `patchProps` 内部（class / style / event 的具体对比逻辑）
- `Fragment` 的 diff 处理
- 编译器静态提升（`hoistStatic`）如何配合 diff 跳过静态节点
