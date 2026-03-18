# gstack 项目总结文档

## 一、项目概述

**gstack**（Garry's Stack）是由 Y Combinator CEO Garry Tan 创建的**开源 AI 工程工作流自动化系统**。它是一组 Claude Code 技能（Skills）+ 持久化无头浏览器 CLI 的集合，能将 Claude 转变为一支虚拟工程团队，涵盖从规划、实现、QA 到发布的完整软件开发生命周期。

**核心价值：** 一个仓库、一次安装，即可获得完整的 AI 工程工作流。系统包含 13+ 个专业斜杠命令（Slash Commands），每个都是一个 Claude Code 技能，使开发者每天可产出 10,000–20,000 行可用代码。

**技术栈：** TypeScript + Bun（编译为原生二进制）、Playwright + Chromium（持久化守护进程模型）、SQLite（Cookie 解密）、MIT 许可证。

---

## 二、项目架构

```
gstack/
├── browse/                # 无头浏览器 CLI（Playwright）
│   ├── src/               # CLI + 服务器 + 命令
│   │   ├── commands.ts    # 命令注册表（唯一真实来源）
│   │   ├── snapshot.ts    # 无障碍树 + @ref 引用系统
│   │   ├── cli.ts         # CLI 入口
│   │   ├── server.ts      # Bun HTTP 服务器
│   │   └── browser-manager.ts
│   ├── dist/browse        # 编译后的二进制（~58MB）
│   └── test/              # 集成测试
├── scripts/               # 构建 + 开发工具
│   ├── gen-skill-docs.ts  # 模板 → SKILL.md 生成器
│   ├── skill-check.ts     # 健康仪表板
│   └── dev-skill.ts       # 监听模式
├── test/                  # 技能验证 + 评估测试
│   ├── helpers/           # skill-parser, session-runner, llm-judge, eval-store
│   ├── fixtures/          # 基准数据、植入 Bug 测试
│   ├── skill-validation.test.ts   # Tier 1: 静态验证（免费）
│   ├── skill-e2e.test.ts          # Tier 2: E2E 测试（付费）
│   └── skill-llm-eval.test.ts     # Tier 3: LLM 评判（付费）
├── [13 个技能目录]         # 每个目录包含 SKILL.md.tmpl
├── docs/                  # 文档
├── bin/                   # 辅助脚本
├── SKILL.md.tmpl          # 主 CLI 技能模板
└── package.json           # 版本 0.3.3
```

### 关键设计决策：守护进程模型

- 首次调用自动启动 Chromium 守护进程（约 3 秒）
- 后续调用仅需约 100–200ms（通过 HTTP POST 到 localhost）
- 持久化状态：Cookie、标签页、localStorage、登录会话在命令间保持
- 空闲 30 分钟自动关闭
- 随机端口选择（10000–60000），防止并行工作区冲突

---

## 三、13 个技能/斜杠命令详解

### 阶段 1：规划与设计（4 个技能）

| 技能 | 角色 | 功能描述 |
|------|------|----------|
| `/plan-ceo-review` | 创始人模式审查 | 重新思考问题，寻找 10 星级产品。四种模式：扩展、选择性扩展、保持范围、缩减。使用 Bezos、Grove、Munger 等 14 种 CEO 认知模式 |
| `/plan-eng-review` | 工程经理审查 | 锁定执行计划：架构、数据流、图表、边界情况、测试覆盖。使用 15 种工程认知模式。生成测试计划供 QA 消费 |
| `/plan-design-review` | 高级设计师审计（仅报告） | 80 项设计审计，覆盖 10 个类别。包含 AI 风格检测（识别 10 种常见 AI 生成设计模式）。输出设计评分 A–F |
| `/design-consultation` | 从零设计系统构建 | 研究竞品、提出安全选择与创意冒险、生成 DESIGN.md + 模型图 |

### 阶段 2：实现（2 个技能）

| 技能 | 角色 | 功能描述 |
|------|------|----------|
| `/review` | PR 预合并审查 | 两遍审查：关键（SQL 安全、竞态条件、LLM 信任边界）+ 信息性（副作用、魔法数字、死代码）。自动修复机械性问题，设计决策则询问用户 |
| `/ship` | 全自动发布工作流 | 拉取并合并基础分支 → 运行测试 → 审查差异（调用 /review）→ 版本号 → CHANGELOG → 推送 → 创建 PR。全程非交互式 |

### 阶段 3：质量保证（4 个技能）

