# 个股深度分析工具 — 落地计划 PLAN v0.2

> 配套文档：[PROPOSAL.md](./PROPOSAL.md) v0.3 / [REVIEW.md](./REVIEW.md) v1.0 / [RESEARCH_zhulinsen.md](./RESEARCH_zhulinsen.md)
> 最近更新：2026-05-14
> 状态：方案已收敛 + REVIEW v1.0 修订全部内化，等用户拍板进入"第 1 步"
> 总体策略：**Cherry-pick 优先 + GBrain 兜底记忆 + 单 Skill 单步骤验证**

---

## v0.2 主要变更（相对 v0.1，回应 [REVIEW.md](./REVIEW.md) v1.0）

1. **§0 总览图 M3 描述修正**：M3 仅是"MVP 先头部队（板块 1）跑通"；新增 M4-M6 占位以补齐 PROPOSAL §6 定义的 MVP（P0-2）
2. **§第 3 步必含字段拆两层**：元数据层 + 判断层（四件套结构化字段），新增"字段级查询"作为显式验证标准（P1-2）
3. **§第 4 步 `name_to_code_resolver.py` 提升为必抽**：第 7 步入口依赖（RS-5）
4. **§第 7 步流程加入"分析前 query 历史"步骤**：让跨次对比能力真正落地（RS-4）
5. **§4 回归用例扩展为 4 只 + 1 边界**：覆盖白马 / 中盘活跃 / 周期 / 边界（停牌/ST）（P2-2）
6. **新增"GBrain 降级"贯穿条款**：每个 ingest/query 步骤都标注"失败不阻塞"
7. **新增 LLM Prompt 约束**：在第 7 步明确"baseline 仅参考、user_custom 优先"（RS-2 配合）

---

## 0. 计划总览（One-page View）

```
        ┌──────────────────────────────────────────────────┐
        │            v0.3 实施路径（共 7 步）               │
        └──────────────────────────────────────────────────┘

  Phase A：地基（不写新代码，先把可复用资产/记忆层都摸清）
    ├─ 第 1 步  抽 strategies/*.yaml（零依赖宝藏）  ⭐
    ├─ 第 2 步  探索 GBrain 现有能力（试 ingest / query）
    └─ 第 3 步  设计 Markdown Page Schema（数据流形状）

  Phase B：代码移植（按依赖从浅到深）
    ├─ 第 4 步  抽 code utils（纯函数、零外部依赖）
    ├─ 第 5 步  抽 data_provider 核心（akshare + efinance）
    └─ 第 6 步  抽 stock_analyzer.py（精简版分析骨架）

  Phase C：Skill 化首跑
    └─ 第 7 步  包装"板块环境分析" Skill（含 ingest 入脑）→ MVP 先头部队

  里程碑（本 PLAN 之内）：
    M1 = 第 3 步结束 → 数据流形状定型
    M2 = 第 6 步结束 → 后端能跑通"取数→分析→Markdown 产物"
    M3 = 第 7 步结束 → MVP 先头部队（板块 1）跑通，验证端到端管道

  里程碑（本 PLAN 之外，待板块 1 跑通后再立项）：
    M4 = 板块 2（基本盘）Skill 上线
    M5 = 板块 3（资金面）Skill 上线
    M6 = 一句话摘要 Agent 编排接通
    → M3-M6 全部完成才算交付 PROPOSAL §6 定义的"第一阶段 MVP"
```

---

## 1. 通用约束与原则

| 约束 | 说明 |
|---|---|
| **每一步都要可验证** | 跑通 1 个具体 case 才算完成，不允许"代码搬过来 = 完成" |
| **每一步都先做"最小切片"** | 不追求一次抽完，宁可多抽几次 |
| **每一步都更新 ATTRIBUTION.md** | 抽了哪个文件、哪个 commit hash，立刻登记 |
| **每一步产出都同步进 GBrain** | 用自己设计的 Markdown Page 写进去，**用我们自己的工具吃自己的狗粮** |
| **GBrain 失败不阻塞主流程** | 任何 ingest/query 步骤遇错都进入 stateless 模式继续，按 PROPOSAL §3.3.1 降级 |
| **遇到耦合 / 包袱** | 立即放弃，重新评估是否值得整合，不要硬抠 |

