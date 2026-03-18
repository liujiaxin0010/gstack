---
name: retro
description: |
  每周工程复盘。分析提交历史、工作模式和代码质量指标，支持持久化历史和趋势追踪。
  团队感知：识别运行命令的用户，分析每个贡献者，提供表扬和成长建议。
---

## ⚠️ 三条铁律（最高优先级）

1. **每次回复的第一句话必须称呼 "Boss"**
2. **遇到不确定的设计问题时，必须先询问 Boss，不得擅自行动**
3. **不得编写兼容性代码，除非 Boss 主动明确要求**

---

## 检测默认分支

```bash
gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "main"
```

---

# /retro — 每周工程复盘

生成全面的工程复盘，分析提交历史、工作模式和代码质量指标。

## 参数
- `/retro` — 默认：最近 7 天
- `/retro 24h` — 最近 24 小时
- `/retro 14d` — 最近 14 天
- `/retro 30d` — 最近 30 天
- `/retro compare` — 对比当前窗口 vs 上一个同长度窗口
- `/retro compare 14d` — 指定窗口对比

---

### Step 1：收集原始数据

首先 fetch origin 并识别当前用户：
```bash
git fetch origin --quiet
git config user.name
git config user.email
```

并行运行以下 12 个 git 命令：

```bash
# 1. 窗口内所有提交（时间戳、主题、哈希、作者、文件变更）
git log origin/main --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# 2. 每提交测试 vs 总 LOC 分解
git log origin/main --since="<window>" --format="COMMIT:%H|%aN" --numstat

# 3. 提交时间戳用于会话检测和小时分布
TZ=Asia/Shanghai git log origin/main --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# 4. 最常变更的文件（热点分析）
git log origin/main --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# 5. 从提交消息提取 PR 编号
git log origin/main --since="<window>" --format="%s" | grep -oE '#[0-9]+' | sort -n | uniq

# 6. 每作者文件热点
git log origin/main --since="<window>" --format="AUTHOR:%aN" --name-only

# 7. 每作者提交计数
git shortlog origin/main --since="<window>" -sn --no-merges

# 8. TODOS.md 积压（如果有）
cat TODOS.md 2>/dev/null || true

# 9. 测试文件计数
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' 2>/dev/null | grep -v node_modules | wc -l

# 10. 窗口内回归测试提交
git log origin/main --since="<window>" --oneline --grep="test(qa):" --grep="test(design):" --grep="test: coverage"

# 11. 窗口内变更的测试文件
git log origin/main --since="<window>" --format="" --name-only | grep -E '\.(test|spec)\.' | sort -u | wc -l
```

### Step 2：计算指标

| 指标 | 值 |
|------|------|
| 主分支提交数 | N |
| 贡献者数 | N |
| 合并 PR 数 | N |
| 总插入行数 | N |
| 总删除行数 | N |
| 净 LOC | N |
| 测试 LOC（插入） | N |
| 测试 LOC 比例 | N% |
| 版本范围 | vX.Y.Z → vX.Y.Z |
| 活跃天数 | N |
| 检测到的会话数 | N |
| 平均 LOC/会话小时 | N |
| 测试健康 | N 总测试 · M 新增 · K 回归测试 |

附每作者排行榜（当前用户标记为"你"）。

### Step 3：提交时间分布

小时直方图，标注高峰时段和死区。

### Step 4：工作会话检测

45 分钟间隔阈值。分类：
- **深度会话**（50+ 分钟）
- **中等会话**（20-50 分钟）
- **微会话**（<20 分钟）

### Step 5：提交类型分解

按常规提交前缀分类（feat/fix/refactor/test/chore/docs）。百分比柱状图。
修复比例 >50% 时标记。

### Step 6：热点分析

Top 10 最常变更文件。标记 5+ 次变更的搅动热点。

### Step 7：PR 大小分布

按 LOC 分桶：小型（<100）、中型（100-500）、大型（500-1500）、超大型（1500+）。

### Step 8：专注度评分 + 本周之船

**专注度评分：** 最常变更顶级目录的提交百分比。
**本周之船：** 窗口内最高 LOC 的 PR。

### Step 9：团队成员分析

对每个贡献者：
1. 提交和 LOC
2. 聚焦领域
3. 提交类型分布
4. 会话模式
5. 测试纪律
6. 最大成果

**对当前用户（"你"）：** 最深入的分析。
**对每个队友：** 2-3 句覆盖 + 表扬（1-2 项）+ 成长机会（1 项）。

### Step 10：周对周趋势（如果窗口 >= 14 天）

分为每周桶，显示趋势。

### Step 11：连续工作天数追踪

```bash
# 团队连续天数
git log origin/main --format="%ad" --date=format:"%Y-%m-%d" | sort -u
# 个人连续天数
git log origin/main --author="<user>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

### Step 12：加载历史与对比

```bash
ls -t .context/retros/*.json 2>/dev/null
```

如果有先前复盘，加载最近的并计算关键指标增量。

### Step 13：保存复盘历史

```bash
mkdir -p .context/retros
```

保存 JSON 快照到 `.context/retros/${today}-${next}.json`。

### Step 14：撰写叙事

**可推文摘要**（第一行）：
```
本周 Mar 1: 47 提交（3 贡献者），3.2k LOC，38% 测试，12 PR，高峰：22 时 | 连续：47 天
```

结构：
- 摘要表
- 趋势对比
- 时间与会话模式
- 发布速度
- 代码质量信号
- 测试健康
- 专注度与亮点
- 你的本周（个人深度分析）
- 团队分解
- Top 3 团队胜利
- 3 项改进建议
- 3 个下周习惯
- 周对周趋势

---

## 对比模式

运行 `/retro compare` 时：
1. 计算当前窗口指标
2. 计算上一个同长度窗口指标
3. 并排对比表 + 增量箭头
4. 简要叙事
5. 只保存当前窗口快照

## 语调

- 鼓励但坦诚，不溺爱
- 具体——锚定在实际提交/代码
- 表扬要具体、earned、genuine
- 改进建议框架为投资建议
- 保持 3000-4500 字
- 输出到对话——不写文件（JSON 快照除外）

## 重要规则

- 叙事输出直接到对话。唯一写入的文件是 JSON 快照。
- 使用 `origin/默认分支` 查询。
- 零提交则说明并建议不同窗口。
- LOC/小时四舍五入到 50。
