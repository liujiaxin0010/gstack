---
name: design-methodology
description: 设计审计方法论。6 阶段设计评审流程，含 80+ 项检查清单和 AI 泛化检测。供设计审计/修复技能引用。
---

# 设计审计方法论

## 审计模式

### 完整模式（默认）
系统化审查从首页可达的所有页面。访问 5-8 个页面。完整清单评估、响应式截图、交互流测试。生成完整的设计审计报告和字母等级评分。

### 快速模式（`--quick`）
仅首页 + 2 个关键页面。第一印象 + 设计系统提取 + 简化清单。最快获得设计评分的路径。

### 深度模式（`--deep`）
全面审查：10-15 个页面，每个交互流，详尽的清单。用于上线前审计或大改版。

### 差异感知模式（在特性分支上且无 URL 时自动启用）
1. 分析分支差异：`git diff main...HEAD --name-only`
2. 将变更文件映射到受影响的页面/路由
3. 检测本地运行的应用（端口 3000、4000、8080）
4. 仅审计受影响的页面，比较变更前后的设计质量

### 回归模式（`--regression` 或找到之前的 `design-baseline.json`）
运行完整审计，然后加载之前的基线。比较：各类别等级差异、新发现、已解决的发现。在报告中输出回归表。

---

## 阶段 1：第一印象

最具设计师特色的输出。在分析任何东西之前形成直觉反应。

1. 导航到目标 URL
2. 拍全页桌面截图：`$B screenshot "$REPORT_DIR/screenshots/first-impression.png"`
3. 使用结构化评价格式写**第一印象**：
   - "这个网站传达了**[什么]**。"（一眼看到什么——专业？趣味？混乱？）
   - "我注意到**[观察]**。"（突出的东西，正面或负面——要具体）
   - "我的视线首先落在这 3 个地方：**[1]**、**[2]**、**[3]**。"（层级检查——这些是有意的吗？）
   - "如果必须用一个词描述：**[词]**。"（直觉判断）

这是用户首先阅读的部分。要有主见。设计师不含糊——他们反应。

---

## 阶段 2：设计系统提取

提取网站实际使用的设计系统（不是 DESIGN.md 说的，而是实际渲染的）：

```bash
# 使用的字体（限制 500 个元素避免超时）
$B js "JSON.stringify([...new Set([...document.querySelectorAll('*')].slice(0,500).map(e => getComputedStyle(e).fontFamily))])"

# 使用的颜色调色板
$B js "JSON.stringify([...new Set([...document.querySelectorAll('*')].slice(0,500).flatMap(e => [getComputedStyle(e).color, getComputedStyle(e).backgroundColor]).filter(c => c !== 'rgba(0, 0, 0, 0)'))])"

# 标题层级
$B js "JSON.stringify([...document.querySelectorAll('h1,h2,h3,h4,h5,h6')].map(h => ({tag:h.tagName, text:h.textContent.trim().slice(0,50), size:getComputedStyle(h).fontSize, weight:getComputedStyle(h).fontWeight})))"

# 触摸目标审计（找到尺寸不足的交互元素）
$B js "JSON.stringify([...document.querySelectorAll('a,button,input,[role=button]')].filter(e => {const r=e.getBoundingClientRect(); return r.width>0 && (r.width<44||r.height<44)}).map(e => ({tag:e.tagName, text:(e.textContent||'').trim().slice(0,30), w:Math.round(e.getBoundingClientRect().width), h:Math.round(e.getBoundingClientRect().height)})).slice(0,20))"

# 性能基线
$B perf
```

将发现结构化为**推断的设计系统**：
- **字体：** 列表及使用次数。如果超过 3 种不同的字体族则标记。
- **颜色：** 提取的调色板。如果超过 12 种非灰色唯一颜色则标记。注明暖色/冷色/混合。
- **标题级别：** h1-h6 大小。标记跳过的层级、非系统化的大小跳跃。
- **间距模式：** 采样 padding/margin 值。标记非比例值。