| 技能 | 角色 | 功能描述 |
|------|------|----------|
| `/qa` | 测试 + 修复 + 验证 | 6 阶段 QA 流程，三种深度（Quick/Standard/Exhaustive）。发现 Bug → 源码修复（原子提交）→ 实时验证 → 自动生成回归测试 |
| `/qa-only` | 仅报告 QA | 与 /qa 相同方法论，但只报告不修复 |
| `/qa-design-review` | 设计师级代码修复 | 80 项设计审计 + 迭代修复源代码中的设计问题，生成前后对比截图 |
| `/setup-browser-cookies` | 会话管理 | 从真实浏览器（Chrome、Arc、Brave 等）导入 Cookie，支持加密解密 |

### 阶段 4：发布与文档（2 个技能）

| 技能 | 角色 | 功能描述 |
|------|------|----------|
| `/retro` | 工程回顾 | 按人员分析贡献、代码质量、发布速度。支持时间范围参数（7d/24h/14d/compare） |
| `/document-release` | 技术写作 | 读取所有文档文件，交叉引用差异，自动更新文档以匹配已发布内容 |

### 阶段 5：工具与维护（1 个技能）

| 技能 | 角色 | 功能描述 |
|------|------|----------|
| `/gstack-upgrade` | 更新管理 | 检查新版本、智能延迟提醒、自动升级模式、同步项目中的 vendored 副本 |

---

## 四、无头浏览器（/browse）

### 命令分类（50+ 命令）

| 类别 | 命令 |
|------|------|
| **导航** | `goto`, `back`, `forward`, `reload`, `url` |
| **读取** | `text`, `html`, `links`, `forms`, `accessibility` |
| **检查** | `js`, `eval`, `css`, `attrs`, `is`, `console`, `network`, `dialog`, `cookies`, `storage`, `perf` |
| **交互** | `click`, `fill`, `select`, `hover`, `type`, `press`, `scroll`, `wait`, `upload`, `viewport` 等 |
| **视觉** | `screenshot`（支持元素裁剪、区域剪切）、`pdf`、`responsive`（手机/平板/桌面）、`diff` |
| **标签页** | `tabs`, `tab`, `newtab`, `closetab` |
| **服务器** | `status`, `stop`, `restart` |
| **元数据** | `snapshot`, `chain`, `help` |

### Snapshot 系统（核心特性）

基于无障碍树的 @ref 引用系统，8 种标志位：

| 标志 | 功能 |
|------|------|
| `-i` | 仅显示可交互元素（按钮、输入框、链接） |
| `-c` | 紧凑模式（无空节点） |
| `-d N` | 限制深度 |
| `-s SEL` | 限定 CSS 选择器范围 |
| `-D` | 与上次快照差异对比 |
| `-a` | 带标注的截图（覆盖框） |
| `-o PATH` | 截图输出路径 |
| `-C` | 光标可交互元素（@c 引用） |

### Ref 系统设计原则

- 使用 Playwright Locators（DOM 外部），不修改 DOM
- 通过 `page.locator().ariaSnapshot()` 获取无障碍树
- 无 CSP 冲突、无框架冲突、无 Shadow DOM 问题

---

## 五、模板生成系统

### SKILL.md.tmpl → SKILL.md 流水线

1. **源文件：** `.tmpl` 文件是包含 `{{PLACEHOLDERS}}` 的 Markdown 模板
2. **生成器：** `gen-skill-docs.ts` 读取源码 + 模板，生成 `.md` 文件
3. **主要占位符：**

| 占位符 | 解析内容 |
|--------|----------|
| `{{PREAMBLE}}` | 更新检查 + 会话跟踪 + 贡献者模式 |
| `{{COMMAND_REFERENCE}}` | 浏览器命令表（来自 commands.ts） |
| `{{SNAPSHOT_FLAGS}}` | 快照标志表（来自 snapshot.ts） |
| `{{BASE_BRANCH_DETECT}}` | 分支检测 bash 代码 |
| `{{QA_METHODOLOGY}}` | 共享 QA 方法论（注入 /qa + /qa-only） |
| `{{DESIGN_METHODOLOGY}}` | 设计审计方法论 |
| `{{BROWSE_SETUP}}` | 二进制搜索设置代码块 |
| `{{TEST_BOOTSTRAP}}` | 测试框架引导代码 |
| `{{REVIEW_DASHBOARD}}` | 审查就绪仪表板读取器 |
| `{{DESIGN_REVIEW_LITE}}` | /review 的 20 项设计检查清单 |

### 模板编写规则

- 使用自然语言传递状态，不使用 Shell 变量跨代码块传递
- 每个 Bash 代码块独立运行（变量不持久化）
- 动态检测分支名（不硬编码）
- 条件逻辑用英文表达，不用嵌套 if/else

