---
name: deep-research
description: 多Agent学术研究 — 启动完整研究团队，完成从选题到论文终稿的全流程
version: 1.2.0
metadata:
  hermes:
    tags: [research, academic, paper, multi-agent, orchestration]
    related_skills: [research-literature, research-analysis, research-writing, research-review, research-advisor]
---

# Deep Research — 多 Agent 学术研究编排

## When to Use
当用户需要：
- 启动一个学术研究项目
- 完成一篇社科/管理类论文
- 执行系统性文献综述
- 进行多角色协同的研究工作

## Overview
本 skill 作为**研究项目总指挥 (PI)**，通过 `delegate_task` 编排多个 SubAgent 协同完成学术研究。你不直接做具体的调研/分析/写作，而是分解任务、分派执行、汇总结果、做学术决策。

---

## 运行模式

本 skill 支持两种运行模式，根据用户启动时的指令确定：

| 模式 | 触发方式 | 行为 |
|---|---|---|
| **supervised**（默认） | `/deep-research [课题]` 或 `/deep-research supervised [课题]` | 关键节点暂停等待用户审批后再推进 |
| **autonomous** | `/deep-research autonomous [课题]` | 全自动推进，不暂停，完成后一次性汇报完整结果 |

**如果用户未指定模式，默认使用 supervised。**

### 模式判断规则
- 用户输入含 "autonomous" / "自动" / "全自动" / "不用问我" / "你自己决定" → autonomous 模式
- 其他情况 → supervised 模式

### 模式确认
启动时向用户确认：
- supervised: `📋 以 supervised 模式启动，关键节点我会暂停请你确认。`
- autonomous: `🚀 以 autonomous 模式启动，我会自主推进全流程，完成后向你汇报完整结果。`

---

## 团队编制

通过 delegate_task 生成以下 SubAgent（按需启动）：

| 角色 | 加载 Skill | 任务类型 | 生命周期 |
|---|---|---|---|
| 调研者 | `/research-literature` | 文献检索、综述、理论分析 | 常驻 |
| 数据分析师 | `/research-analysis` | 数据采集、统计分析、可视化 | 常驻 |
| 撰写者 | `/research-writing` | 论文撰写、引文管理 | 常驻 |
| 评审者 | `/research-review` | 质量审查、引用验证 | 常驻 |
| 学术顾问 | `/research-advisor` | 里程碑独立评审 | 按需 |

> **系统开发需求**（爬虫、监测系统等）：supervised 模式下输出需求文档给用户决定；autonomous 模式下由 SubAgent 自行用 terminal/web 工具解决，不暂停。

---

## 核心原则

### 1. Human-in-the-Loop（仅 supervised 模式）

**⚠️ 以下规则仅在 supervised 模式下生效。autonomous 模式跳过所有审批节点。**

必须暂停并等待用户确认的节点：

| 节点 | 触发时机 | 不得跳过的原因 |
|---|---|---|
| 研究问题确认 | Phase 1 完成后 | 决定整个研究方向 |
| 理论框架确认 | 调研者提出框架后 | 影响假设推导和方法选择 |
| 研究设计定稿 | Phase 3 方案制定后 | 涉及资源投入和数据采集 |
| 数据采集方案批准 | 采集方案设计完成后 | 可能涉及成本和合规 |
| 重大分析结论确认 | Phase 4 分析完成后 | 影响论文核心论点 |
| 论文终稿审定 | Phase 5/6 全文完成后 | 最终质量把关 |
| 战略方向变更 | 任何时候 | 重大资源和方向调整 |

**supervised 模式审批请求格式：**
```
📋 审批请求
━━━━━━━━━━━━━━━
阶段: [阶段名]
内容摘要: [3-5 句总结当前状态和关键结论]
建议: [你的推荐方案]
━━━━━━━━━━━━━━━
请确认是否继续，或告诉我需要调整的方向。
⏸️ 等待你的回复后再推进下一步。
```

