# GBrain 探索测试方案 v1.0

> 用途：交付给 [Hermes Agent](https://github.com/garrytan/gbrain) 在**开发机**上执行，实测 GBrain 的 MCP 工具能力边界
> 对应 [PLAN.md](./PLAN.md) §第 2 步
> 输出去向：所有结果回填到本文件 §6 的"实测结果"小节，并以本文件为基础生成 `RESEARCH_gbrain.md`
> 测试样本：本仓库 [`skills/stock-analysis/_assets/strategies/`](../../skills/stock-analysis/_assets/strategies/) 中的 ZhuLinsen 11 套策略 YAML
> Review 关联：[REVIEW.md](./REVIEW.md) P1-2（字段级查询） / RS-2（baseline 隔离）

---

## 0. 执行须知（给 Hermes 看）

**身份**：你是 Hermes，运行在开发机上，已安装 GBrain MCP 服务。
**目标**：按本文件的 7 个 SCENARIO 顺序执行，每个 scenario 内部按 STEP 编号执行，产出"PASS/FAIL/UNKNOWN" + 关键证据片段。
**原则**：
1. **只读优先**：先列工具、再 dry-run、最后才真写
2. **可重复**：每个 ingest 都用唯一可识别的 `test_run_id`（建议 `gbrain-probe-YYYYMMDD-HHMM`），便于事后清理
3. **不污染**：所有测试页面统一前缀 `_probe/`，事后用一条命令一键清理（见 §7）
4. **失败即停**：某个 scenario 失败时，记录现象后**继续下一个 scenario**，不要尝试修复 GBrain 本身
5. **写产物**：执行完毕后，把"实测结果"写到 §6 表格 + 把 raw 输出存到 `docs/stock-analysis/_gbrain_probe_logs/<scenario>.txt`

---

## 1. 准备

### 1.1 环境前置

```
✅ Hermes 中已安装 GBrain MCP（用户已确认）
✅ 本仓库已 clone 到开发机（含 skills/stock-analysis/_assets/strategies/*.yaml）
✅ 当前分支：main，HEAD = 6d13119（Step 1 cherry-pick 完成）
```

### 1.2 测试常量

| 名称 | 值 |
|---|---|
| `TEST_RUN_ID` | `gbrain-probe-<YYYYMMDD-HHMM>`（执行时取当时时间） |
| `PROBE_NAMESPACE` | `_probe/<TEST_RUN_ID>/`（所有写入页面的统一前缀） |
| `SAMPLE_STRATEGY` | `chan_theory.yaml`（缠论，58 行，结构清晰适合做样本） |
| `SAMPLE_STOCK_CODE` | `300750`（宁德时代，回归用例 RT-02） |

---

## 2. 测试目标对照表

| # | 验证点 | 决定下一步什么 | 关键性 |
|---|---|---|---|
| V1 | 列出所有 MCP 工具 | 选用哪些原语 | 🔥 |
| V2 | frontmatter 是否能显式声明实体类型 | Schema 是否需要规约命名 | 🔥 |
| V3 | typed link 是否需要规约命名（含字段） | Schema 关系语法 | 🔥 |
| V4 | ingest 是否幂等（同 page 重写是否复制） | 重跑回归用例时要不要先 delete | 🔥 |
| V5 | 能否对 frontmatter 字段做**精确过滤查询** | P1-2 结构化字段方案是否成立 | 🔥🔥（决定 Schema 设计） |
| V6 | `source: baseline` 字段能否被独立筛选 | RS-2 baseline 隔离能否实现 | 🔥 |
| V7 | query 失败时返回什么（异常 / 空数组 / 错误码） | PROPOSAL §3.3.1 降级策略落地形式 | ⚠️ |

---

## 3. 测试场景列表

```
SCENARIO 1：盘点 GBrain MCP 工具清单                  (V1)
SCENARIO 2：plain text ingest，看实体提取             (V2)
SCENARIO 3：frontmatter markdown ingest，看 typed link (V2 V3)
SCENARIO 4：幂等性测试（同 page 写两次）              (V4)
SCENARIO 5：基础 query 测试（向量召回）               (V5 基础)
SCENARIO 6：字段级精确过滤查询                        (V5 P1-2 关键判据)
SCENARIO 7：baseline 隔离 query（source=baseline）    (V6 RS-2 关键判据)
SCENARIO 8：异常路径 / 边界 case                      (V7)
```

---

## 4. 详细测试步骤

### 🔍 SCENARIO 1：盘点 MCP 工具清单（V1）

**目的**：列出 GBrain 暴露的全部 MCP 工具，作为后续 scenario 的"原语字典"。

**STEP 1.1**：执行 MCP server 工具列表查询
- Hermes 内部应有 `list_tools` 或类似元能力，直接列出 GBrain 命名空间下的所有工具
- 如果没有元接口，尝试 `gbrain --help` / `gbrain list-tools` / 查看 GBrain MCP 配置文件

**STEP 1.2**：对每个工具记录：
- 工具名 / 简述 / 是否需要 auth / 输入 schema 摘要 / 是否有副作用（读 vs 写）

**预期输出（填到 §6 表格）**：
```
[
  { name: "ingest", side_effect: "write", input: "..." },
  { name: "query",  side_effect: "read",  input: "..." },
  { name: "brain-ops", ... },
  ...
]
```

**通过标准**：列出至少 5 个工具，并能区分读 / 写。

---

### 📝 SCENARIO 2：plain text ingest 实体提取（V2）

**目的**：看 GBrain 在没有任何 frontmatter 提示的情况下，能否从中文文本里自动提取出 Stock / Sector / Strategy 等实体。

**STEP 2.1**：构造测试文本（保存为 `${PROBE_NAMESPACE}plain_test.md`）

```markdown
今日复盘：宁德时代（300750）所属新能源板块今日板块涨幅 2.3%，
属于动力电池行业。技术上呈现波浪理论中的第3浪特征，建议关注。
板块龙头还包括比亚迪（002594）。
```

**STEP 2.2**：调用 `ingest`，page path 用 `${PROBE_NAMESPACE}plain_test.md`

**STEP 2.3**：调用 `query "宁德时代"`

**STEP 2.4**：调用 `query "新能源 板块"`

**STEP 2.5**：尝试取该 page 的图谱视图（如有 `brain-ops` / `get_graph` 类工具）

**预期发现**：
- ✅ "宁德时代" / "300750" 被识别为实体？
- ✅ 是否自动建立"宁德时代 — 新能源"的关联？
- ✅ 是否区分得了 Stock / Sector / Industry / Strategy 类型？

**关键问题（必须回答）**：
- Q2.A：实体类型由 LLM 推断 还是 规则匹配？
- Q2.B：能否在不写 frontmatter 的情况下让它知道"300750 是 Stock 不是 News"？

---

### 🏷️ SCENARIO 3：frontmatter markdown ingest（V2 V3 - 关键）

**目的**：本项目 Schema 的核心赌注 —— 用 frontmatter 显式声明实体类型与 typed link 是否能落地。

**STEP 3.1**：构造测试 page（保存为 `${PROBE_NAMESPACE}stocks/300750.md`）

```markdown
---
id: stock-300750
type: Stock
code: "300750"
name: 宁德时代
sector: 新能源
industry: 动力电池
created_at: 2026-05-17T12:00:00+08:00
test_run_id: <TEST_RUN_ID>
---

# 宁德时代 (300750)

[宁德时代](stocks/300750.md) belongs_to_sector [新能源](sectors/new_energy.md)
[宁德时代](stocks/300750.md) belongs_to_industry [动力电池](industries/dongli_battery.md)
[宁德时代](stocks/300750.md) competes_with [比亚迪](stocks/002594.md)
```

**STEP 3.2**：构造对应的 sector page（`${PROBE_NAMESPACE}sectors/new_energy.md`）

```markdown
---
id: sector-new-energy
type: Sector
name: 新能源
classification: shenwan_l1
test_run_id: <TEST_RUN_ID>
---

# 新能源板块
```

**STEP 3.3**：构造一个 strategy page（直接拷 `chan_theory.yaml` 转 markdown，**带 RS-2 字段**）

```markdown
---
id: strategy-chan-theory
type: Strategy
name: chan_theory
display_name: 缠论
source: baseline
priority: reference
upstream_commit: a75a0c502e06a49f08439275aaeb7be241c5befe
test_run_id: <TEST_RUN_ID>
---

# 缠论一买点

（正文从 chan_theory.yaml 的 instructions 字段拷过来即可）
```

**STEP 3.4**：依次 ingest 三份 page

**STEP 3.5**：调用 `query "宁德时代 belongs_to_sector"` 或类似图谱查询

**STEP 3.6**：调用 `query "新能源板块的股票"` —— 看反向遍历能不能命中宁德

**STEP 3.7**：尝试列出该 stock page 关联的所有 typed link 关系

**关键问题（必须回答）**：
- Q3.A：`type: Stock` 是被 GBrain 当作实体类型语义？还是只是普通元数据？
- Q3.B：markdown 正文里 `[A](X) belongs_to_sector [B](Y)` 这种 typed link 写法是否被识别？还是要用其他语法？
- Q3.C：是否支持 frontmatter 里的 `links:` 数组显式声明？例如：
  ```yaml
  links:
    - rel: belongs_to_sector
      to: sector-new-energy
  ```
- Q3.D：自定义 typed link 名称（如 `belongs_to_sector`）是否需要预注册？

---

### ♻️ SCENARIO 4：幂等性测试（V4）

**目的**：搞清重跑 ingest 会不会复制实体/边，决定回归用例脚本要不要先 delete。

**STEP 4.1**：把 SCENARIO 3 的 `stocks/300750.md` **完全相同地** 再 ingest 一次

**STEP 4.2**：调用 `query "300750"`，看返回 1 条还是 2 条

**STEP 4.3**：修改正文（不改 frontmatter id），第三次 ingest，看会替换 还是 追加

**STEP 4.4**（可选）：尝试找到 GBrain 的 `update` / `upsert` 工具，对比和 `ingest` 的差异

**关键问题**：
- Q4.A：相同 `id` 的 page 重 ingest 是 upsert 还是 append？
- Q4.B：如果是 append，删除老版本的 API 是什么？

---

### 🔎 SCENARIO 5：基础 query 测试（V5 基础）

**目的**：摸清 query 接口的返回结构，作为 SCENARIO 6 / 7 的对照基线。

**STEP 5.1**：调用 `query "动力电池"`（向量相似召回）

**STEP 5.2**：调用 `query "缠论一买点"`

**STEP 5.3**：记录返回结构 —— 字段、是否带 score、是否分组、是否带关系

**关键问题**：
- Q5.A：返回是否带 confidence/score？阈值是什么？
- Q5.B：单次 query 默认返回多少条？能否分页？
- Q5.C：能否限定只查特定 `type` 的实体（如只查 Stock 不查 Strategy）？

---

### 🎯 SCENARIO 6：字段级精确过滤查询（V5 - P1-2 关键判据）

**目的**：**这是整个 Schema 设计的命门**。
[REVIEW.md](./REVIEW.md) P1-2 说"如果四件套是非结构化文本块，GBrain 只能向量召回相似文本，做不了字段级对比"。本场景就是要直接把它打透。

**STEP 6.1**：构造一份**带判断层 frontmatter**的 report page（按 PLAN §第 3 步设计）

```markdown
---
id: report-300750-2026-05-17
type: Report
code: "300750"
name: 宁德时代
created_at: 2026-05-17T12:00:00+08:00
test_run_id: <TEST_RUN_ID>
modules:
  sector_environment:
    verdict: "板块处于轮动中段"
    confidence: medium
    indicators:
      - name: 板块5日涨幅排名
        value: "第8/31"
        meaning: 中等偏强
    reasoning_chain:
      - step: 由板块涨幅推出资金温和进场
      - conclusion: 综合中性偏多
    counter_signals:
      - 板块龙头量能开始萎缩
    strategy_refs_used: [chan_theory]
---

# 宁德时代 板块环境分析（2026-05-17）

正文略
```

**STEP 6.2**：再构造一份 confidence=low 的 report（编号改 `report-300750-2026-05-18`）：
```yaml
modules:
  sector_environment:
    verdict: "板块退潮"
    confidence: low
```

**STEP 6.3**：再构造一份 confidence=high 的 report（不同股票，编号 `report-600519-2026-05-17`）

**STEP 6.4**：尝试 query —— 这是关键尝试矩阵：

| 试法 | 期望结果 | 真实结果 |
|---|---|---|
| 试法 A：直接自然语言 `"confidence 为 low 的 report"` | 看是否能精确过滤 | ⬜ |
| 试法 B：`query` + 过滤参数（如 `where: { type: "Report", confidence: "low" }`） | 看是否支持参数化过滤 | ⬜ |
| 试法 C：调用 brain-ops 类原语，直接做 SQL/图谱查询 | 看是否暴露底层 PG 通道 | ⬜ |
| 试法 D：query 全部 Report 后客户端过滤 | 兜底方案，但低效 | ⬜ |

**关键问题（必须回答 — 决定 Schema 设计）**：
- Q6.A：**至少一种试法能精确过滤 frontmatter 字段吗**？
- Q6.B：如果都不行，PLAN §第 3 步必须改成"双轨" —— 主流向量召回 + 旁路本地索引
- Q6.C：嵌套字段（`modules.sector_environment.confidence`）能否被检索？还是要拍平到顶层

---

### 🛡️ SCENARIO 7：baseline 隔离 query（V6 - RS-2 关键判据）

**目的**：验证 RS-2 的对策是否有效 —— LLM 在做判断时，能否被约束"只参考 user_custom 优先于 baseline"。

**STEP 7.1**：在 SCENARIO 3 已写入的 `strategy-chan-theory` 基础上，再写入一份 user_custom 策略：

```markdown
---
id: strategy-my-rotation-rule
type: Strategy
name: my_rotation_rule
display_name: 我的轮动规则
source: user_custom
priority: primary
test_run_id: <TEST_RUN_ID>
---

# 我的轮动规则
板块连续 3 日资金净流入 + 涨幅前 5，且本股属于该板块前 3 大权重，视作轮动加速期入场信号。
```

**STEP 7.2**：query "可用于判断板块轮动的策略"

**STEP 7.3**：query 时尝试加 `where: { source: "user_custom" }` 或 `where: { priority: "primary" }`

**STEP 7.4**：构造一段假分析 prompt 并调用：
```
"基于已有的 Strategy 实体，给我宁德时代板块轮动判断。
要求：user_custom 优先于 baseline；输出必须标注引用了哪条 strategy_id。"
```
看 LLM 输出会不会真的优先引用 `my_rotation_rule` 而不是 `chan_theory`。

**关键问题**：
- Q7.A：能否单纯按 `source` 字段分组返回？
- Q7.B：LLM 的 priority 约束是 prompt 工程的事 还是 GBrain 能在召回层就帮我们排序？

---

### ⚠️ SCENARIO 8：异常路径 / 边界 case（V7）

**目的**：搞清楚 PROPOSAL §3.3.1 "降级策略" 写得对不对 —— GBrain 各种异常时究竟是抛错还是返空？

**STEP 8.1**：query 一个肯定不存在的字符串（如 `"asdfqwer1234不存在的板块"`）
- 是空数组、null、还是抛错？

**STEP 8.2**：ingest 一份 frontmatter 故意写错的 markdown（`type: NotARealType`）
- 是拒绝、警告，还是默默吞下？

**STEP 8.3**：ingest 一份 frontmatter 字段类型不对的（`confidence: 高`，String 替代 enum）
- 是 schema 校验失败 还是 接受？

**STEP 8.4**（可选，谨慎）：临时停止 GBrain MCP 服务，再 query 一次
- 客户端是 timeout、connection refused、还是有专门的 fallback？

**关键问题**：
- Q8.A：失败的错误码 / 异常类型是什么？（决定 Skill 层 try/except 怎么写）
- Q8.B：是否有 retry 机制？

---

## 5. 一键下发 Prompt（粘给 Hermes 即可执行）

> 把下面这一整段复制到 Hermes，触发本次探索

```
你现在按 docs/stock-analysis/GBRAIN_PROBE.md 的计划，
依次执行 SCENARIO 1 到 SCENARIO 8。

执行规则：
1. TEST_RUN_ID = gbrain-probe-$(date +%Y%m%d-%H%M)
2. 所有写入 page 都加前缀 _probe/$TEST_RUN_ID/
3. 每个 scenario 完成后，立刻在 docs/stock-analysis/GBRAIN_PROBE.md §6
   实测结果表格中追加一行（PASS/FAIL/UNKNOWN + 一句话证据）
4. 把每个 scenario 的 raw I/O 存到
   docs/stock-analysis/_gbrain_probe_logs/<scenario>.txt
5. 全部 scenario 跑完后，根据实测结果填充 §7 结论清单
6. 最后执行 §7 清理脚本 把 _probe/ 命名空间清掉

如果某个 scenario 失败，记录后继续下一个，不要尝试修复 GBrain 本身。

执行完毕后，把所有改动 commit:
"chore(stock-analysis): GBrain probe results [Step 2]"
然后 ping 我等下一步指示。
```

---

## 6. 实测结果（Hermes 执行后回填）

### 6.1 工具清单（来自 SCENARIO 1）

| 工具名 | 类型 | 输入 schema 摘要 | 备注 |
|---|---|---|---|
| _待填_ | _read/write_ | _..._ | _..._ |

### 6.2 验证点矩阵

| ID | 验证点 | 结果 | 关键证据 / 现象 | 影响 |
|---|---|---|---|---|
| V1 | 工具清单 | ⬜ PASS / FAIL / UNKNOWN | _执行后填_ | _执行后填_ |
| V2 | 实体类型显式声明 | ⬜ | | |
| V3 | typed link 写法 | ⬜ | | |
| V4 | ingest 幂等性 | ⬜ | | |
| V5 | 基础 query 结构 | ⬜ | | |
| V6 | **字段级精确过滤** | ⬜ | | **决定 P1-2 Schema 是否成立** |
| V7 | baseline 隔离 | ⬜ | | **决定 RS-2 落地方式** |
| V8 | 异常返回形式 | ⬜ | | 决定 §3.3.1 降级实现 |

### 6.3 关键问题答案表

| Question | 答案 | 影响下一步 |
|---|---|---|
| Q2.A 实体类型由 LLM 还是规则提取？ | _待填_ | _Schema 写法严格度_ |
| Q2.B 不写 frontmatter 能区分类型吗？ | _待填_ | _frontmatter 是否必填_ |
| Q3.A `type: Stock` 是否被实体化？ | _待填_ | **Schema 必含字段** |
| Q3.B markdown link 形式的 typed link 被识别吗？ | _待填_ | **Schema 关系语法** |
| Q3.C 是否支持 `links:` 数组显式声明？ | _待填_ | _首选语法_ |
| Q3.D typed link 名是否需要预注册？ | _待填_ | _初始化脚本是否必须_ |
| Q4.A id 重 ingest 是 upsert 还是 append？ | _待填_ | **回归脚本要不要先 delete** |
| Q4.B 删除 API 是什么？ | _待填_ | _清理脚本怎么写_ |
| Q5.A 返回有 score 吗？ | _待填_ | _Skill 是否要做阈值过滤_ |
| Q5.B 默认返回数量？分页？ | _待填_ | _query 时要不要传 limit_ |
| Q5.C 能限定 type 吗？ | _待填_ | _精度方案_ |
| **Q6.A 至少一种试法能精确过滤吗？** | _待填_ | **决定 P1-2 是否成立** |
| Q6.B 如不行，备用方案？ | _待填_ | _要不要建本地 index_ |
| Q6.C 嵌套字段能检索吗？ | _待填_ | **决定 modules 是否拍平** |
| Q7.A 能按 source 分组吗？ | _待填_ | _RS-2 落地形式_ |
| Q7.B priority 排序是 prompt 还是召回层？ | _待填_ | _LLM 提示词复杂度_ |
| Q8.A 失败错误码 / 异常类型？ | _待填_ | _try/except 模板_ |
| Q8.B 有 retry 吗？ | _待填_ | _重试队列实现方式_ |

### 6.4 总结性结论（执行完后写）

> _待填_：用 PROPOSAL §5 的"四件套"格式给最终判断：
>
> - 🎯 结论：GBrain 是否足够支撑 P1-2 结构化 Schema 设计？（高/中/低 置信度）
> - 📊 核心指标：成功试法数 / 失败试法数 / unknown 数
> - 🧠 判断逻辑：
> - ⚠️ 反向信号 / 风险：

---

## 7. 收尾

### 7.1 清理脚本（执行完毕后必跑）

```bash
# 通过 GBrain MCP 删除所有 _probe/${TEST_RUN_ID}/ 下的 page
# 具体命令取决于 GBrain 的 delete API（在 SCENARIO 4 / 8 中已摸清）
gbrain delete --prefix "_probe/${TEST_RUN_ID}/"
# 或者用 brain-ops 暴露的批量删除原语
```

### 7.2 Step 2 验收标准（[PLAN.md](./PLAN.md) §第 2 步要求）

- [ ] §6.1 工具清单至少 5 个工具
- [ ] §6.2 V1-V8 全部填完，没有遗留 ⬜
- [ ] §6.3 18 个 Q 全部有答案（"不支持"也是答案）
- [ ] §6.4 给出明确的"GBrain 是否支撑 P1-2"结论
- [ ] 探索过程中产生的所有 _probe/ page 已清理
- [ ] commit message: `chore(stock-analysis): GBrain probe results [Step 2]`

---

## 8. 下一步联动

执行完毕后，根据 §6.4 的结论分支：

### 分支 A：V6（字段级查询）= PASS
→ 直接进 PLAN §第 3 步：按当前 Schema 设计落地，无需妥协。

### 分支 B：V6 = FAIL，但 V2/V3 = PASS
→ Schema 设计需要"双轨"：
- 主轨：markdown page 入 GBrain 走向量召回 + 关系遍历
- 辅轨：本地 SQLite/JSONL 维护一份"判断字段索引"，跨次对比走辅轨
→ PROPOSAL.md §3.4 需要追加"双轨索引"段落

### 分支 C：V2/V3 也 FAIL（GBrain 不能接受我们设想的 frontmatter）
→ 重大方向调整，回到 PROPOSAL 重审 GBrain 选型：
- 可能要把 GBrain 降级为"纯文本笔记仓"
- 或者改用其他更明确支持图谱 schema 的工具

---

## 9. 决策记录（探索过程中如有发现，追加在此）

| 日期 | 发现 | 决策 |
|---|---|---|
| _Hermes 填_ | _..._ | _..._ |
