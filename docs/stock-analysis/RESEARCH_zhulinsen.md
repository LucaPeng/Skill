# ZhuLinsen/daily_stock_analysis 代码考古报告

> 调研时间：2026-05-12
> 仓库 commit：a75a0c502e06a49f08439275aaeb7be241c5befe (2026-05-13)
> License：MIT ✅
> 调研定位：评估"哪些代码值得 cherry-pick 进我们的个股分析 Skill 集合"

---

## 1. 仓库整体认知（必读）

### 1.1 它实际上是一个"完整的产品系统"，不是"几个分析脚本"

| 层 | 内容 | 规模 |
|---|---|---|
| 前端 | `apps/dsa-web` (React) + `apps/dsa-desktop` (Electron) | 数百个组件 |
| 后端 API | `api/` (FastAPI 完整服务) | 11 个 endpoint 模块 |
| 主流程入口 | `main.py` (35506 字节) + `webui.py` + `server.py` | 主流程几千行 |
| Bot 平台 | `bot/` (企微/飞书/Telegram/Discord/Slack/邮箱) | 多平台适配 |
| 数据源 | `data_provider/` (7 个 fetcher) | **10999 行** |
| Agent 框架 | `src/agent/` (orchestrator + agents + skills + tools + strategies) | **10099 行** |
| 业务服务 | `src/services/` (21 个服务文件) | **~14000 行** |
| 核心引擎 | `src/core/` (pipeline + market_review + backtest + ...) | pipeline 单文件 2234 行 |
| 分析器 | `src/analyzer.py` (3223 行) + `stock_analyzer.py` (847 行) + `market_analyzer.py` (1232 行) | **5300+ 行** |
| 策略 | `strategies/*.yaml` (11 个) | 纯 Prompt 配置，588 行 |
| 模板 | `templates/*.j2` (4 个) | Markdown 报告渲染 |
| 修补 | `src/patches/eastmoney_patch.py` | 东财接口补丁 |
| 测试 | `tests/` (133 项) | 覆盖完善 |

**Python 主代码估算：~50000 行级别**。

> ⚠️ 这跟我们最初猜测"~3000-5000 行可抽取"完全不在一个量级。

### 1.2 它**自带**一个 SKILL.md，但跟我们想做的 Skill 不是一回事

仓库根目录的 `SKILL.md` 是给 **Claude Code 用** 的"调用本项目服务"指南——它的本质是：

```
SKILL.md → 调用 src/services/analyzer_service.py 的 Python 函数
```

也就是说，**他们的 "Skill 化" = 把整个产品当作一个黑盒服务，用 SKILL.md 教 AI 怎么调它**。这跟我们要做的"由多个轻量 Skill 在 Hermes 里编排"是完全不同的设计。

### 1.3 .claude/skills/ 目录是个误导

`.claude/skills/` 里的 `analyze-issue / analyze-pr / fix-issue` —— 这是给项目自身的开发流程用的（处理 GitHub Issue 和 PR），**不是股票分析 Skill**。

---

## 2. 模块价值评估（核心）

按"可拿性 × 价值密度"两个维度打分。

### 🟢 可直接 Cherry-pick（高价值 + 低耦合）

#### ⭐⭐⭐⭐⭐ `strategies/*.yaml` (11 个策略 Prompt)
- **大小**：588 行 yaml，纯文本
- **依赖**：零依赖，独立配置文件
- **价值**：缠论 / 波浪 / 均线金叉 / 情绪周期 / 龙头战法 / 一阳三阴 / 缩量回踩 / 量价突破 / 牛熊趋势 / 箱体震荡 / 底部放量
- **能力**：每个策略都包含 **判断规则 + 工具调用 + 评分调整建议** 的完整 Prompt
- **如何用**：直接作为我们 Skill 的 Prompt 模板素材库；用户后期能从这 11 个里挑选/魔改成自己的判断框架
- **抽取动作**：复制到 `skills/_templates/strategies/`，按需删改

#### ⭐⭐⭐⭐⭐ `templates/report_*.j2` (Jinja2 模板)
- **大小**：267 行
- **依赖**：Jinja2 + 数据结构（`AnalysisResult`）
- **价值**：核心结论 / 数据透视 / 情报 / 作战计划 四段式，含 emoji 标签和多语言
- **如何用**：参考其结构设计我们的 HTML 模板（不直接用 Markdown，因为我们要 HTML）
- **抽取动作**：作为"输出结构设计"的参考蓝本

#### ⭐⭐⭐⭐ `src/patches/eastmoney_patch.py`
- **大小**：单文件
- **价值**：东财接口的实战补丁经验，绕坑必备
- **抽取动作**：直接复制，必要时简化