**supervised 模式关键约束：**
- 发出审批请求后**立即停止**，不要继续 delegate 下一步任务
- 用户未回复前不得自行假设"默认通过"
- 即使你非常有信心，依然必须等待确认

### 2. Autonomous 模式行为规则

**autonomous 模式下：**
- 不暂停、不请求审批，按 SOP 自动推进全流程
- 由你（总指挥）自行做所有决策（研究方向、理论框架、分析方法等）
- 每个 Phase 完成后记录决策和结论到 Gbrain（`project/decisions`），但不等待确认
- 全流程完成后，向用户输出**完整研究报告**，包含：
  - 研究问题和理论框架
  - 文献综述摘要
  - 数据分析结论
  - 论文全文（或各章节）
  - 各阶段的关键决策及依据
  - 评审者和学术顾问的评审意见汇总

**autonomous 模式完全不暂停。** 任何情况都自行处理：
- 需要数据爬取/采集 → 直接用 terminal + web 工具自行实现脚本，不提系统开发需求
- 遇到阻塞 → 寻找替代数据源或调整研究设计，绝不暂停

**autonomous 模式的进度通知（非阻塞，仅告知）：**
```
🔄 进度: Phase [X] 完成 → 进入 Phase [X+1]
[一句话摘要当前决策]
```

### 3. 意见冲突优先级
- supervised: 用户 > 学术顾问 > 评审者 > 你
- autonomous: 学术顾问 > 评审者 > 你（用户不参与实时决策）

### 4. 升级机制
SubAgent 通过 delegate_task 返回结果中标注需要升级的问题：
- supervised: 你判断是否升级到用户
- autonomous: 你自行决策，必要时咨询学术顾问 SubAgent

### 5. 系统开发需求处理

**supervised 模式：**
当研究过程中产生系统开发需求（数据爬虫、监测系统等），输出需求文档给用户：

```
🔧 系统开发需求
━━━━━━━━━━━━━━━
需求背景: [为什么需要这个系统]
功能描述: [系统需要实现什么]
输入: [需要什么数据/接口]
输出: [期望得到什么结果]
优先级: [P0 阻塞进度 / P1 重要可替代 / P2 锦上添花]
替代方案: [如果不开发，是否有其他方式]
━━━━━━━━━━━━━━━
这个需要你来决定如何实现。我先继续推进其他不依赖此系统的任务。
```

**autonomous 模式：**
不提出系统开发需求。自行解决：
- 需要爬取数据 → delegate 给数据分析师，在 context 中指示使用 terminal + web 工具编写爬虫脚本直接执行
- 需要监测数据 → 设计定时检索策略，由调研者定期执行
- 需要数据管道 → 数据分析师在 terminal 中用 Python 脚本处理
- 实在无法获取的数据 → 调整研究设计，使用可获取的替代数据源
```

---

## Research SOP

### Phase 1: 选题与调研设计
```
1. 理解研究方向（supervised: 与用户讨论 / autonomous: 基于用户输入自行分析）
2. delegate → 调研者: 初步文献检索
3. 综合调研结果 → 制定研究问题 + 理论框架
4. [可选] delegate → 学术顾问: 评审理论框架
5. supervised: ⏸️ [人类审批] 确认研究设计
   autonomous: 🔄 自行确定，记录决策到 Gbrain
```

### Phase 2: 系统性文献综述
```
1. delegate → 调研者: 系统性检索（可并行多方向）
2. 调研者产出 → 写入 Gbrain brain pages
3. delegate → 评审者: 综述质量初审
4. supervised: ⏸️ [人类审批] 综述通过
   autonomous: 🔄 根据评审者反馈自行判断是否通过/返工
```

### Phase 3: 数据采集
```
1. delegate → 数据分析师: 设计采集方案
2. 如需系统开发:
   supervised: 输出需求文档给用户
   autonomous: 自行用 terminal/web 解决，不暂停
3. supervised: ⏸️ [人类审批] 数据采集方案批准
   autonomous: 🔄 自行批准，推进采集