提取后提供："Boss，要我把这个保存为你的 DESIGN.md 吗？我可以将这些观察锁定为项目的设计系统基线。"

---

## 阶段 3：逐页视觉审计

对每个在范围内的页面：

```bash
$B goto <url>
$B snapshot -i -a -o "$REPORT_DIR/screenshots/{page}-annotated.png"
$B responsive "$REPORT_DIR/screenshots/{page}"
$B console --errors
$B perf
```

### 认证检测

首次导航后，检查 URL 是否变成了登录路径：
```bash
$B url
```
如果 URL 包含 `/login`、`/signin`、`/auth` 或 `/sso`：网站需要认证。告知 Boss："Boss，这个网站需要认证。要导入浏览器的 cookies 吗？"

### 设计审计清单（10 类，约 80 项）

在每个页面应用。每个发现获得影响评级（高/中/打磨）和类别。

**1. 视觉层级与构图**（8 项）
- 有清晰的焦点吗？每个视图一个主要 CTA？
- 视线自然地从左上到右下流动？
- 视觉噪音——竞争注意力的元素？
- 信息密度适合内容类型？
- Z-index 清晰——没有意外重叠？
- 首屏内容 3 秒内传达目的？
- 模糊测试：模糊后层级仍然可见？
- 白空间是有意的，不是剩余的？

**2. 排版**（15 项）
- 字体数量 ≤ 3（超过则标记）
- 比例遵循比率（1.25 大三度或 1.333 纯四度）
- 行高：正文 1.5x，标题 1.15-1.25x
- 度量：每行 45-75 字符（66 理想）
- 标题层级：无跳过的级别（h1→h3 没有 h2）
- 字重对比：使用 ≥ 2 种字重来建立层级
- 无黑名单字体（Papyrus、Comic Sans、Lobster、Impact、Jokerman）
- 如果主字体是 Inter/Roboto/Open Sans/Poppins → 标记为可能过于通用
- 正文 ≥ 16px
- 标题/标签 ≥ 12px
- 小写文字无字间距

**3. 颜色与对比**（10 项）
- 调色板连贯（≤ 12 种非灰色唯一颜色）
- WCAG AA：正文 4.5:1，大文字（18px+）3:1，UI 组件 3:1
- 语义颜色一致（成功=绿，错误=红，警告=黄/琥珀）
- 无仅颜色编码（始终添加标签、图标或图案）
- 深色模式：表面使用高程，不仅仅是亮度反转
- 深色模式：文字偏白（~#E0E0E0），不是纯白
- 无红/绿仅组合（8% 男性有红绿色盲）

**4. 间距与布局**（12 项）
- 网格在所有断点一致
- 间距使用比例（4px 或 8px 基础），不是任意值
- 对齐一致——没有在网格外浮动
- 节奏：相关项更近，不同部分更远
- 圆角层级（不是所有东西统一的圆润圆角）
- 内圆角 = 外圆角 - 间距（嵌套元素）
- 移动端无水平滚动
- 设置了最大内容宽度（无全幅正文）
- URL 反映状态（筛选器、标签、分页在查询参数中）

**5. 交互状态**（10 项）
- 所有交互元素有悬停状态
- 有 `focus-visible` 环（永远不要 `outline: none` 而无替代）
- 激活/按下状态有深度效果或颜色变化
- 禁用状态：降低透明度 + `cursor: not-allowed`
- 加载中：骨架形状匹配真实内容布局
- 空状态：温暖的消息 + 主要操作 + 视觉（不仅仅是"无项目。"）
- 错误消息：具体 + 包含修复/下一步
- 成功：确认动画或颜色，自动消失
- 触摸目标 ≥ 44px
- 所有可点击元素 `cursor: pointer`

**6. 响应式设计**（8 项）
- 移动端布局有*设计*意义（不仅仅是堆叠桌面列）
- 移动端触摸目标足够（≥ 44px）
- 任何视口无水平滚动
- 图片处理响应式
- 移动端文字无需缩放即可阅读（≥ 16px）
- 导航适当折叠
- 表单在移动端可用
- 视口 meta 无 `user-scalable=no`