---

### 🟡 可抽取但需要解耦（高价值 + 中耦合）

#### ⭐⭐⭐⭐ `data_provider/` (10999 行，7 个 fetcher)
- **价值**：AkShare / Tushare / Pytdx / Baostock / YFinance / EFinance / Longbridge / TickFlow 全套
- **耦合点**：
  - `base.py` (2625 行) 是个 `DataFetcherManager`，做降级/熔断/重试，**重要但复杂**
  - 每个 fetcher 都依赖 `src.config`、`src.storage`、`src.logging_config`
  - 部分依赖 `src.repositories`（数据库持久化）
- **建议拆分**：
  - **拿**：`akshare_fetcher.py`、`efinance_fetcher.py`（最常用，A 股核心）
  - **拿**：`base.py` 简化版（保留 manager + 降级逻辑，去掉 DB 持久化）
  - **暂不拿**：`tushare_fetcher.py`（要 token）、`longbridge_fetcher.py`（港美股）、`tickflow_fetcher.py`（要注册）
- **改造工作量预估**：**1-2 天**（去掉 config/storage/repositories 依赖，写一个最小化的 manager）

#### ⭐⭐⭐ `src/stock_analyzer.py` (847 行)
- **价值**：完整的 **趋势分析器**（多周期均线 + 乖离率 + 量能分析 + 趋势状态分类）
- **核心代码**：`StockTrendAnalyzer` 类
- **耦合点**：依赖 `src.config`，但相对独立
- **建议**：**直接 cherry-pick**，去掉 `Config` 依赖（用默认参数代替）
- **改造工作量**：半天

#### ⭐⭐⭐ `src/market_analyzer.py` (1232 行)
- **价值**：**大盘复盘**（板块表现 / 指数点评 / 板块主线）
- **耦合点**：依赖 `src.search_service`、`src.config`、`data_provider.base`、`src.core.market_profile`、`src.core.market_strategy`
- **建议**：**有选择地拿**，主要是它的 Prompt 部分和"板块主线提取"逻辑
- **改造工作量**：1-2 天（去耦合 + 改造为板块轮动分析）

---

### 🔴 不建议拿（包袱重 / 价值密度低）

#### ❌ `src/analyzer.py` (3223 行)
- 这是**他们的"中央分析器"**，绑定了：LiteLLM 路由 + 整套 Config + Storage + 多模型适配 + LLM 用量持久化 + 报告语言系统
- 拿过来等于把他们一半的基建拖过来
- **建议**：不要。我们重写一个更轻的，只调 LLM、不管路由

#### ❌ `src/agent/` 整套 Agent 框架 (10099 行)
- 包含 orchestrator / executor / runner / memory / events / llm_adapter / agents / skills / strategies / tools
- **这是一个完整的 Agent 框架**，跟 Hermes 的 AgentTeam 是**重复的轮子**
- **建议**：不要。我们用 Hermes 的能力即可
- **唯一可借鉴**：`src/agent/strategies/` 里的策略调用机制（思路）、`src/agent/tools/data_tools.py` 中的"工具签名设计"（思路）

#### ❌ `src/services/` 大部分 (~14000 行)
- portfolio / backtest / history / notification / task_queue / system_config 全部不要
- **唯一可看**：`analyzer_service.py` (134 行) 学习"如何聚合调用各个分析模块"的设计思路
- **唯一可拿**：`stock_code_utils.py` (102 行)、`name_to_code_resolver.py` (223 行) — 股票代码归一化工具

#### ❌ `src/core/pipeline.py` (2234 行)
- 全产品的核心流水线，绑定了 DB / 调度 / 通知 / 历史 / Config
- **不要**

#### ❌ 整个 `apps/`、`api/`、`bot/`、`docker/`、`scripts/`
- 都是产品壳层，与我们目标无关

#### ❌ `src/agent/skills/defaults.py` 中的 `CORE_TRADING_SKILL_POLICY_ZH`
- 等等！这个**值得看一眼**——它是他们的"核心交易策略"Prompt 常量
- 如果是高价值 Prompt，单独拿出来即可

---

## 3. 修正后的 Cherry-pick 清单

### ✅ 第一批（启动期就要的，~3-5 天工作量）