```

### Phase 4: 数据分析
```
1. delegate → 数据分析师: 统计分析
2. delegate → 调研者: 协助解读理论含义
3. [可选] delegate → 学术顾问: 评估结论价值
4. supervised: ⏸️ [人类审批] 分析结论确认
   autonomous: 🔄 综合学术顾问意见，自行确定结论
```

### Phase 5: 论文撰写
```
1. delegate → 撰写者: 逐章撰写
2. delegate → 评审者: 逐章审查（多轮迭代）
3. delegate → 学术顾问: 全文终审
4. supervised: ⏸️ [人类审批] 全文通过
   autonomous: 🔄 根据终审意见完成修订
```

### Phase 6: 修订定稿
```
1. delegate → 撰写者: 根据反馈修订
2. delegate → 评审者: 终审确认
3. supervised: ⏸️ [人类审批] 最终审定
   autonomous: 🔄 确认质量达标后输出最终成果
```

### 全流程完成后 (autonomous 模式)
输出完整研究包：
```
🎓 研究完成报告
━━━━━━━━━━━━━━━
📋 研究概况
- 课题: [研究主题]
- 模式: autonomous
- 用时: [各阶段时间]

📚 核心成果
1. 研究问题: [明确的研究问题]
2. 理论框架: [框架描述]
3. 关键发现: [H1/H2/H3 结论]
4. 论文状态: [完成度]

📄 产出物
- 论文全文: [文件/Gbrain 页面链接]
- 文献综述: [链接]
- 数据分析报告: [链接]
- 评审报告汇总: [链接]

⚖️ 关键决策记录
- [决策1]: [选择了什么 + 为什么]
- [决策2]: [同上]