---

## 2. 详细步骤

### 🔧 第 1 步：抽 strategies/\*.yaml

> **零依赖宝藏，10 分钟出活，先出战果**

| 项 | 内容 |
|---|---|
| **目标** | 把 ZhuLinsen 的 11 套交易策略 YAML 拷进本仓库，作为后续 AI 判断的提示词基线 |
| **输入** | `_research/daily_stock_analysis/strategies/*.yaml`（11 个文件，588 行，零依赖） |
| **输出** | `skills/stock-analysis/_assets/strategies/*.yaml`（路径暂定，可在 Step 7 调整） |
| **不要做** | 不要改写 / 不要本土化 —— 先原样拷过来，让"理解他怎么思考的"过程发生 |
| **入脑要求（RS-2）** | ingest 进 GBrain 时必须带 `source: baseline` + `priority: reference`，避免污染 user_custom |
| **验证标准** | ① 11 个文件全部到位；② 在 ATTRIBUTION.md 中登记 commit hash + 来源；③ 抽 1-2 个手动 ingest，query 能召回且 source 字段正确 |
| **风险** | 极低 |
| **预估工作量** | 10 分钟 |
| **依赖** | 无 |

**完成后立刻产出**：
- 在 ATTRIBUTION.md 中登记
- 在 GBrain 中新建 `strategies/<name>.md` 页面，做成 `Strategy` 实体（先手动 ingest 1-2 个验证流程）

---

### 🧠 第 2 步：探索 GBrain 现有能力

> **先摸清家底，再设计 Schema**

| 项 | 内容 |
|---|---|
| **目标** | 实测 GBrain 的 ingest / query / brain-ops 等 MCP 工具，确认能力边界 |
| **输入** | 已安装的 GBrain（在 Hermes 的 MCP 配置中） |
| **任务清单** | ① 列出所有可用 MCP 工具；② 用一段 plain text ingest，看它怎么提取实体；③ 用 frontmatter 标记的 markdown ingest，看 typed link 是否自动建立；④ query 一次，看返回结构；⑤ 验证"字段级 query"是否可行（决定 P1-2 Schema 设计） |
| **输出** | `docs/stock-analysis/RESEARCH_gbrain.md`（实测笔记） |
| **关键验证点** | a. 能否用 frontmatter 显式声明实体类型？<br>b. typed link 是否需要规约命名？<br>c. ingest 是否幂等？同一篇 markdown 重 ingest 会复制吗？<br>d. **能否对 frontmatter 中的结构化字段（如 `confidence: low`）做精确过滤查询？** |
| **风险** | 中 — 文档可能不全，需通过实验探索 |
| **预估工作量** | 1-2 小时 |
| **依赖** | 无 |

**完成产出**：
- RESEARCH_gbrain.md 至少回答上述 4 个验证点
- 在第 3 步设计 Schema 时直接引用这份笔记

---

### 📐 第 3 步：设计 Markdown Page Schema 【里程碑 M1】

> **整个数据流的"形状"在这里定型**

| 项 | 内容 |
|---|---|
| **目标** | 定义两类核心 markdown 页面的 frontmatter 与正文约定 |
| **输入** | 第 2 步的 GBrain 实测结论 |
| **输出** | 在 PROPOSAL.md 第 3.4 节下追加 §3.4.1 Page Schema |
| **包含两类 Schema** | a. **Stock Master Page**（`stocks/<code>.md`）—— 长期更新主页<br>b. **Report Page**（`reports/<code>_<date>.md`）—— 单次分析报告 |

