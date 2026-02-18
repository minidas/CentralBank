# CentralBank 变量词典与实现映射（Vic3）

> 文档目标：面向多人协作，把**现有可用变量**与**设计中待实现变量**统一到一个词典中，避免语义冲突。  
> 范围：`JE 信用货币系统`、`银行建筑/PM`、`货币法律`、`每周债务循环`。

---

## 0. 命名与单位约定（协作必读）

### 0.1 命名后缀约定
- `_stock`：存量（例如债务总量）。
- `_flow_weekly`：周流量（例如周新增贷款）。
- `_rate`：比率（0~1 或可超过1，需注释）。
- `_factor`：乘数系数。
- `_idx`：指数/评分（通常有上下限）。
- `_cap`：硬上限。
- `_target`：目标值（由公式得出，不一定直接存储）。

### 0.2 单位约定
- 货币存量：`£`
- 周流量：`£/week`
- GDP 增速、利率、准备金率：`rate`（建议十进制，如 0.05）
- 评分：无单位（需标注范围）

### 0.3 关键原则
1. **禁止同名“变量 vs script_value”**（当前 `leverage_target` 已冲突，必须拆分）。
2. 所有分母变量必须最小值保护（`max(x, epsilon)` 思路）。
3. 周期写入必须写明来源：`weekly pulse` / `half-year pulse` / `law tick`。

---

## 1. 当前已实现变量（As-Is）

> 来自当前仓库脚本，供维护与回归。

| 变量名 | 建议新名 | 当前初始化 | 当前更新 | 周期 | 单位 | 说明 |
|---|---|---|---|---|---|---|
| `bank_reserves` | `bank_reserves_stock` | JE immediate=0 | 暂无自动更新 | 初始化后静态 | £ | 用于 `debt_bound`。 |
| `total_debt` | `total_debt_stock` | JE immediate=0 | 每周：加利息 + 加目标变化量 | weekly | £ | 当前系统核心债务存量。 |
| `leverage_target` | `leverage_target_var`（或删除） | JE immediate=0 | 暂无自动更新 | 初始化后静态 | ratio | 与同名 `script_value` 冲突。 |
| `turnover_amount` | `loan_velocity_cap_weekly` | JE immediate=0 | 暂无自动更新 | 外部驱动 | £/week | 当前作为增长情景借贷上限近似量。 |
| `this_season_gdp` | `gdp_halfyear_current` | JE immediate=gdp | `gdp_record` 每半年刷新 | half-yearly | £/halfyear | 当前半年GDP快照。 |
| `last_season_gdp` | `gdp_halfyear_last` | JE immediate=gdp | `gdp_record` 每半年刷新 | half-yearly | £/halfyear | 上期GDP快照。 |
| `confidence_factor` | `confidence_idx` | JE immediate=1 | 暂无自动更新 | 外部驱动 | factor | 放大GDP增长目标。 |
| `reserves_rate` | `reserve_ratio_rate` | JE immediate=0.1 | 暂无自动更新 | law/event | rate | 法律核心参数之一。 |
| `credit_level` | `creditworthiness_idx` | JE immediate=0 | 暂无自动更新 | law/event | idx | 风险溢价修正输入。 |
| `bank_interest_rate_factor` | `bank_policy_rate_factor` | JE immediate=1 | 暂无自动更新 | PM/law | factor | 银行PM利率乘数。 |
| `target_debt_change` | `debt_delta_target_weekly` | 每周设置为`debt_change` | 递归修正后入账 | weekly | £/week | 周债务变化目标。 |

---

## 2. 当前 script_values（As-Is）