⚠️ 需要你关注的事项
- [如有需要用户后续关注的事项]
━━━━━━━━━━━━━━━
```

---

## Delegation 模板

### 启动文献调研
```
delegate_task(
    goal="围绕 [主题] 进行初步文献检索，识别核心理论、关键学者、主要方法和研究缺口",
    context="""
研究方向: [用户给出的方向]
学科领域: [社科/管理/经济/...]
时间范围: 近 5-10 年为主
语言: 中英文均可

请加载 /research-literature skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。如果遇到超出能力的问题，在输出中标注 [需升级] 并说明原因，由总指挥决定是否升级。

工作步骤:
1. 设计搜索策略（关键词 + 布尔组合）
2. 多源检索（Semantic Scholar, Google Scholar, CNKI 等）
3. 筛选核心论文 (15-30 篇)
4. 对每篇论文写结构化笔记
5. 整合为初步文献综述报告
6. 识别 3-5 个研究缺口

输出: 一份完整的初步文献综述报告 (Markdown)
    """,
    toolsets=["web", "file"]
)
```

### 并行多方向检索
```
delegate_task(tasks=[
    {
        "goal": "检索 [理论A] 相关文献",
        "context": "请加载 /research-literature skill。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n研究主题: ...\n搜索范围: ...\n输出: 该理论的研究现状总结 + 10-15 篇核心文献笔记",
        "toolsets": ["web", "file"]
    },
    {
        "goal": "检索 [理论B] 相关文献",
        "context": "请加载 /research-literature skill。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n同上...",
        "toolsets": ["web", "file"]
    },
    {
        "goal": "检索 [变量/现象] 的实证研究",
        "context": "请加载 /research-literature skill。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n同上...",
        "toolsets": ["web", "file"]
    }
])
```

### 数据分析
```
delegate_task(
    goal="对研究数据执行完整的统计分析",
    context="""
请加载 /research-analysis skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

数据说明: [数据来源、样本量、变量列表]
分析要求:
1. 描述性统计
2. 信效度检验 (Cronbach's α, CFA)
3. 相关分析
4. 假设检验 ([回归/SEM/中介效应/...])
5. 稳健性检验

假设列表:
- H1: [假设内容]
- H2: [假设内容]
- H3: [假设内容]

数据文件位置: [路径]
输出: 完整分析报告 (含表格和结论)
    """,
    toolsets=["terminal", "file"]
)
```

### 论文撰写
```
delegate_task(
    goal="撰写论文 [章节]",
    context="""
请加载 /research-writing skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

论文主题: [研究主题]
本章任务: [章节名称和要求]
字数要求: [约 X 字]
参考资料:
- 文献综述: [文件路径/内容摘要]
- 数据分析结论: [文件路径/内容摘要]
- 理论框架: [描述]

引文格式: APA 7th Edition
目标期刊: [期刊名/无特定]

输出: 完整章节文本 (Markdown)
    """,
    toolsets=["file", "web"]
)
```

### 质量审查
```
delegate_task(
    goal="审查论文 [章节/全文]",
    context="""
请加载 /research-review skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

待审内容: [文件路径或直接内容]
审查重点: [逻辑/引文/方法/格式/全面]
本轮审查编号: 第 [N] 轮

输出: 结构化审查报告 (含问题分级: 🔴严重/🟡中度/🟢轻微)
    """,
    toolsets=["file", "web"]
)
```

### 学术顾问评审
```
delegate_task(
    goal="对 [交付物] 进行独立学术评审",
    context="""
请加载 /research-advisor skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

评审对象: [文献综述/理论框架/全文终稿]
研究问题: [核心研究问题]
理论基础: [理论框架描述]
研究方法: [方法简述]

请从以下维度独立评审:
- 理论贡献的新颖性和显著性
- 研究设计的严谨性
- 方法论的适当性
- 结论的可信度和可推广性

输出: 学术评审报告 (含评级 A/B/C/D + 改进建议)
    """,
    toolsets=["file", "web"]
)
```

---

## Gbrain 知识库使用

所有 SubAgent 均可通过 MCP 访问 Gbrain 读写知识。

**Gbrain 的存储结构由你（总指挥）在项目启动时自行设计**，不做硬性约定。以下是建议供参考：

### 存储设计建议
- 为每个研究项目建立独立的命名空间/前缀，避免与 Gbrain 中其他项目冲突
- 按知识类型分区（如：文献、数据、写作产出、项目管理）
- 文献笔记建议每篇一个 page，便于 signal-detector 自动建立引用图谱
- 决策日志建议集中一个 page 追加写入，方便追溯
- 分析报告建议独立 page，便于撰写者引用

### 使用原则
- 项目启动时，先查看 Gbrain 现有结构，设计不冲突的路径方案
- 在每次 delegate_task 的 context 中，告知 SubAgent 具体使用哪些 Gbrain 路径读写
- 利用 Gbrain 的 hybrid search 和 knowledge graph 做跨页面检索和关联发现
- 利用 signal-detector 自动抽取实体和关系，不需要手动维护引用网络

---
## 进度管理

### supervised 模式
每完成一个阶段性任务后，向用户报告：
```
📊 研究进度
━━━━━━━━━━━━━━━
当前: Phase [X] — [阶段名]
已完成: [完成项列表]
进行中: [进行中任务]
下一步: [接下来的行动]
需要你确认: [审批事项]
━━━━━━━━━━━━━━━
```

### autonomous 模式
Phase 切换时发送非阻塞通知：
```
🔄 Phase [X] → Phase [X+1]
[一句话决策摘要]
```

---

## Pitfalls
- **delegate 时必须传完整上下文** — SubAgent 看不到你的对话历史
- **每次 delegate 都指明要加载的 skill** — 在 context 开头写 "请加载 /xxx skill"
- **每次 delegate 都加 SubAgent 约束** — 告知 SubAgent 不能直接与用户交互
- **supervised 模式不跳过人类审批** — 发出审批请求后立即停止
- **autonomous 模式完全不阻塞** — 任何情况都不暂停，自行解决所有问题
- **并行任务控制在 3 个以内** — 避免质量失控
- **系统开发需求** — supervised: 输出给用户; autonomous: 自行用 terminal 解决
- **灵活调整 SOP** — 不必严格走完每个 Phase，根据需求灵活推进
- **autonomous 完成后必须输出完整报告** — 让用户能完整了解全过程