**Schema 分两层（P1-2）**：

a. **元数据层（frontmatter）**

```yaml
id: <唯一 id>
type: stock | report
code: 300750
name: 宁德时代
sector: 新能源
industry: 动力电池
created_at: 2026-05-14T10:30:00+08:00
strategy_refs: [chan_theory_one_buy, ...]
report_refs: [300750_2026-05-13]
news_refs: []
```

b. **判断层（每个分析模块一个 entry，强结构化）**

```yaml
modules:
  sector_environment:                # 板块 1
    verdict: <一句话结论>
    confidence: high | medium | low
    indicators:
      - name: 板块5日涨幅排名
        value: 第8/31
        meaning: 中等偏强
      - name: 板块净流入连续天数
        value: 3
        meaning: 资金持续进场
    reasoning_chain:
      - step: 由「指标X」推出「子结论A」
      - step: 由「指标Y+Z」推出「子结论B」
      - conclusion: 综合 → ...
    counter_signals:
      - 可能反驳此结论的指标 / 视角
    strategy_refs_used: [chan_theory_one_buy]   # 本次判断引用了哪些策略
```

**关系约定**：用 markdown link 显式写出 typed link，例如
> `[宁德时代](stocks/300750.md) belongs_to_sector [新能源](sectors/new_energy.md)`

| 项 | 内容 |
|---|---|
| **验证标准** | ① 手写一份 sample report page，ingest 进 GBrain，query 后能正确返回实体与关系；② **关键**：能用 GBrain query 做"字段级"取回（如：列出 confidence=low 的所有 report）—— 仅向量相似召回不算通过 |
| **风险** | 中 — Schema 设计错了，后面所有 Skill 都要返工 |
| **预估工作量** | 半天 |
| **依赖** | 第 2 步 |

**🚩 里程碑 M1：数据流形状定型，进入移植阶段**

---

### 🛠️ 第 4 步：抽 code utils

> **从依赖最浅的开始动**

| 项 | 内容 |
|---|---|
| **目标** | 抽取 ZhuLinsen 中纯函数 / 工具类，不涉及外部 IO |
| **抽取清单** | ① `src/services/stock_code_utils.py`（代码归一化）— **必抽**<br>② `src/services/name_to_code_resolver.py`（名→码）— **必抽（第 7 步入口依赖，RS-5）**<br>③ `src/patches/eastmoney_patch.py`（东财补丁，0 依赖）— 候选 |
| **路径** | `skills/stock-analysis/_lib/utils/`（与 _assets 同级） |
| **不要带的** | 任何依赖 LiteLLM / Config / Storage 的工具函数 |
| **验证标准** | 写 3-5 条 pytest 用例（输入名→输出代码、输入脏代码→输出干净代码、输入"宁德时代"→输出 300750） |
| **风险** | 低 |
| **预估工作量** | 半天 |
| **依赖** | 无（可与第 2、3 步并行） |

**完成产出**：
- ATTRIBUTION.md 追加登记
- 单元测试通过

---

### 📡 第 5 步：抽 data_provider 核心

> **数据是命脉，但要解耦**

| 项 | 内容 |
|---|---|
| **目标** | 把 ZhuLinsen 的两个稳定取数适配器搬过来，去掉框架耦合 |
| **抽取清单** | ① `data_provider/akshare_fetcher.py`<br>② `data_provider/efinance_fetcher.py`<br>③ `data_provider/base.py`（精简版，去掉 Config / 日志依赖） |
| **路径** | `skills/stock-analysis/_lib/data_provider/` |
| **解耦动作** | a. 去掉对全局 Config 的依赖，改成函数参数注入<br>b. 去掉对项目内 logger 的依赖，改用 stdlib logging<br>c. 保留接口层，下层实现可替换 |
| **验证标准** | 跑全量"回归用例池"（见 §4），5 只股票都能取到行情 / 板块 / 资金流 3 个最常用接口的数据；停牌/ST 边界 case 不报错（按降级路径返回空数据 + 警告） |
| **风险** | 中 — akshare / efinance 上游可能改接口；停牌/ST 数据格式不稳定 |
| **预估工作量** | 1 天 |
| **依赖** | 第 4 步（utils） |