| # | 来源 | 去向 | 改造程度 |
|---|---|---|---|
| 1 | `strategies/*.yaml` (全 11 个) | `skills/_templates/strategies/` | 复制即用 |
| 2 | `data_provider/akshare_fetcher.py` | `skills/_lib/data/akshare.py` | 去 config 依赖 |
| 3 | `data_provider/efinance_fetcher.py` | `skills/_lib/data/efinance.py` | 去 config 依赖 |
| 4 | `data_provider/base.py`（简化版） | `skills/_lib/data/manager.py` | **大改**，只保留降级 |
| 5 | `src/patches/eastmoney_patch.py` | `skills/_lib/data/patches.py` | 复制即用 |
| 6 | `src/stock_analyzer.py` | `skills/_lib/analysis/trend.py` | 去 config 依赖 |
| 7 | `src/services/stock_code_utils.py` + `name_to_code_resolver.py` | `skills/_lib/utils/code.py` | 复制即用 |

### ✅ 第二批（第一阶段后期补充）

| # | 来源 | 用途 |
|---|---|---|
| 8 | `src/market_analyzer.py` 的 **Prompt 部分** | 板块轮动分析的 Prompt 蓝本 |
| 9 | `templates/report_markdown.j2` 的**结构设计** | HTML 报告设计参考（不直接用） |
| 10 | `src/agent/skills/defaults.py` 中的核心策略常量 | Prompt 素材 |

### ❌ 明确不拿

| # | 不拿的 | 理由 |
|---|---|---|
| - | `apps/`、`api/`、`bot/` | 产品壳层 |
| - | `src/agent/` 整套 | 与 Hermes AgentTeam 功能重复 |
| - | `src/analyzer.py` | 耦合过重 |
| - | `src/core/pipeline.py` | 流程框架 |
| - | `src/services/` 大部分 | 业务壳层 |
| - | `tushare/longbridge/tickflow` fetcher | 要 token / 注册 |

---

## 4. 关键风险与对策

### ⚠️ 风险 1：解耦比想象的难
- 这个项目的 `Config` 单例渗透到几乎所有模块
- 拿任何一个文件，都要先看它 import 了哪些 `src.config` 里的东西
- **对策**：写一个轻量 `MiniConfig` 适配层，把它们用到的配置项都打桩成默认值

### ⚠️ 风险 2：数据源 fetcher 的依赖链
- 比如 `akshare_fetcher` 可能依赖 `repositories`（持久化）和 `storage`（缓存）
- 拿过来要写个内存版的"假持久化"
- **对策**：抽取后第一件事是写一个最小可跑通的 demo（拿一只股票数据），验证依赖闭环

### ⚠️ 风险 3：作者代码量级远超我们设想
- **总规模 ~50000 行 Python**
- 真正"金子" ~3000 行（fetcher 核心 + 趋势分析器 + 策略 yaml）
- **比例不到 10%**
- **对策**：严格按清单拿，不要"顺便也带一下"

### ⚠️ 风险 4：MIT 合规底线
- 你已经决定深度整合不留 vendored
- **必须保留**：`docs/stock-analysis/ATTRIBUTION.md` 这一个文件，列出 ZhuLinsen 仓库 + commit + MIT
- 这是合规底线，也是你良心的备份

---

## 5. 推荐路径

```
[第 1 步] 先抽 strategies/*.yaml 这批"零依赖宝藏"
         → 这一步不会失败，立刻有产出
         → 同时把 ATTRIBUTION.md 建起来

[第 2 步] 抽 stock_code_utils.py + name_to_code_resolver.py
         → 工具类，没什么依赖，先垫底层

[第 3 步] 真正的硬骨头：抽 data_provider
         → 先拿 akshare_fetcher.py + base.py 简化版
         → 写个最小 demo："拿到 600519 近 60 日 K 线"
         → 跑通了，再陆续加 efinance 等

[第 4 步] 抽 stock_analyzer.py（趋势分析器）
         → 把 K 线数据喂进去，输出趋势判断

[第 5 步] 包装成第一个 Skill
         → 这时候，MVP 的 stock-fetch-basic 雏形就有了
```

---

## 6. 一句话结论

> **这个仓库的"信息获取与分析能力"确实很强大，但它是"完整产品"，不是"工具库"。**
>
> **可直接拿的金子约 3000 行（占总量 6%），其余 94% 是基建/产品壳/重复轮子，不应迁入。**
>
> **建议按清单严格 cherry-pick，分 5 步推进，从零依赖配置文件抽起，逐步啃硬骨头。**

---

## 7. 待用户决策的问题

| # | 问题 |
|---|---|
| Q1 | 是否认可"严格按清单抽取，预估 ~3000 行核心代码"这个范围？ |
| Q2 | 第 1 步是否就开始干（抽策略 yaml）？还是先确认更多细节？ |
| Q3 | 抽取过程中如果发现"某文件实际比预期更值钱"，是当场加进清单还是先跳过？ |
| Q4 | `ATTRIBUTION.md` 是否同意保留作为合规底线？ |
