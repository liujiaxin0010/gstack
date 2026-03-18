# 移植计划：gstack 角色能力 → superpowers-for-codebuddy

## 目标

将 gstack 的 8 个角色技能（CEO 评审、工程经理评审、设计师审计/修复/咨询、QA 测试/修复、代码审查、发布工程、技术写作、工程复盘）完整移植到 `superpowers-for-codebuddy` 的 `.codebuddy/skills/` + `.codebuddy/agents/` + `.codebuddy/commands/` 体系中。

## 架构适配原则

| gstack 概念 | CodeBuddy 对应 | 适配方式 |
|-------------|---------------|---------|
| `SKILL.md.tmpl` → `SKILL.md` | `.codebuddy/skills/<name>/SKILL.md` | 模板占位符内联展开，直接写 SKILL.md |
| `{{PREAMBLE}}` | 各 SKILL.md 开头统一段落 | 去除 gstack 专有部分（升级检查、session 追踪），保留方法论（完整性原则、AskUserQuestion 格式） |
| `{{BROWSE_SETUP}}` | 新增 `browse-setup/` 公共技能 | 浏览器工具定位逻辑独立为技能 |
| `{{BASE_BRANCH_DETECT}}` | 内联到每个需要的技能 | 用 `gh` 命令检测，与 CodeBuddy 的 SVN 双支持合并 |
| `{{QA_METHODOLOGY}}` | `qa-methodology/` 公共技能 | 提取为独立技能，被 qa/qa-only 引用 |
| `{{DESIGN_METHODOLOGY}}` | `design-methodology/` 公共技能 | 提取为独立技能，被 design 系列引用 |
| `AskUserQuestion` 工具 | 自然语言提问 | 改为 "请向 Boss 提问：..." 格式，保留推荐+完整度评分结构 |
| `gstack-slug` | `basename $(pwd)` 或 `git remote` | 简单 bash 替代 |
| `~/.gstack/projects/` | `docs/` 或 `~/.codebuddy/projects/` | 持久化路径改为 CodeBuddy 体系 |
| `gstack-diff-scope` | 内联 bash 逻辑 | 用 `git diff --name-only` 直接判断 |
| `$B` (browse 二进制) | `$B` (保持不变) | browse CLI 整体搬入 |

## 统一前置段落（Preamble）

所有移植的技能共享以下前置段落（代替 gstack 的 `{{PREAMBLE}}`）：

```markdown
## 三条铁律（最高优先级）
1. 每次回复的第一句话必须称呼 "Boss"
2. 遇到不确定的设计问题时，必须先询问 Boss，不得擅自行动
3. 不得编写兼容性代码，除非 Boss 主动明确要求

## 提问格式（向 Boss 提问时必须遵循）
1. **重新定位：** 说明当前项目、当前分支、当前任务（1-2 句）
2. **简化说明：** 用通俗语言解释问题，聪明的高中生能看懂
3. **推荐：** `推荐：选择 [X]，因为 [理由]` — 附带完整度评分 (1-10)
4. **选项：** A) ... B) ... C) ... — 显示人工/AI 双时间

## 完整性原则
AI 辅助编码使完整实现的边际成本趋近于零。永远推荐完整方案而非捷径。
```

## 移植清单（按执行顺序）

---

### 第 0 步：环境准备

1. Fork/clone `superpowers-for-codebuddy` 到工作目录
2. 在 gstack 仓库的 `claude/project-review-summary-qxzTe` 分支上工作
3. 创建目标目录结构

---

### 第 1 步：公共方法论技能（被多个角色引用）

#### 1.1 创建 `.codebuddy/skills/qa-methodology/SKILL.md`
- **来源：** `gen-skill-docs.ts` 的 `generateQAMethodology()` 函数（~270 行）
- **内容：** QA 6 阶段方法论（初始化→认证→定位→探索→文档→总结）
- **适配：** `$B` 保持不变，`gstack-slug` 改为 `basename $(pwd)`，健康评分量表保留

#### 1.2 创建 `.codebuddy/skills/design-methodology/SKILL.md`
- **来源：** `gen-skill-docs.ts` 的 `generateDesignMethodology()` 函数（~270 行）
- **内容：** 设计审计 6 阶段（第一印象→设计系统提取→逐页审计→交互流评审→跨页一致性→报告编制）
- **适配：** 80 项设计检查清单完整保留，AI 泛化检测 10 项反模式完整保留

