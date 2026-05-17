# GBrain 文档考古报告

> 调研时间：2026-05-17
> 仓库：[garrytan/gbrain](https://github.com/garrytan/gbrain)（master 分支，v0.12.3）
> License：MIT ✅
> 调研定位：在动笔写 PROPOSAL v0.4 / PLAN v0.3 之前，把 GBrain 的设计哲学、API 现状、运维约束**一次性吃透**，避免 v1.0 probe 时出现的"按想象用 API"翻车
> 证据来源：`llms.txt` 列出的 22 份核心文档 + 5 个目录（已全部通读）
> 验证依据：`docs/stock-analysis/_gbrain_probe_logs/`（v1.0 八场景 + v1.1 五场景，共 13 份 Hermes 实测日志）

---

## 0. TL;DR — 决策者只看这一节

| 决策项 | 结论 | 依据 |
|---|---|---|
| **GBrain 是什么** | 一个**"由 Agent 维护的个人 Markdown wiki"**，Postgres 只是为了搜索/图查询而构建的派生层 | `ethos/MARKDOWN_SKILLS_AS_RECIPES.md` |
| **DB 引擎选哪个** | **PGLite 默认**（个人方案、零配置）；> 1000 个个股页或多设备访问时再上 Supabase | `ENGINES.md` |
| **写入路径** | **page-driven**：写 frontmatter + body，让 GBrain 自己抽 link / take | v1.1 probe (H4 FAIL) + `UPGRADING_DOWNSTREAM_AGENTS.md` |
| **判断怎么持久化** | 写进 frontmatter（`verdict`/`confidence`）+ timeline 一行；dream cycle 自动 → take | `takes-vs-facts.md` + v1.1 probe (H4) |
| **图怎么建** | body 里 `[name](slug)` + frontmatter `sectors:/companies:`，靠 `put_page` 的 auto-link | v1.1 probe (H1 PASS) + v0.12 changelog |
| **Source 拆几个** | 至少 3 个：`stock-analysis` / `market-research` / 默认；避免 federated 搜索串味 | `architecture/brains-and-sources.md` |
| **Hermes 怎么连** | **stdio MCP**（trusted），auto-link 才会触发；远程 MCP 需补一步 `extract links` | `mcp/DEPLOY.md` + `architecture/infra-layer.md` |
| **Cron 怎么排** | 三段式：盘后 16:00 抓数据 → 17:00 dream cycle → 次日 08:30 briefing | `guides/cron-schedule.md` + `guides/quiet-hours.md` |
| **风险底线** | 必须跑 `gbrain doctor / stats / orphans` 三件套作为部署 acceptance | `GBRAIN_VERIFY.md` |

---

## 1. 仓库整体认知（必读）

### 1.1 这不是"一个 KG 库"，是"一套 Agent 自治范式"

打开 GBrain 的第一反应是：怎么没有传统知识图谱库的那些 API（add_node / add_edge / query_path）？读完后才理解：

> "The skill file is **simultaneously documentation, specification, package, and source code**." — `ethos/MARKDOWN_SKILLS_AS_RECIPES.md`

Garry Tan 押注的核心信念：**Markdown wiki + LLM 维护，胜过数据库 schema + 业务代码**。所有的"业务逻辑"（怎么抽实体、怎么建图、怎么巩固记忆）**不写在 Python 里，写在 26 个 fat-markdown skills 里**——LLM 读 skill 文件，按 skill 描述执行。

| 层 | 内容 | 我们的关系 |
|---|---|---|
| **Markdown 文件夹** | source-of-truth | ✅ 我们的"个股页"应当物理存在于此 |
| **26 个 skill markdown** | "业务逻辑"的可读形式 | ✅ 我们应当**仿照他们的 skill 写法**写自己的股票 skill |
| **Postgres 派生层** | 索引 + 向量 + 图边 | ⚠️ 不要直接写 SQL；走 MCP 工具 |
| **63 个 MCP 工具** | Agent 与 brain 的交互面 | ✅ Hermes 通过这层调用 |
| **Thin CLI / Worker** | 调度 + cron + dream cycle | ✅ 我们用 cron 触发，不重写 |

### 1.2 "Thin Harness, Fat Skills" — 决定了我们 Skill 文件的写法

> "**The harness should never compete with the skill for control.**" — `ethos/THIN_HARNESS_FAT_SKILLS.md`

GBrain 的 skill 文件不是"配置 + 代码引用"，而是**给 LLM 看的完整剧本**：包含触发条件、检查清单、错误处理、范例输出、甚至 anti-patterns。LLM 读完就知道怎么做，不需要二次解释。

我们的股票 Skill 应当**完全照搬这种风格**：
- ❌ 不写 "调用 fetch_stock_data() 获取数据"
- ✅ 写 "输入是 6 位代码或股票名；先用 akshare 试，失败时降级 efinance；产出 `Compiled Truth` 段落必须包含…"

### 1.3 不要被 SKILL.md 数量唬到 —— 哪些跟我们直接相关

GBrain repo 有 26 个 skill。跟我们**强相关**的只有这 6 个：

| Skill | 我们怎么用 |
|---|---|
| `signal-detector` | 给 Hermes 上层用：判断"这条信息要不要触发新一轮分析" |
| `brain-ops` | 直接调（写页 / 读页 / 抽图） |
| `query` | 历史调取（"2 周前我对茅台的判断是什么"） |
| `ingest` | 从 akshare/efinance 数据流 → page 的标准管道（不用自己重发明） |
| `enrich` | 触发 dream cycle 风格的补全 |
| `maintain` | 定期 doctor / repair |

**剩下 20 个不要碰**（它们是 brain 自治用的：dream / migration / forensic / planner 等，我们当用户而不是当开发者）。

---

## 2. 心智模型：6 个必须先吃透的概念

### 2.1 双层 Page = Compiled Truth + Timeline

每个 page 物理结构：

```markdown
---
type: stock
slug: stocks/sh-600519
title: 贵州茅台 (600519)
verdict: hold
confidence: 0.62
last_verified: 2026-05-17
sectors: [baijiu, consumer-staples]
---

## Compiled Truth（被 LLM 改写覆盖）
当前画像：高 ROE 消费白马，2026 Q1 库存周期触底...
（这里是"现在如何"，是判断快照，会被覆盖）

<!-- timeline -->

## Timeline（append-only）
- **2026-05-17** | 中报预告超预期 → verdict 从 watch → hold | [Source: efinance/sh600519]
- **2026-05-10** | 渠道库存数据：[渠道反馈](evidence/channel-202605) | confidence 0.55 → 0.62
```

**两层各司其职**：
- 上半 = 被覆盖区，每次 enrichment 由 LLM **重写**，是"当前最佳认知"
- 下半 = 仅追加区，**永不删改**，是"判断的演化轨迹"

> ⚠️ **陷阱 1**：v0.12.2 之后**裸 `---` 不再是分隔符**（与 markdown HR 冲突）。必须 `<!-- timeline -->` 或 `--- timeline ---`。我们的 writer 工具如果用裸 `---`，timeline 整段会被吞进 compiled_truth，dream cycle 抽不到 take。
> 证据：`UPGRADING_DOWNSTREAM_AGENTS.md` v0.12.2 section + `skills/migrations/v0.12.0.md`

### 2.2 两轴 = Brain × Source

```
brain  : 哪一个 DB（host = 本机，team_xxx = 远端 mount）
source : DB 内的 repo（slug 在 source 内唯一）
```

GBrain 默认所有查询是 **federated**（所有 source 一起搜）。如果我们把股票数据塞进 user 的默认 source，搜"茅台"会跟用户的"茅台冰激凌购物笔记"混在一起。

**对我们的决策**：
- brain：永远 `host`（本机 PGLite）
- source 拆分：
  - `stock-analysis`：个股画像页、行业页、策略判断
  - `market-research`：宏观、政策、外部研报
  - 不动用户默认 source（保持 brain 的"个人感"不被股票污染）

> 证据：`architecture/brains-and-sources.md` + `skills/conventions/brain-routing.md`

### 2.3 Auto-link：write-time，不是 query-time

这是 v1.0 probe 翻车的根因。auto-link 实际机制（v0.12+）：

| 触发时机 | `put_page` 完成后立刻 |
|---|---|
| 触发条件 | `OperationContext.trustedWorkspace = true` |
| 扫描对象 | body 里的 `[name](slug)` markdown link + frontmatter 的已知字段 |
| 推断的 link_type | `mentions` / `belongs_to` / `attended` / `works_at` / `invested_in` / `founded` / `advises` / `source` |
| frontmatter 投影 | `sectors: [baijiu]` → `belongs_to` 边；`companies: [...]` → `mentions`；`peers: [...]` → `competes_with` |

> ⚠️ **陷阱 2**：trustedWorkspace 默认只对**本机 stdio MCP** 为 true。如果 Hermes 走远程 MCP（http/sse），auto-link 不触发。
> 解决：要么本地 stdio，要么 `put_page` 后显式补一步 `gbrain extract links --slug <s>`
> 证据：`architecture/infra-layer.md` + v1.1 probe V11-1 PASS、V11-5 FAIL

### 2.4 Takes vs Facts — 别再找"写 take 的 API"

| | **Facts**（v0.31 hot memory） | **Takes**（cold storage） |
|---|---|---|
| 写入 | 每次对话**实时**抽取 | dream cycle **夜间**从 page 抽 |
| 写 API | `extract_facts`（MCP 有） | ❌ 没有，且**永远不会有** |
| kinds | event / preference / commitment / belief / fact | take / fact / bet / hunch |
| 多持有者 | ❌ 单用户 | ✅ 多 holder（"我"、"老王"、"研报"分别持有不同 take） |
| 桥 | dream cycle `consolidate` 阶段 hot fact → cold take | |

> ⚠️ **陷阱 3**（v1.0 H4 FAIL 复盘）：所有 `take_add` / `submit_take` / `create_take` 都是 "Unknown tool"，**这不是 bug 是设计**。
>
> **正确写法**：把判断写进 frontmatter（`verdict: hold`, `confidence: 0.62`）+ timeline 一行，dream cycle 自然抽出来。
> 证据：`docs/takes-vs-facts.md` + v1.1 probe `_gbrain_probe_logs/v11_v11-3.txt`

### 2.5 Search Mode 9 格成本矩阵

> "Cost spread between corners is 25x — silent acceptance is the wrong default."

`gbrain init` 跑完会打印 **9 格矩阵**（3 search mode × 3 downstream model），最便宜与最贵差 25 倍。AGENTS.md **强制要求**：agent 必须把矩阵展示给用户、获得明确选择。

> 对我们 PLAN：Hermes 启动 GBrain 的脚本中，**不能默认跳过 search mode 选择**。这是 acceptance 的硬条件。

### 2.6 Dream Cycle 是 brain 的"睡眠"

> "Without it, signal leaks out of every conversation. With it, nothing is lost." — `guides/cron-schedule.md`

Dream cycle 不是可选 cron job，是**brain 不腐化的必要条件**。每晚做 4 件事：
1. **Entity scan** — 扫描所有今日改动的 page，抽实体
2. **Thin pages** — 找出"标题 + 一行内容"的稀薄页，触发 enrichment 补全
3. **Reference repair** — 修复死链
4. **Consolidation** — hot facts → cold takes，重写 Compiled Truth

> 对我们：A 股 16:00 收盘 → 17:00 跑 dream cycle，把当日所有"判断"巩固成 takes，第二天 08:30 briefing 时就能用 take history。

---

## 3. API 现状对账（哪些能用、哪些不能用、哪些要绕）

基于 `_gbrain_probe_logs/` 的 13 份实测日志和 MCP tool schema dump：

### ✅ 验证可用（可以放心写进我们方案）

| API | 用法 | 我们的用途 |
|---|---|---|
| `put_page` | 写 frontmatter + body 双层结构 | 个股画像、行业页、判断快照 |
| `get_page` / `read_page` | 按 slug 读取 | 读历史画像 |
| `add_link(from, to, type, context)` | 显式建图（非 auto-link 场景） | 远程 MCP 时的补救路径 |
| `get_links` / `traverse_graph(slug, depth, link_type, direction)` | 图遍历，**支持 link_type 精确过滤** | "茅台所属行业的所有同业" |
| `add_tag` | `:` 命名空间 tag（如 `source:akshare`、`stage:1`） | 数据来源标记、阶段分类 |
| `list_pages(tag=...)` | 按 tag 精确列出 | "本周所有 verdict=buy 的页" |
| `extract_facts` | 单次对话抽 5 类 fact | 用户 chat 时的偏好/承诺记录（个股本身不用） |
| `vector_search` / `keyword_search` / `hybrid_search` | 语义/关键词/混合 | 历史相似判断检索 |
| `gbrain doctor / stats / orphans` | 健康检查 | 部署 acceptance |

### ❌ 验证不可用（设计如此，不要再尝试）

| API | 错误模式 | 替代方案 |
|---|---|---|
| `take_add / takes_add / submit_take / create_take / add_take` | "Unknown tool" | 写进 page，dream cycle 抽 |
| `add_node` / `create_entity` | 不存在 | 实体即 page |
| 通过 body 里的 `[[wikilink]]` 自动建边 | v0.12 不识别 `[[]]` 语法 | 用标准 markdown `[name](slug)` |
| `list_pages(type='custom_type')` | 自定义 type 过滤有 bug | 用 tag 替代 |

### ⚠️ 行为不直观（必须显式处理）

| 现象 | 应对 |
|---|---|
| `sync` 之后新页**没 embedding** | 必须 `&& gbrain embed --stale` |
| Supabase Transaction pooler **静默丢页** | 用 Session pooler 或 direct（5432） |
| auto-link 不触发 | 检查 trustedWorkspace；不行就 `extract links --slug` |
| LISTEN/NOTIFY 在 PGLite 不工作 | 退化为 2s polling，能接受就用 |
| Minions worker 在 PGLite **拒绝启动** | 用 `gbrain jobs submit ... --follow` 启动临时 worker |

---

## 4. 部署/运维的 9 个硬陷阱

| # | 陷阱 | 影响 | 防御 |
|---|---|---|---|
| 1 | 裸 `---` 不再是 timeline 分隔符 | timeline 被吞进 compiled_truth，dream cycle 抽不到 | writer 用 `<!-- timeline -->` |
| 2 | trustedWorkspace 默认 false（远程 MCP） | auto-link 不建图 | 本地 stdio MCP / 显式 `extract links` |
| 3 | Supabase Transaction pooler | 同步**静默丢页** | 用 Session 模式 |
| 4 | sync 后没 embed | vector search 看不见 | `sync && embed --stale` |
| 5 | PGLite 不支持长进程 worker | Minions 启动失败 | `--follow` 临时 worker |
| 6 | Quiet hours 误推 | 用户 3 AM 被吵 → 关掉系统 | 所有推送先写 `/tmp/cron-held/`，盘后捞起 |
| 7 | Search mode 默认未选 | 25× 成本差异 | init 后强制展示 9 格矩阵 |
| 8 | take_add 不存在 | 判断不进 KG | 写进 page 等 dream cycle |
| 9 | source 不分 | federated 搜索串味 | 至少 3 个 source |

---

## 5. 直接复用清单：可拿来即用的 GBrain 抽象

来自 `designs/KNOWLEDGE_RUNTIME.md` 的 v0.20 路线图（部分已落地、部分在 main）。这些是 GBrain 一等公民原语，**我们的 Skill 应当顺着这些原语写**，未来零成本对接：

| GBrain 原语 | 我们的对应物 |
|---|---|
| **Resolver**（统一外部数据源接口） | `fetch-capital` / `fetch-sector` 数据采集 Skill 套这个壳 |
| **EnrichmentOrchestrator**（trigger / tier / budget / cascade） | "个股深度分析"的编排 |
| **Completeness Rubric**（按字段加权打分） | "个股画像完整度"评估，不要用"长度 > 500 字 = 完成"这种简单启发 |
| **BudgetLedger**（硬上限 + 日级断路器） | LLM / 数据 API 调用成本控制 |
| **BrainWriter Transaction**（all-or-nothing） | 多源数据写入（避免 timeline 写了 frontmatter 没写） |
| **FailImproveLoop**（确定性优先 → LLM 兜底） | "先 akshare → 失败降 efinance → 再失败才走爬虫" |
| **Hot/Cold 双层记忆** | 当日决策（hot fact）/ 历史判断（cold take） |

---

## 6. 对 PROPOSAL v0.4 / PLAN v0.3 的具体修正项

| # | 现有方案 | 修正后 | 理由 |
|---|---|---|---|
| M1 | "用 frontmatter + body 双层" | ✅ 保留，但分隔符显式 `<!-- timeline -->` | 陷阱 1 |
| M2 | "用 add_link API 主动建图" | 改为 page-driven：body `[name](slug)` + frontmatter 字段 | v1.1 H1 PASS |
| M3 | "用 take 写判断" | 改为 frontmatter `verdict/confidence` + timeline 一行 | 陷阱 8 |
| M4 | 单 source | 拆 `stock-analysis` / `market-research` / 默认 | §2.2 |
| M5 | 默认 Postgres | 默认 PGLite，> 1000 页或多设备再上 Supabase | `ENGINES.md` |
| M6 | "收盘后分析"单 cron | 三段式：盘后采集 → dream cycle → 次日 briefing | §2.6 + quiet-hours |
| M7 | "Hermes 走 MCP" | 必须 stdio（trusted），否则补 `extract links` | 陷阱 2 |
| M8 | 无 acceptance check | 部署完必跑 `doctor / stats / orphans` 三件套 | `GBRAIN_VERIFY.md` |
| M9 | LLM 成本无控 | 套 BudgetLedger 模式，日级断路器 | §5 |
| M10 | search mode 跳过 | init 后强制让用户在 9 格矩阵中选 | 陷阱 7 |
| M11 | timeline 无格式约束 | 强制 `**YYYY-MM-DD** \| 事件 \| 影响 \| [Source: ...]` | 让 dream cycle 抽得稳 |
| M12 | 数据源容错无策略 | FailImproveLoop：akshare → efinance → 爬虫，每跳记 timeline | §5 |

---

## 7. Lessons Learned（带证据）

| Lesson | 证据来源 |
|---|---|
| **Markdown 是真理，DB 是派生层** —— 我们要按"写 wiki"的姿势用 GBrain，不是按"写 ORM"的姿势 | `ethos/MARKDOWN_SKILLS_AS_RECIPES.md` |
| **API 不是越多越好** —— take 故意没有写 API，逼 agent 走 page-driven 路径，反而保证 timeline 完整 | v1.1 probe H4 + `takes-vs-facts.md` |
| **trustedWorkspace 是隐性开关** —— 不读 `infra-layer.md` 永远不知道为什么 auto-link 不触发 | `architecture/infra-layer.md` + v1.1 V11-5 |
| **Cron 串联很脆** —— `sync` 不带 `embed --stale`，后续 vector search 找不到新页，很多人在这里踩坑 | `guides/cron-schedule.md` |
| **PGLite vs Supabase 不是性能选择，是架构选择** —— PGLite 砍掉 Minions worker，换来零运维 | `ENGINES.md` |
| **dream cycle 是必需** —— 跳过它，brain 越用越乱（稀薄页堆积、判断不巩固） | `designs/KNOWLEDGE_RUNTIME.md` |
| **probe 优先于设计** —— 我们在 v1.0 的失败证明：不实测就按文档想象，会写出"看起来合理但 GBrain 不支持"的方案 | `_gbrain_probe_logs/` 双轮 |

---

## 8. 待办（不在本文档解决）

- [ ] PROPOSAL.md → v0.4 落实 §6 的 M1-M12 修正
- [ ] PLAN.md → v0.3 落实三段式 cron / acceptance / probe-driven 验收
- [ ] GBRAIN_PROBE 留作"知识库"，未来 GBrain 升级时复跑（保鲜测试）
- [ ] 写第一个 stock-analysis Skill 时，**仿照 GBrain 自己的 skill markdown 风格**（thin harness, fat skill）

---

## 附录 A：本次通读的文档清单

| 类别 | 文档 |
|---|---|
| 入口 | `llms.txt`、`AGENTS.md`、`CLAUDE.md`、`INSTALL_FOR_AGENTS.md`、`README.md` |
| 哲学 | `ethos/THIN_HARNESS_FAT_SKILLS.md`、`ethos/MARKDOWN_SKILLS_AS_RECIPES.md` |
| 架构 | `architecture/infra-layer.md`、`architecture/brains-and-sources.md`、`ENGINES.md`、`docs/GBRAIN_RECOMMENDED_SCHEMA.md`、`docs/takes-vs-facts.md` |
| Skill 调度 | `skills/RESOLVER.md`、`conventions/brain-first.md`、`conventions/quality.md`、`conventions/brain-routing.md` |
| 部署 | `mcp/DEPLOY.md`、`guides/live-sync.md`、`guides/cron-schedule.md`、`guides/quiet-hours.md`、`guides/minions-deployment.md`、`guides/minions-fix.md` |
| 调试/迁移 | `GBRAIN_VERIFY.md`、`integrations/reliability-repair.md`、`UPGRADING_DOWNSTREAM_AGENTS.md`、`CHANGELOG.md`、`skills/migrations/v0.12.0.md` |
| 设计 | `designs/HOMEBREW_FOR_PERSONAL_AI.md`、`designs/KNOWLEDGE_RUNTIME.md`、`designs/MINIONS_AGENT_ORCHESTRATION.md`、`designs/CODE_CATHEDRAL_II.md` |

实测证据：`docs/stock-analysis/_gbrain_probe_logs/`（v1.0 8 场景 + v1.1 5 场景）。
