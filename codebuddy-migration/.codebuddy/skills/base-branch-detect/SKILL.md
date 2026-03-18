---
name: base-branch-detect
description: 检测当前分支的目标基础分支（main/master/develop 等）。供其他技能引用。
---

# 基础分支检测

确定当前 PR 或分支的目标基础分支。在后续所有步骤中使用检测结果作为"基础分支"。

## 检测步骤

1. 检查是否已有 PR 存在（Git 项目）：
   ```bash
   gh pr view --json baseRefName -q .baseRefName
   ```
   如果成功，使用打印的分支名称作为基础分支。

2. 如果没有 PR（命令失败），检测仓库默认分支：
   ```bash
   gh repo view --json defaultBranchRef -q .defaultBranchRef.name
   ```

3. 如果以上命令都失败，回退到 `main`。

4. **SVN 兼容**：如果项目使用 SVN 而非 Git：
   ```bash
   svn info --show-item url 2>/dev/null
   ```
   从 URL 中提取分支信息（trunk/branches/tags 结构）。

输出检测到的基础分支名称。在后续所有 `git diff`、`git log`、`git fetch`、`git merge`、`gh pr create` 命令中，替换为检测到的分支名称。