#### 1.3 创建 `.codebuddy/skills/base-branch-detect/SKILL.md`
- **来源：** `gen-skill-docs.ts` 的 `generateBaseBranchDetect()` 函数
- **内容：** `gh pr view` → `gh repo view` → 回退 `main` 的三级检测
- **适配：** 加入 SVN 分支检测兼容

#### 1.4 创建 `.codebuddy/skills/review-checklist/SKILL.md`
- **来源：** `review/checklist.md`（172 行）
- **内容：** 两遍审查结构（关键：SQL/竞态/LLM 信任/枚举完整性；信息性：8 类）

#### 1.5 创建 `.codebuddy/skills/todos-format/SKILL.md`
- **来源：** `review/TODOS-format.md`（62 行）
- **内容：** TODOS.md 标准格式参考

---

### 第 2 步：纯方法论角色（无浏览器依赖）

#### 2.1 CEO 评审 — `/ceo-review`

**新文件：**
- `.codebuddy/skills/ceo-review/SKILL.md`（从 `plan-ceo-review/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/ceo-reviewer.md`（新建 agent）
- `.codebuddy/commands/ceo-review.md`（新建命令）

**SKILL.md 核心内容（~500 行）：**
- 14 种 CEO 认知模式完整保留（分类本能、偏执扫描、反转反射、减法焦点、时间深度等）
- 4 种范围模式：扩展/选择性扩展/保持/收缩
- 10 个评审维度：架构、错误恢复、安全、数据流、代码质量、测试、性能、可观测性、部署、长期轨迹
- 输出：CEO 计划文档 + 架构图 + 失败模式登记表 + TODOS 更新

**Agent 定义：**
```yaml
name: CEO 评审员
description: 创始人/CEO 视角的战略级方案评审。前提质疑、范围决策、10 年思维、逆向失败分析。
model: glm-4.7
tools: use_skill, read_file, search_content, list_files, execute_command, write_to_file
```

#### 2.2 工程经理评审 — `/eng-review`

**新文件：**
- `.codebuddy/skills/eng-review/SKILL.md`（从 `plan-eng-review/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/eng-reviewer.md`
- `.codebuddy/commands/eng-review.md`

**SKILL.md 核心内容（~400 行）：**
- 工程经理认知模式（状态诊断、爆炸半径本能、无聊优先、系统优于英雄等）
- 范围质疑（复杂度嗅觉：>8 文件=坏味道）
- 4 维评审：架构、代码质量、测试、性能
- 输出：测试计划工件 + 逐维度问题 + 失败模式登记表

#### 2.3 代码审查增强 — 增强 `code-review-standards/`

**修改文件：**
- `.codebuddy/skills/code-review-standards/SKILL.md`（增强现有技能）
- `.codebuddy/agents/code-reviewer.md`（增强现有 agent）

**新增内容：**
- gstack 的两遍审查结构（关键 + 信息性）
- Fix-First 启发式（自动修复 vs 询问分类）
- SQL 安全、竞态条件、LLM 信任边界、枚举完整性专项检查
- Greptile 评论分类+回复模板（适配为通用外部审查工具）
- TODOS 交叉引用
- 文档过时检查

#### 2.4 技术写作 — `/document-release`

**新文件：**
- `.codebuddy/skills/document-release/SKILL.md`（从 `document-release/SKILL.md.tmpl` 移植）
- `.codebuddy/commands/document-release.md`

**SKILL.md 核心内容（~300 行）：**
- 9 步文档更新流程（预检→逐文件审计→自动修正→主观变更确认→CHANGELOG 润色→跨文档一致性→TODOS 清理→版本号→提交）
- CHANGELOG 用户化改写规则
- 文档健康状态摘要

#### 2.5 工程复盘 — `/retro`

**新文件：**
- `.codebuddy/skills/retro/SKILL.md`（从 `retro/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/retro-analyst.md`
- `.codebuddy/commands/retro.md`

**SKILL.md 核心内容（~500 行）：**
- 12 项并行 git 数据采集
- 工作会话检测（45 分钟间隔阈值，深度/中度/微会话分类）
- commit 类型分解（feat/fix/refactor/test/chore/docs 百分比柱）
- 热点分析 + PR 大小分布 + 焦点评分
- 逐人分析（表扬 1-2 项 + 成长机会 1 项）
- 连续工作天数追踪
- JSON 快照持久化 + 周对周趋势

#### 2.6 发布工程 — `/ship`

**新文件：**
- `.codebuddy/skills/ship/SKILL.md`（从 `ship/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/release-engineer.md`
- `.codebuddy/commands/ship.md`