**7. 动效与动画**（6 项）
- 缓动：进入用 ease-out，退出用 ease-in，移动用 ease-in-out
- 持续时间：50-700ms 范围
- 目的：每个动画传达某些东西
- 尊重 `prefers-reduced-motion`
- 无 `transition: all`——属性明确列出
- 仅动画 `transform` 和 `opacity`（不是布局属性）

**8. 内容与微文案**（8 项）
- 空状态有温度设计（消息 + 操作 + 图标）
- 错误消息具体：发生了什么 + 为什么 + 怎么办
- 按钮标签具体（"保存 API 密钥" 不是 "继续" 或 "提交"）
- 无占位符/lorem ipsum 文本
- 截断处理（`text-overflow: ellipsis`、`line-clamp`）
- 主动语态
- 加载状态以 `…` 结束（"保存中…" 不是 "保存中..."）
- 破坏性操作有确认对话框或撤销窗口

**9. AI 泛化检测**（10 项反模式——黑名单）

测试：一个受尊敬的工作室的人类设计师会发布这个吗？

- 紫色/紫罗兰/靛蓝渐变背景或蓝到紫的配色方案
- **3 列特性网格：** 彩色圆圈中的图标 + 粗体标题 + 2 行描述，对称重复 3 次。最可辨识的 AI 布局。
- 彩色圆圈中的图标作为区域装饰
- 所有东西居中
- 所有元素统一的圆润 border-radius
- 装饰性色块、浮动圆圈、波浪 SVG 分隔符
- Emoji 作为设计元素
- 卡片左侧彩色边框
- 通用英雄文案（"Welcome to [X]"、"Unlock the power of..."）
- 千篇一律的区域节奏（英雄 → 3 个特性 → 推荐 → 定价 → CTA）

**10. 性能即设计**（6 项）
- LCP < 2.0s（Web 应用），< 1.5s（信息网站）
- CLS < 0.1
- 骨架质量：形状匹配真实内容
- 图片：`loading="lazy"`，设置 width/height，WebP/AVIF 格式
- 字体：`font-display: swap`，预连接到 CDN
- 无可见的字体闪烁（FOUT）

---

## 阶段 4：交互流评审

走 2-3 个关键用户流程，评估*感觉*，不仅仅是功能：

```bash
$B snapshot -i
$B click @e3           # 执行操作
$B snapshot -D          # 查看什么变化了
```

评估：
- **响应感觉：** 点击感觉响应吗？有延迟或缺失的加载状态吗？
- **过渡质量：** 过渡是有意的还是通用/缺失？
- **反馈清晰度：** 操作明确成功还是失败了？反馈是即时的吗？
- **表单打磨：** 焦点状态可见？验证时机正确？错误靠近来源？

---

## 阶段 5：跨页一致性

比较不同页面的截图和观察：
- 导航栏在所有页面一致？
- 页脚一致？
- 组件复用 vs 一次性设计（同一按钮在不同页面样式不同？）
- 语调一致性（一个页面活泼而另一个商务？）
- 间距节奏跨页面延续？

---

## 阶段 6：编制报告

### 评分系统

**双标题评分：**
- **设计评分：{A-F}** — 所有 10 个类别的加权平均
- **AI 泛化评分：{A-F}** — 独立评分加简短评语

**各类别等级：**
- **A：** 有意的，精致的，令人愉悦的。展示设计思维。
- **B：** 基础扎实，轻微不一致。看起来专业。
- **C：** 功能性但通用。没有大问题，没有设计观点。
- **D：** 明显问题。感觉未完成或粗心。
- **F：** 积极损害用户体验。需要重大改造。

### 基线保存

写 `design-baseline.json` 用于回归模式：
```json
{
  "date": "YYYY-MM-DD",
  "url": "<target>",
  "designScore": "B",
  "aiSlopScore": "C",
  "categoryGrades": { "hierarchy": "A", "typography": "B", "..." : "..." },
  "findings": [{ "id": "FINDING-001", "title": "...", "impact": "high", "category": "typography" }]
}
```
