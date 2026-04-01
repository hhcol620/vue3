# Vue 3 源码学习笔记

## 目录

| 文件 | 内容 | 状态 |
|------|------|------|
| [00-学习路线.md](./00-学习路线.md) | 整体学习路线：两条主线、学习顺序与理由、知识地图 | ✅ |
| [01-createApp.md](./01-createApp.md) | 应用启动：createApp → mount → 与响应式的汇合点 | ✅ |
| [02-renderer-overview.md](./02-renderer-overview.md) | renderer 导读：baseCreateRenderer 学习策略与主线顺序 | ✅ |
| [03-setupRenderEffect.md](./03-setupRenderEffect.md) | 响应式与渲染的连接点：setupRenderEffect 详解 | ✅ |
| [04-mountComponent.md](./04-mountComponent.md) | 组件挂载：mountComponent 完整流程 | ✅ |
| [05-patch-diff.md](./05-patch-diff.md) | DOM 更新：patch / diff 算法 | ✅ |
| 06-compiler.md | 模板编译：template → render 函数 | 🔜 |
| [07-reactivity.md](./07-reactivity.md) | 响应式系统：ref / reactive / computed / watch 全链路 | 🔄 |

## 核心包依赖关系

```
vue ──────────────────────────┐
  └─► runtime-dom             │
        └─► runtime-core      │
              └─► reactivity  │
  └─► compiler-dom ◄──────────┘
        └─► compiler-core
              ▲
        compiler-sfc
        compiler-ssr
```

## 两条主线

```
【启动线】createApp → mount → mountComponent → setupRenderEffect
                                                      ↕ 汇合
【响应线】ref/reactive → dep.track → dep.trigger → effect.run → 重渲染
```