---

## 六、测试策略：三层验证

| 层级 | 类型 | 成本 | 速度 | 内容 |
|------|------|------|------|------|
| **Tier 1** | 静态验证 | 免费 | <1s | 43+ 单元测试，解析所有 `$B` 命令，验证命令注册表 + 快照标志一致性 |
| **Tier 2** | E2E 测试 | ~$3.85/次 | 分钟级 | 通过 Claude Code SDK 启动真实会话，端到端运行技能 |
| **Tier 3** | LLM 评判 | ~$0.15/次 | 秒级 | LLM 对生成的 SKILL.md 进行清晰度/完整度/可操作性评分（≥4/5 通过） |

### 差异化测试选择

- 每个测试在 `test/helpers/touchfiles.ts` 中声明其文件依赖
- 仅运行受当前 `git diff` 影响的测试
- 全局文件变更（session-runner、eval-store 等）触发所有测试
- 使用 `EVALS_ALL=1` 或 `:all` 脚本强制运行所有测试

---

## 七、开发命令速查

```bash
bun install              # 安装依赖
bun test                 # 运行免费测试（Tier 1）
bun run test:evals       # 运行付费评估（Tier 2+3，基于 diff，~$4/次）
bun run test:evals:all   # 强制运行所有付费评估
bun run test:e2e         # 仅 E2E 测试（基于 diff）
bun run test:e2e:all     # 全部 E2E 测试
bun run eval:select      # 预览哪些测试会运行
bun run build            # 生成文档 + 编译二进制
bun run gen:skill-docs   # 重新生成 SKILL.md 文件
bun run skill:check      # 技能健康仪表板
bun run dev:skill        # 监听模式：自动重新生成 + 验证
bun run dev <cmd>        # 开发模式运行 CLI
```

---

## 八、路线图（优先级概要）

### P1（即将发布）
- 从 `/ship` 自动调用 `/document-release`

### P2（近期）
- 发布日志（持久化 /ship 运行记录供 /retro 使用）
- 部署后验证（浏览 staging、截图、错误检查）
- PR 中的视觉标注（在 PR 正文中嵌入截图）
- 内联 PR 标注（通过 gh api 在 file:line 级别评论）
- CI/CD QA 集成（QA 健康分数下降时阻止 PR）
- `/merge` 技能（基于审查门控的 PR 合并）

### P3（长期）
- 浏览器会话隔离
- 视频录制
- 状态持久化（保存/加载 Cookie + localStorage）
- 加密凭证保险库（LLM 永远不可见密码）
- 完整性指标仪表板
- E2E 可观测性仪表板

### P4（远期，锦上添花）
- Iframe 支持、语义定位器、设备模拟预设、网络模拟、下载处理、WebSocket 实时预览、CDP 模式、Linux/Windows Cookie 解密

---

## 九、核心设计原则

1. **模板系统而非硬编码逻辑** — 技能是 Markdown 模板，不是可执行代码，通过占位符注入实现单一真实来源
2. **唯一真实来源** — 浏览器命令在 `commands.ts` 中定义一次，自动注入到所有文档和验证中
3. **Ref 系统不修改 DOM** — 使用 Playwright Locators，无 CSP/框架/Shadow DOM 冲突
4. **项目状态持久化** — 审查决策、CEO 计划、测试计划、评估结果均持久化存储
5. **完整性优先** — AI 使全面实现变得廉价，不推荐偷工减料（"煮沸整个湖"理念）
6. **E2E 故障归因协议** — 声称失败"与本次变更无关"前，必须在 main 分支上复现证明

---

## 十、总结

gstack 是一次 **AI 原生软件工程的实验**——验证一个假设：借助 Claude Code 作为力量倍增器，一个人能否达到以往需要二十人团队的发布速度？

系统通过以下方式工作：
1. 赋予 Claude **持久化浏览器访问能力**（首次调用后每次仅约 100ms）
2. 将工作流组织为**专业角色**（CEO、工程经理、设计师、QA 主管、发布工程师、技术写作）
3. 使决策**显式化**（每个建议都附带选项和推理）
4. **状态在技能间流转**（Cookie、登录会话、测试计划、审查决策）
5. **偏好完整实现**（AI 使全面实现变得廉价，捷径是虚假的经济）

整个系统免费、MIT 许可、开源、从源码自动生成文档（不可能产生漂移）、三层测试覆盖，已被 Garry Tan 在 YC 日常使用，每周产出 10K+ 行代码。

**它不是副驾驶，它是一支团队。**
