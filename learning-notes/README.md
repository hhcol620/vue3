# Vue 3 源码学习笔记

## 目录

| 文件 | 内容 | 状态 |
|------|------|------|
| [01-reactivity.md](./01-reactivity.md) | 响应式系统：ref / reactive / computed / watch 全链路 | ✅ |
| [02-createApp.md](./02-createApp.md) | 应用启动：createApp → mount → 与响应式的汇合点 | ✅ |
| 03-mountComponent.md | 组件挂载：setupRenderEffect → 首次渲染 | 🔜 |
| 04-patch.md | DOM 更新：patch / diff 算法 | 🔜 |
| 05-compiler.md | 模板编译：template → render 函数 | 🔜 |

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