**SKILL.md 核心内容（~600 行）：**
- 8 步全自动发布流程
  - Step 1: 预检（分支、未提交更改）
  - Step 2: 合并基础分支
  - Step 2.5: 测试框架引导
  - Step 3: 运行测试
  - Step 3.4: 测试覆盖率审计（代码路径追踪+ASCII 图）
  - Step 3.5: 预着陆审查（引用 review-checklist 技能）
  - Step 4: 版本号升级决策
  - Step 5: CHANGELOG 自动生成
  - Step 5.5: TODOS.md 更新
  - Step 6: 可二分提交拆分
  - Step 7: 推送
  - Step 8: 创建 PR
- 适配：`gstack-slug` → bash 替代，`gstack-diff-scope` → 内联逻辑

---

### 第 3 步：浏览器依赖角色

#### 3.1 移植 browse CLI

**新文件：**
- `.codebuddy/skills/browse/` 目录（从 `gstack/browse/` 整体复制）
  - `src/` — CLI 源码
  - `dist/` — 编译产物
  - `SKILL.md` — 使用指南

**适配：**
- browse 是纯 Playwright CLI，零 AI 模型依赖，直接可用
- 构建脚本：`cd .codebuddy/skills/browse && bun install && bun run build`
- 设置 `$B` 变量指向二进制路径

#### 3.2 设计审计（仅报告）— `/design-review`

**新文件：**
- `.codebuddy/skills/design-review/SKILL.md`（从 `plan-design-review/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/design-reviewer.md`
- `.codebuddy/commands/design-review.md`

**SKILL.md 核心内容（~400 行）：**
- 设计师认知模式（系统视角、同理心模拟、层级即服务、约束崇拜、减法默认等）
- 引用 `design-methodology` 公共技能执行 6 阶段审计
- 80 项设计检查清单（10 类）
- AI 泛化检测（10 项反模式黑名单）
- 双评分：设计分 A-F + AI 泛化分 A-F
- 输出：设计审计报告 + 截图证据 + 推断的设计系统

#### 3.3 设计审计+修复 — `/design-fix`

**新文件：**
- `.codebuddy/skills/design-fix/SKILL.md`（从 `qa-design-review/SKILL.md.tmpl` 移植）
- `.codebuddy/commands/design-fix.md`

**SKILL.md 核心内容（~250 行）：**
- Phase 1-6: 同 design-review 的审计流程
- Phase 7: 发现分级（高/中/打磨）
- Phase 8: 修复循环（定位→修复→原子提交→截图验证）
- Phase 8e.5: 回归测试（CSS 跳过；JS 行为改动需测试）
- Phase 8f: 自律机制（设计修复风险启发式，30 项硬上限）
- Phase 9: 最终审计（重新评分，检测回归）
- Phase 10: 报告（已验证/尽力/已回滚/已推迟）

#### 3.4 设计咨询 — `/design-consultation`

**新文件：**
- `.codebuddy/skills/design-consultation/SKILL.md`（从 `design-consultation/SKILL.md.tmpl` 移植）
- `.codebuddy/commands/design-consultation.md`

**SKILL.md 核心内容（~400 行）：**
- 6 阶段咨询流程（检查现有 DESIGN.md → 大问题 → 竞品研究 → 完整提案 → 细化 → 预览页+DESIGN.md 输出）
- 设计知识库：10 种美学方向、字体推荐/黑名单/过度使用列表
- AI 泛化反模式清单
- 输出：DESIGN.md + 字体颜色预览 HTML 页面

#### 3.5 QA 测试报告 — `/qa-report`

**新文件：**
- `.codebuddy/skills/qa-report/SKILL.md`（从 `qa-only/SKILL.md.tmpl` 移植）
- `.codebuddy/commands/qa-report.md`

**SKILL.md 核心内容（~100 行）：**
- 引用 `qa-methodology` 公共技能
- 仅报告模式：不修复、不建议、不碰源码
- 输出：QA 报告 + 截图 + 健康评分 + 基线 JSON

#### 3.6 QA 测试+修复 — `/qa-fix`

**新文件：**
- `.codebuddy/skills/qa-fix/SKILL.md`（从 `qa/SKILL.md.tmpl` 移植）
- `.codebuddy/agents/qa-engineer.md`
- `.codebuddy/commands/qa-fix.md`

**SKILL.md 核心内容（~300 行）：**
- 11 阶段 QA 流程（发现→分级→修复循环→最终 QA→报告→TODOS 更新）
- WTF 概率自律机制
- 回归测试生成（Phase 8e.5）
- 修复循环：定位→修复→原子提交→截图验证→分类结果
- 输出：修复的 commit + 测试结果工件 + QA 报告

---