**完成产出**：
- ATTRIBUTION.md 追加登记
- 一个简单的 demo 脚本能跑通"取宁德时代板块环境数据"

---

### 🧱 第 6 步：抽 stock_analyzer.py 【里程碑 M2】

> **核心分析骨架，最考验取舍**

| 项 | 内容 |
|---|---|
| **目标** | 抽取 ZhuLinsen 中的 `src/stock_analyzer.py`（约 847 行）作为分析编排骨架 |
| **不要抽的** | 不要抽 `src/analyzer.py`（3223 行，已在决策中拒绝） |
| **路径** | `skills/stock-analysis/_lib/analyzer/` |
| **必须重写的** | a. LLM 调用层 → 改成"输出结构化字典"，由上层 Skill 负责接 LLM<br>b. 报告渲染 → 输出 Markdown Page（按第 3 步 Schema，包含元数据层 + 判断层） |
| **验证标准** | 跑全量"回归用例池"（5 只）：取数 → 简单分析 → 输出一份符合 Schema 的 markdown 报告；其中 confidence/indicators/reasoning_chain 等字段必须落到 frontmatter |
| **风险** | 高 — 改动量大，最容易出意外耦合 |
| **预估工作量** | 1-2 天 |
| **依赖** | 第 3、5 步 |

**🚩 里程碑 M2：后端能跑通"取数→分析→Markdown 产物"，开始 Skill 化**

---

### 🚀 第 7 步：包装"板块环境分析" Skill 【里程碑 M3】

> **MVP 先头部队（板块 1），端到端打通**

| 项 | 内容 |
|---|---|
| **目标** | 把"板块 1：板块与行业环境"包装成一个可执行的 Skill，端到端跑通 |
| **Skill 名** | `analyze-sector-environment`（暂定） |
| **输入** | 股票代码 / 名称 |
| **板块 1 范围** | 1.1 板块轮动 + 1.3 个股 vs 板块 vs 行业；**1.2 仅做"通用版指标"（行业 ROE / 毛利率 / 增速中位数）**，定制模板/PMI/政策推第二阶段（PROPOSAL §4 RS-1） |

**流程（7 步）**：

```
① resolve 股票代码（依赖第 4 步 name_to_code_resolver）
② 【新增】GBrain query 拉历史分析摘要（最近 3 次）作为 LLM 的 prior context
   ── 失败不阻塞，降级为"无历史 context"，结果中标注 ⚠️
③ 调 data_provider 取板块/行业数据（依赖第 5 步）
④ 调 LLM（带"四件套"prompt + 历史摘要 + baseline strategies）做轮动阶段判断
   ── Prompt 显式声明：'baseline 仅作参考视角；如存在 user_custom 策略以 user_custom 优先；
      输出必须明确指出参考了哪条 strategy_id。'（RS-2）
⑤ 渲染成 Markdown Page（按第 3 步 Schema：frontmatter 元数据层 + modules.sector_environment 判断层）
⑥ 调 GBrain MCP `ingest` 入脑
   ── 失败不阻塞，写入 `~/.stock-skills/ingest_retry.jsonl`，主流程继续
⑦ Markdown → HTML 渲染（图表 placeholder）
```