| 名称 | 建议新名 | 公式摘要 | 单位 | 备注 |
|---|---|---|---|---|
| `debt_bound` | `loan_limit_cap` | `bank_reserves / reserves_rate + 1` | £ | `+1`是防除零补丁。 |
| `leverage_rate` | `leverage_ratio` | `total_debt / gdp` | ratio | 与设计一致。 |
| `trade_deficit_factor` | `trade_balance_factor` | 常量1（占位） | factor | 待接货币法律。 |
| `risk_premium_factor` | `risk_premium_factor` | `(4-credit_level)^leverage_rate` | factor | 建议替换为可调曲线。 |
| `real_interest_rate` | `real_interest_rate` | `0.04 * bank_factor * risk_premium * trade_factor` | rate | 当前基准4%。 |
| `leverage_target` | `leverage_ratio_target` | `(total_debt + delta)/debt_bound` | ratio | 与变量同名冲突。 |
| `target_risk_premium` | `risk_premium_factor_target` | `(4-credit_level)^leverage_target` | factor | 目标状态溢价。 |
| `target_interest_rate` | `target_interest_rate` | `0.04 * bank_factor * target_risk * trade_factor` | rate | 目标利率。 |
| `gdp_growth_rate` | `gdp_growth_rate_annualized` | `((this-last)/last)*2*confidence` | rate | 半年转年化。 |
| `target_debt` | `debt_stock_target` | `total_debt + target_delta` | £ | 目标债务存量。 |
| `weekly_interest` | `interest_cost_weekly` | `total_debt*(real_interest_rate/52)` | £/week | 每周利息。 |
| `debt_change` | `debt_delta_raw_weekly` | 高利率时去杠杆，否则借贷 | £/week | 递归前原始变化量。 |

---

## 3. 按设计文档新增变量（To-Be）

> 下表是建议落地的数据层；多人开发时请先建“空变量 + 注释”，再分功能迭代。

### 3.1 银行建筑聚合（由 building/PM 聚合到国家）

| 新变量（建议） | 类型 | 单位 | 来源 | 用途 |
|---|---|---|---|---|
| `bank_bonds_output_weekly` | flow | £/week | 银行建筑债券产出总和 | 表征银行盈利能力（非国家债务）。 |
| `bank_reserves_stock` | stock | £ | 银行各级被动效果聚合 | 贷款上限分子。 |
| `loan_velocity_cap_weekly` | cap | £/week | 银行各级被动效果聚合 | 每周新增贷款硬上限。 |
| `bank_policy_rate_factor` | factor | - | 银行PM（极低~极高利率） | 利率政策乘数。 |

### 3.2 债务循环核心

| 新变量（建议） | 类型 | 单位 | 公式/来源 | 说明 |
|---|---|---|---|---|
| `total_debt_stock` | stock | £ | 现有 `total_debt` | 社会总信贷债务。 |
| `leverage_ratio` | derived | ratio | `total_debt_stock / GDP` | 风险核心状态量。 |
| `base_rate_law` | rate | rate | 货币法律 + 贸易状态 | 法律决定的基础利率。 |
| `real_interest_rate` | derived | rate | `base_rate_law * bank_policy_rate_factor * risk_premium_factor` | 实际利率。 |
| `debt_base_theoretical` | derived | £ | `r == g*4`时反推 | 设计中的“基准债务值”。 |
| `confidence_idx` | idx/factor | - | 政治稳定/忠诚派/增长 | 债务目标放大器。 |
| `debt_stock_target` | derived | £ | `debt_base_theoretical * (1 + confidence_idx)` | 目标债务。 |
| `loan_limit_cap` | cap | £ | `bank_reserves_stock / reserve_ratio_rate` | 债务硬上限。 |
| `debt_delta_target_weekly` | flow | £/week | min(目标差额, 周转上限) | 当周借贷/还款目标。 |

### 3.3 货币法律与外部平衡

| 新变量（建议） | 类型 | 单位 | 来源 | 说明 |
|---|---|---|---|---|
| `net_exports_balance` | flow | £/week | 出口-进口 | 贸易顺逆差输入。 |
| `exchange_rate_idx` | idx | - | 现代货币法律下启用 | 汇率指数。 |
| `reserve_ratio_rate` | rate | rate | 法律 | 准备金率（5%~40%等）。 |
| `capital_control_cost_auth_weekly` | flow | authority/week | 资本管制法律 | 每周权威消耗。 |
| `fiat_confidence_penalty` | factor | - | 汇率过低时触发 | 压低信用/信心。 |

---

## 4. 每周 Tick 标准流程（目标实现版）

