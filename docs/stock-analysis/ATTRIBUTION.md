# ATTRIBUTION

> 本文件是 [PROPOSAL.md](./PROPOSAL.md) §9 决策"保留 ATTRIBUTION.md 作为唯一合规底线"的落地。
> 所有从外部 MIT/Apache/BSD 等许可项目抽取（cherry-pick）进入本仓库的代码与资产，必须在此登记。
> 形式：来源仓库 + 上游 commit hash + 抽取的文件清单 + 本地落地路径 + 抽取日期 + License。

---

## 1. ZhuLinsen / daily_stock_analysis

- **上游仓库**：https://github.com/ZhuLinsen/daily_stock_analysis
- **上游 License**：MIT License
- **本地参考克隆位置**：`_research/daily_stock_analysis/`（不入 git，已在 `.gitignore` 排除）
- **依据决策**：[PROPOSAL.md](./PROPOSAL.md) §9（2026-05-12 起）/ [PLAN.md](./PLAN.md) §第 1 步（2026-05-14）

### 1.1 抽取记录

| # | 抽取日期 | 上游 commit | 上游路径 | 本地路径 | 行数 | 备注 |
|---|---|---|---|---|---|---|
| 1 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/bottom_volume.yaml` | `skills/stock-analysis/_assets/strategies/bottom_volume.yaml` | 49 | 原样拷贝 |
| 2 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/box_oscillation.yaml` | `skills/stock-analysis/_assets/strategies/box_oscillation.yaml` | 69 | 原样拷贝 |
| 3 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/bull_trend.yaml` | `skills/stock-analysis/_assets/strategies/bull_trend.yaml` | 49 | 原样拷贝 |
| 4 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/chan_theory.yaml` | `skills/stock-analysis/_assets/strategies/chan_theory.yaml` | 58 | 原样拷贝 |
| 5 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/dragon_head.yaml` | `skills/stock-analysis/_assets/strategies/dragon_head.yaml` | 44 | 原样拷贝 |
| 6 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/emotion_cycle.yaml` | `skills/stock-analysis/_assets/strategies/emotion_cycle.yaml` | 79 | 原样拷贝 |
| 7 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/ma_golden_cross.yaml` | `skills/stock-analysis/_assets/strategies/ma_golden_cross.yaml` | 46 | 原样拷贝 |
| 8 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/one_yang_three_yin.yaml` | `skills/stock-analysis/_assets/strategies/one_yang_three_yin.yaml` | 37 | 原样拷贝 |
| 9 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/shrink_pullback.yaml` | `skills/stock-analysis/_assets/strategies/shrink_pullback.yaml` | 46 | 原样拷贝 |
| 10 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/volume_breakout.yaml` | `skills/stock-analysis/_assets/strategies/volume_breakout.yaml` | 47 | 原样拷贝 |
| 11 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/wave_theory.yaml` | `skills/stock-analysis/_assets/strategies/wave_theory.yaml` | 64 | 原样拷贝 |
| 12 | 2026-05-14 | `a75a0c502e06a49f08439275aaeb7be241c5befe` | `strategies/README.md` | `skills/stock-analysis/_assets/strategies/_UPSTREAM_README.md` | — | 上游策略说明，重命名以避免与本目录将来自有 README 冲突 |

**合计**：11 个 YAML 策略文件（588 行） + 1 个上游 README。

### 1.2 GBrain 入脑约束（详见 PLAN §第 1 步）

11 个策略 ingest 进 GBrain 时必须带：

```yaml
source: baseline           # 来自 ZhuLinsen 上游，非用户自定义
priority: reference        # 仅作为 LLM 推理时的参考视角，不覆盖 user_custom
upstream_commit: a75a0c502e06a49f08439275aaeb7be241c5befe
```

避免 baseline 抑制用户自己交易策略的生长（详见 [REVIEW.md](./REVIEW.md) RS-2）。

### 1.3 后续 Step 计划抽取的内容（待登记）

下列条目按 [PLAN.md](./PLAN.md) 节奏抽取，每步完成后立刻补登：

- Step 4：`src/services/stock_code_utils.py`、`src/services/name_to_code_resolver.py`（必抽）；`src/patches/eastmoney_patch.py`（候选）
- Step 5：`data_provider/akshare_fetcher.py`、`data_provider/efinance_fetcher.py`、`data_provider/base.py`（精简版）
- Step 6：`src/stock_analyzer.py`（精简骨架，~847 行，需重写 LLM 层与渲染层）

### 1.4 不抽取（已在 PROPOSAL §9 决策）

- ❌ `src/agent/`（10099 行）—— 与 Hermes AgentTeam 功能重复
- ❌ `src/analyzer.py`（3223 行）—— 耦合 LiteLLM + Config + Storage，包袱过重
- ❌ `apps/` / `api/` / `bot/` —— 应用层，本项目不需要

---

## 2. License 全文存档

### 2.1 ZhuLinsen / daily_stock_analysis (MIT)

```
MIT License

Copyright (c) 2024 ZhuLinsen

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

> 上游 LICENSE 文件（如有更新）可在 `_research/daily_stock_analysis/LICENSE` 中实时核对。

---

## 3. 维护规则

1. **每次抽取必登记**：哪怕只抽 1 行，也要在 §1.1 表格里加一条记录
2. **commit hash 不可省**：精确到 commit 才能在上游变更时知道我们抽的是哪个版本
3. **变更要追加**：如果某个文件后续做了二次抽取或重新同步，新增一行（不修改旧行），保留时间线
4. **删除需声明**：如果某个抽取被废弃，加 `状态: removed` 字段而不是直接删行

---

## 4. 决策追溯

- 2026-05-12 [PROPOSAL.md](./PROPOSAL.md) §9：以 ZhuLinsen 为 Cherry-pick 基线（深度整合，不留 vendored）
- 2026-05-12 [PROPOSAL.md](./PROPOSAL.md) §9：保留 ATTRIBUTION.md 作为唯一合规底线
- 2026-05-14 [PLAN.md](./PLAN.md) §第 1 步：执行首批抽取（11 个 strategies/*.yaml）
- 2026-05-17 本文件首次创建