| 项 | 内容 |
|---|---|
| **不要做** | 不要在第一个 Skill 里塞所有 10 个板块，**只做板块 1**，其他后续逐个加 |
| **验证标准** | a. 在 Hermes 中手动调用，输入"宁德时代"能拿到一份板块环境报告<br>b. 报告同时落地为 HTML、Markdown Page、被 GBrain 收录<br>c. query GBrain 能召回，且能按 confidence 字段过滤<br>d. **跨次对比验证**：对同一股票连续跑两次，第二次报告的 reasoning_chain 中应能引用第一次的结论变化（如"相比 N 天前 confidence 由中变低"）<br>e. **降级验证**：手动断开 GBrain，主流程仍能产出 HTML/Markdown 不报错<br>f. **回归用例**：5 只全集（含边界 case）都能产出有效报告，停牌/ST 不崩 |
| **风险** | 中 — Skill 编排约定可能需要迭代 |
| **预估工作量** | 1 天 |
| **依赖** | 第 1、3、6 步 |

**🚩 里程碑 M3：MVP 先头部队（板块 1）跑通，验证端到端管道**

---

## 3. 各步依赖关系图

```
第1步 ────────────────────────────────────┐
（YAML 抽取）                              │
                                          ▼
第2步 ──▶ 第3步 ──┐                  第7步（Skill 化）
（GBrain 实测）（Schema 设计）         ▲
                  │                    │
                  ▼                    │
                第6步 ◀── 第5步 ◀── 第4步
                （analyzer）（data_provider）（utils）
```

并行机会：
- 第 1 步可与一切步骤并行
- 第 4 步与第 2 步可并行
- 第 5 步必须在第 4 步之后

---

## 4. 验证机制（贯穿每一步）

| 检查点 | 触发时机 | 内容 |
|---|---|---|
| **走查 ATTRIBUTION.md** | 每抽一个文件 | 登记 ZhuLinsen commit hash、文件路径、本地路径 |
| **dogfood：自己 ingest 进 GBrain** | 每完成一个产物 | 看是否被正确收录、能否 query 出（含字段级过滤） |
| **真实股票回归测试** | 第 5、6、7 步 | 见下方"回归用例池" |
| **降级路径回归** | 第 6、7 步 | 手动断开 GBrain / 模拟 akshare 异常，确认不阻塞 |
| **遇到坑写进 RESEARCH** | 每一步 | RESEARCH_zhulinsen.md / RESEARCH_gbrain.md 持续追加 |

### 4.1 回归用例池（固定，每次第 5/6/7 步都跑全集，P2-2）

| 编号 | 股票 | 类型 | 验证目标 |
|---|---|---|---|
| RT-01 | 600519 贵州茅台 | 大盘价值白马 | 基础流程、板块定位稳定性 |
| RT-02 | 300750 宁德时代 | 大盘成长白马 | 板块龙头判断 |
| RT-03 | 当期热门概念中盘活跃股（每月动态选 1 只） | 中盘活跃 | 板块轮动加速期识别、矛盾信号识别 |
| RT-04 | 周期股龙头（如紫金/万华，动态确认） | 强周期 | 行业景气度判断、上下游传导（MVP 仅通用版） |
| RT-EDGE-01 | 任一停牌或 ST 股 | 边界 | 错误降级路径、数据缺失行为 |

每次回归运行后，把 5 份 markdown report 都 ingest 进 GBrain，
再用一条 `gbrain query` 验证"跨股票"召回是否合理（例如 query "高 confidence 的板块龙头" 能否同时召回相关用例）。

---

## 5. 不在本计划中（防止范围蔓延）

- ❌ 板块 2-9（基本盘、资金、筹码、走势、业绩、消息、风险、同业）→ 第 7 步之后逐个新建 Skill（对应 M4-M6）
- ❌ "一句话摘要" 汇总 Agent → 等至少 3 个板块 Skill 都跑通了再做（对应 M6）
- ❌ HTML 视觉风格优化 / 图表交互 → 拿到 markdown → HTML 的最简管道即可
- ❌ AgentTeam 多 Skill 编排脚本 → 第 7 步只跑单 Skill，编排留给后续
- ❌ Cookie 抓取流程 → 第一阶段只用 akshare / efinance（ZhuLinsen 已有），cookie 化在第二阶段
- ❌ 板块 1.2 行业定制模板 / PMI / 政策导向 / 一致预期 → 第二阶段（需 cookie）