1. 更新银行聚合量：`bank_reserves_stock`、`loan_velocity_cap_weekly`、`bank_policy_rate_factor`。  
2. 计算法律基础利率：`base_rate_law`（受法律+贸易/汇率影响）。  
3. 计算风险溢价：`risk_premium_factor = f(leverage_ratio, creditworthiness_idx)`。  
4. 得到实际利率：`real_interest_rate = base_rate_law * bank_policy_rate_factor * risk_premium_factor`。  
5. 利息累积：`total_debt_stock += total_debt_stock * (real_interest_rate / 52)`。  
6. 计算目标债务：`debt_stock_target = debt_base_theoretical * (1 + confidence_idx)`。  
7. 借/还判定：
   - 若 `total_debt_stock < debt_stock_target`：
     - `borrow = min(debt_stock_target-total_debt_stock, loan_velocity_cap_weekly)`。
   - 若 `total_debt_stock > debt_stock_target`：
     - `repay = min(total_debt_stock-debt_stock_target, investment_pool_available)`。
8. 应用硬上限：`total_debt_stock <= loan_limit_cap`。超出则强制截断并触发警报事件。  
9. 写入UI/报警变量：危机等级、偿付压力、投资池吸血状态。

---

## 5. PM 与法律的配置映射建议

### 5.1 银行利率PM（建议）
| PM | `bank_policy_rate_factor` | 债券产出倾向 | 玩法含义 |
|---|---:|---|---|
| 极低利率 | 0.50 | 极少 | 促投资、弱银行利润 |
| 低利率 | 0.75 | 少 | 温和刺激 |
| 基础利率 | 1.00 | 中 | 中性 |
| 高利率 | 1.50 | 多 | 抑制借贷、高利润 |
| 极高利率 | 2.00 | 极多 | 危机手段 |

### 5.2 货币法律（建议）
| 法律 | 关键开关 | 主要变量影响 |
|---|---|---|
| 金本位 | 贸易顺差↑→基础利率↑ | `base_rate_law`, `reserve_ratio_rate`偏高 |
| 固定汇率 | 波动较小，有限干预 | `base_rate_law`低波动 |
| 资本管控 | 利率与贸易脱钩；每周权威消耗 | `capital_control_cost_auth_weekly` |
| 现代货币 | 启用汇率机制 | `exchange_rate_idx`, `fiat_confidence_penalty` |

---

## 6. 数值标定参考（来自你的草案，转为脚本参数）

### 6.1 杠杆率→利率乘数
| 杠杆率 | 乘数 |
|---:|---:|
| 0.00 | 1.00 |
| 0.25 | 1.41 |
| 0.50 | 2.00 |
| 0.75 | 2.82 |
| 1.00 | 4.00 |

> 可近似为 `risk_premium_factor = 2^(2*leverage_ratio)`，并再由 `creditworthiness_idx` 做减免。

### 6.2 参数初值建议
- `base_rate_law = 0.05`
- `reserve_ratio_rate`：金本位 0.40；现代货币 0.05~0.10
- `confidence_idx`：默认 0（繁荣上限可到 0.5~1.0 视平衡而定）

---

## 7. 代码落地任务拆分（多人并行）

1. **数据层**（A同学）：补齐 To-Be 变量声明与初始化。  
2. **银行层**（B同学）：银行建筑/PM 产出债券与国家聚合量写入。  
3. **债务循环层**（C同学）：周Tick借还逻辑 + 上限截断 + 投资池交互。  
4. **法律层**（D同学）：货币法律对 `base_rate_law` / `reserve_ratio_rate` / `exchange_rate_idx` 的修正。  
5. **UI与告警层**（E同学）：JE进度条、危机预警、调试信息面板。  

---

## 8. 对应源码（当前版本）
- `common/journal_entries/je_credit_currency.txt`
- `common/script_values/credit_currency_functions.txt`
- `common/scripted_effects/debt_change_calculator.txt`
- `common/on_actions/debt_change.txt`
- `common/production_methods/bank_pm.txt`
- `common/production_method_groups/bank_pmg.txt`
- `common/buildings/z_building_bank.txt`
