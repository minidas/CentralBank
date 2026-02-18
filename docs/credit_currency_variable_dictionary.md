# CentralBank 变量词典（Vic3 模组）

> 适用范围：当前信用货币/债务系统相关脚本。用于多人协作时统一变量语义、单位与刷新节奏。

## 1) 国家变量（`var:*`）

| 变量名 | 初始化位置（默认值） | 主要更新位置 | 刷新频率 | 建议单位/量纲 | 说明 |
|---|---|---|---|---|---|
| `bank_reserves` | `je_credit_currency` immediate（`0`） | 当前版本未见自动更新 | 事件初始化后静态（除非其他脚本修改） | 货币（£） | 银行准备金；用于计算债务上限 `debt_bound = bank_reserves / reserves_rate + 1`。 |
| `total_debt` | `je_credit_currency` immediate（`0`） | JE 每周脉冲：先加 `weekly_interest`，再加 `target_debt_change` | 每周 | 货币（£） | 系统核心债务存量。 |
| `leverage_target` | `je_credit_currency` immediate（`0`） | 当前版本未见直接更新（同名 script_value 会实时计算） | 初始化后静态（变量本体） | 比率（建议无单位） | 当前作为国家变量存在，但主要“目标杠杆”逻辑在同名 script_value 中计算。建议后续避免同名歧义。 |
| `turnover_amount` | `je_credit_currency` immediate（`0`） | 当前版本未见自动更新 | 依赖外部脚本 | 货币（£） | 在 `debt_change` 的增长情景中作为新增债务基数。 |
| `this_season_gdp` | `je_credit_currency` immediate（`gdp`） | `gdp_record`：每半年设置为当前 `gdp` | 半年 | 货币/期（半年） | 本期 GDP（脚本注释语义为“最近半年的 GDP 快照”）。 |
| `last_season_gdp` | `je_credit_currency` immediate（`gdp`） | `gdp_record`：每半年写入上期值 | 半年 | 货币/期（半年） | 上期 GDP；用于增长率计算。 |
| `confidence_factor` | `je_credit_currency` immediate（`1`） | 当前版本未见自动更新 | 依赖外部脚本 | 系数（无单位） | 信心系数；直接放大/缩小 `gdp_growth_rate`。 |
| `reserves_rate` | `je_credit_currency` immediate（`0.1`） | 当前版本未见自动更新 | 依赖外部脚本 | 比率（0-1） | 准备金率；分母变量，建议保持正值并设置最小值保护。 |
| `credit_level` | `je_credit_currency` immediate（`0`） | 当前版本未见自动更新 | 依赖外部脚本 | 等级/评分（建议离散） | 信用等级；进入风险溢价计算 `4 - credit_level`。 |
| `bank_interest_rate_factor` | `je_credit_currency` immediate（`1`） | 当前版本未见自动更新 | 依赖外部脚本 | 系数（无单位） | 银行利率系数；影响实际利率与目标利率。 |
| `target_debt_change` | JE 每周脉冲中设置为 `debt_change` | 递归脚本 `recursive_loan_calculator` 可反复调整；并用于当周入账 | 每周（含递归迭代） | 货币（£/周） | 当周目标债务增量（可正可负）。 |

---

## 2) Script Values（公式/派生量）

> 这些不是持久化变量，而是“按需计算”的表达式。

| 名称 | 定义 | 建议单位/量纲 | 说明 |
|---|---|---|---|
| `debt_bound` | `bank_reserves / reserves_rate + 1` | 货币（£） | 债务上限；`+1` 用于避免极端情况下分母问题。 |
| `leverage_rate` | `total_debt / gdp` | 比率 | 当前杠杆率。 |
| `trade_deficit_factor` | 当前常量 `1` | 系数 | 预留外贸缺口因子（测试位）。 |
| `risk_premium_factor` | `(4 - credit_level) ^ leverage_rate` | 系数 | 风险溢价。 |
| `real_interest_rate` | `0.04 * bank_interest_rate_factor * risk_premium_factor * trade_deficit_factor` | 年化利率（比率） | 实际利率。 |
| `leverage_target` | `(total_debt + target_debt_change) / debt_bound` | 比率 | 目标杠杆率（注意与国家变量同名）。 |
| `target_risk_premium` | `(4 - credit_level) ^ leverage_target` | 系数 | 目标风险溢价。 |
| `target_interest_rate` | `0.04 * bank_interest_rate_factor * target_risk_premium * trade_deficit_factor` | 年化利率（比率） | 目标利率。 |
| `gdp_growth_rate` | `((this_season_gdp - last_season_gdp) / last_season_gdp) * 2 * confidence_factor` | 年化增速（比率） | 用半年 GDP 快照换算年化增速。 |
| `target_debt` | `total_debt + target_debt_change` | 货币（£） | 目标债务存量。 |
| `weekly_interest` | `total_debt * (real_interest_rate / 52)` | 货币（£/周） | 每周利息支出。 |
| `debt_change` | 若 `real_interest_rate > gdp_growth_rate`，则 `-min(investment_pool_gross_income, total_debt)`；否则 `turnover_amount` | 货币（£/周） | 递归修正前的原始债务变化量。 |

---

## 3) 执行顺序（周/半年）

### 周期：`on_weekly_pulse`（JE 内）
1. `total_debt += weekly_interest`
2. `target_debt_change = debt_change`
3. 调用 `recursive_loan_calculator` 对 `target_debt_change` 递归修正
4. `total_debt += target_debt_change`
5. `add_investment_pool = target_debt_change`

### 周期：`on_half_yearly_pulse_country`（on_actions）
1. `last_season_gdp = this_season_gdp`
2. `this_season_gdp = gdp`

---

## 4) 协作约定（建议）

1. **命名避免冲突**：建议将国家变量 `leverage_target` 改名为 `leverage_target_var`（或删去该变量），避免与 script_value 同名导致协作误解。  
2. **单位写入注释**：凡是债务流量变量（如 `target_debt_change`）统一标注“£/周”；存量统一“£”。  
3. **边界保护**：对 `reserves_rate`、`last_season_gdp` 增加最小值保护逻辑，降低除零/爆值风险。  
4. **调试变量分层**：建议新增 `debug_*` 变量存放关键中间量快照，便于多人联调与平衡。  

---

## 5) 来源文件

- `common/journal_entries/je_credit_currency.txt`
- `common/script_values/credit_currency_functions.txt`
- `common/scripted_effects/debt_change_calculator.txt`
- `common/on_actions/debt_change.txt`