---

## 6. 立刻可以开干的最小动作（请用户拍板）

> 用户只要说"开干"，下一步就执行：

```
[第 1 步] 抽 strategies/*.yaml（10 分钟）
  ├─ 创建 skills/stock-analysis/_assets/strategies/
  ├─ 拷贝 11 个 yaml 文件
  ├─ 创建 docs/stock-analysis/ATTRIBUTION.md
  └─ 提交一次 commit："chore(stock-analysis): cherry-pick strategies from ZhuLinsen"
```

**预计 10 分钟内完成首个产出，让整件事从"讨论"进入"动手"状态。**

---

## 7. 与 PROPOSAL.md 的对应关系

| PROPOSAL 节 | 在本 PLAN 中的落地步骤 |
|---|---|
| §3.1 形态 | 第 7 步（Skill + Hermes 跑通） |
| §3.2 三层产物 | 第 3 步（Schema） + 第 6/7 步（实际产出） |
| §3.3 GBrain 集成 | 第 2 步（实测）+ 第 7 步（ingest + query） |
| §3.3.1 GBrain 降级策略 | §1 通用约束 + 第 5/6/7 步降级验证 |
| §3.4 实体 / typed link（含 Strategy.source） | 第 1 步 ingest 时落库 + 第 3 步 Schema 固化 |
| §4 板块 1（含 1.2 通用版降级） | 第 7 步 |
| §4 板块 2-9 | 不在本 PLAN（M4-M6 之后） |
| §5 AI 输出 4 件套 | 第 3 步（Schema）+ 第 7 步（LLM prompt 模板） |
| §6 三阶段 + 分档 SLA | 本 PLAN 只覆盖板块 1（"先头部队"），M4-M6 才补齐第一阶段 MVP |

---

## 8. 决策记录（PLAN 层）

| 日期 | 决策 | 理由 |
|---|---|---|
| 2026-05-12 | **从 strategies/*.yaml 开始** | 零依赖、零风险、出战果最快、最能建立信心 |
| 2026-05-12 | **GBrain 实测先于 Schema 设计** | 实测决定 Schema 的可行性，反之会拍脑袋 |
| 2026-05-12 | **第一个 Skill 只做板块 1** | 端到端跑通比覆盖广更重要 |
| 2026-05-12 | **回归用例固定 300750 + 600519** | 一只成长 + 一只价值，覆盖典型分析路径 |
| 2026-05-12 | **第一阶段不接 cookie 抓取** | 先用 akshare/efinance 把流程跑通，cookie 留给第二阶段 |
| 2026-05-14 | **M3 改名为"MVP 先头部队（板块 1）跑通"，新增 M4-M6 占位** | Review P0-2 命名对齐 |
| 2026-05-14 | **Schema 拆元数据层 + 判断层，"四件套"全部结构化** | Review P1-2 跨次对比能力前置 |
| 2026-05-14 | **`name_to_code_resolver.py` 提升为必抽** | Review RS-5：第 7 步入口依赖 |
| 2026-05-14 | **第 7 步流程从 6 步扩为 7 步，新增"分析前 query 历史"** | Review RS-4：跨次对比必经路径 |
| 2026-05-14 | **回归用例池扩为 4 只 + 1 边界**，每次第 5/6/7 步跑全集 | Review P2-2 覆盖盲区 |
| 2026-05-14 | **GBrain 降级条款写进 §1 通用约束**，每个 ingest/query 都标注"失败不阻塞" | 配合 PROPOSAL §3.3.1 |
| 2026-05-14 | **LLM Prompt 显式声明 baseline ≤ user_custom 优先级** | Review RS-2 配合 |