### 第 4 步：路由集成

#### 4.1 更新 `devflow-router` 技能
- 新增任务类型路由：
  - `ceo-review` → `/ceo-review`
  - `eng-review` → `/eng-review`
  - `design-review` → `/design-review`
  - `design-fix` → `/design-fix`
  - `design-consultation` → `/design-consultation`
  - `qa-report` → `/qa-report`
  - `qa-fix` → `/qa-fix`
  - `document-release` → `/document-release`
  - `retro` → `/retro`
  - `ship` → `/ship`

#### 4.2 更新 Featureflow 总控 Agent
- 在路由决策中增加新的命令识别
- 更新 `recommendedCommand` 候选列表

---

### 第 5 步：browse CLI 构建支持

- 在项目根目录添加 browse 构建说明
- 创建 setup 脚本：`cd .codebuddy/skills/browse && bun install && bun run build`
- 验证 `$B` 变量可正确指向二进制

---

### 第 6 步：README 更新

更新项目 README.md：
- 新增角色能力矩阵表
- 新增命令列表（从 20 个扩展到 ~30 个）
- 新增 browse CLI 安装说明

---

## 新增文件清单（共 ~30 个文件）

### 公共方法论技能（5 个）
- `.codebuddy/skills/qa-methodology/SKILL.md`
- `.codebuddy/skills/design-methodology/SKILL.md`
- `.codebuddy/skills/base-branch-detect/SKILL.md`
- `.codebuddy/skills/review-checklist/SKILL.md`
- `.codebuddy/skills/todos-format/SKILL.md`

### 角色技能（10 个）
- `.codebuddy/skills/ceo-review/SKILL.md`
- `.codebuddy/skills/eng-review/SKILL.md`
- `.codebuddy/skills/document-release/SKILL.md`
- `.codebuddy/skills/retro/SKILL.md`
- `.codebuddy/skills/ship/SKILL.md`
- `.codebuddy/skills/design-review/SKILL.md`
- `.codebuddy/skills/design-fix/SKILL.md`
- `.codebuddy/skills/design-consultation/SKILL.md`
- `.codebuddy/skills/qa-report/SKILL.md`
- `.codebuddy/skills/qa-fix/SKILL.md`

### Agent 定义（6 个）
- `.codebuddy/agents/ceo-reviewer.md`
- `.codebuddy/agents/eng-reviewer.md`
- `.codebuddy/agents/retro-analyst.md`
- `.codebuddy/agents/release-engineer.md`
- `.codebuddy/agents/design-reviewer.md`
- `.codebuddy/agents/qa-engineer.md`

### 命令（10 个）
- `.codebuddy/commands/ceo-review.md`
- `.codebuddy/commands/eng-review.md`
- `.codebuddy/commands/document-release.md`
- `.codebuddy/commands/retro.md`
- `.codebuddy/commands/ship.md`
- `.codebuddy/commands/design-review.md`
- `.codebuddy/commands/design-fix.md`
- `.codebuddy/commands/design-consultation.md`
- `.codebuddy/commands/qa-report.md`
- `.codebuddy/commands/qa-fix.md`

### Browse CLI（整体复制）
- `.codebuddy/skills/browse/` 目录

### 修改文件（3 个）
- `.codebuddy/skills/devflow-router/SKILL.md` — 新增路由
- `.codebuddy/agents/Featureflow.md` — 新增命令识别
- `README.md` — 新增文档

---

## 不移植的内容

| 内容 | 原因 |
|------|------|
| `gstack-upgrade/` | 完全绑定 gstack 生态的自动升级 |
| `setup-browser-cookies/` | 浏览器 Cookie 导入，可后续按需添加 |
| E2E 测试系统 | 绑定 `@anthropic-ai/sdk`，需重写为 GLM |
| LLM 评判系统 | 评判标准需针对 GLM 重新校准 |
| Greptile 集成 | 专有外部工具，可作为可选扩展 |
| Contributor Mode | gstack 社区机制，不适用 |
| 升级检查/Session 追踪 | gstack 运维机制，不适用 |

---

## 风险与缓解

| 风险 | 缓解 |
|------|------|
| GLM 对 500+ 行提示词的遵循能力 | 拆分大技能为子步骤，关键指令重复强调 |
| browse CLI 需要 bun + Playwright | 提供 setup 脚本，文档说明依赖 |
| 设计审计的 `$B js` 命令需要浏览器 | 无浏览器时跳过 Phase 2-5，仅做代码级审计 |
| 与现有 32 个技能的命名冲突 | 使用不同的命名空间（ceo-review vs code-review-standards） |
