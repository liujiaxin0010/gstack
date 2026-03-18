---
name: todos-format
description: TODOS.md 的标准格式参考。供其他技能创建和维护 TODOS.md 时引用。
---

# TODOS.md 标准格式

## 文件结构

```markdown
# TODOS

## Active

### P0 — Critical / Blocking
- [ ] **TODO 标题**
  - **What:** 一行描述需要做什么
  - **Why:** 它解决什么具体问题或释放什么价值
  - **Context:** 足够的细节，让 3 个月后拿起这个任务的人能理解动机、当前状态和从哪里开始
  - **Effort:** S/M/L/XL（人工团队）→ AI 辅助：S→S, M→S, L→M, XL→L
  - **Priority:** P0
  - **Depends on:** 任何前置条件或排序约束

### P1 — High Priority
- [ ] ...

### P2 — Important
- [ ] ...

### P3 — Nice to Have
- [ ] ...

### P4 — Someday/Maybe
- [ ] ...

## Completed
- [x] **已完成的 TODO 标题**
  - **Completed:** vX.Y.Z (YYYY-MM-DD)
  - （保留原始 What/Why/Context 以供参考）
```

## 优先级定义

| 优先级 | 含义 | 何时处理 |
|--------|------|---------|
| P0 | 阻塞其他工作或影响生产环境 | 立即 |
| P1 | 高优先级，本迭代内完成 | 本周 |
| P2 | 重要但不紧急 | 本月 |
| P3 | 锦上添花 | 有空时 |
| P4 | 也许有一天 | 不确定 |

## 规则

1. 每个 TODO 必须有 What + Why + Context，缺少上下文的 TODO 比没有 TODO 更糟
2. 完成项移入 Completed 区域，标注版本和日期
3. 不要删除已完成项——它们是历史记录
4. 新增 TODO 时，通过提问让 Boss 确认优先级
