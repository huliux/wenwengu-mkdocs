# DCF 滚动重评实践指南（季度/中报更新版）

## 结论速览
- 季报/中报发布后 **可以且建议进行滚动重评**。核心是把“估值日”与“基期口径”同步到最新报表，然后用一致口径重算经营与资本结构两条线。
- “基期现金”用于 EV→Equity 桥的净债务计算，应取 **估值日的现金及现金等价物**（剔除受限现金），而不是平均值或年末数。
- 本项目已支持滚动重评的关键要素：
  - 年中折现（Mid-Year Convention）
  - WACC 增强 CAPM（CRP/IRP）与“目标/市场价值”两种权重模式
  - NWC 分项口径基期（`last_component_nwc`）与日数平滑过渡（`nwc_days_transition_years`）
- 股权桥以“最新资产负债表”一行计算净债务
- 股息与 as-of 展示（2025-09-10）：
  - TTM 股息优先采用“实施/完成”，没有则“TTM·预案”，仍没有则“年度最近一次”；UI 标签同步标注来源
  - 最新价格按 `trade_date ≤ 估值日` 回退，EPS（年报）按 `end_date ≤ 估值日` 回退；基本信息中显示 as-of
- 若要“完全 LTM（过去12个月）基期”的经营滚动，需要很小的抓取与处理扩展；当前已可实现“资本结构滚动+年中折现+NWC平滑”的规范重评。

## 重评的两种方式
- 快速重评（仅市场侧滚动）
  - 估值日设到季报/中报披露后，更新：股价/总股本（E）、Rd/Rf/β/MRP/CRP/IRP、终值假设；勾选年中折现。
  - EV 变化主要受 WACC 与 TV 驱动；净债务与经营基期仍是上次年报口径（快速但粗略）。
- 规范重评（经营+资本结构滚动）
  - 估值日设到季报/中报披露日（或接近的交易日）。
  - NWC 基期用分项口径 `last_component_nwc`，并设置 `nwc_days_transition_years`（建议 3 年）从 `last_*_days` → 历史中位数日数平滑过渡，避免首年 ΔNWC 跳变。
  - 净债务：以“最新资产负债表”一行的 **有息债务 − 现金及现金等价物** 计算（估值日口径）。
  - 勾选年中折现；TV 参数按最新判断重估。

## 关键口径与公式
- UFCF 不含现金：NWC 以经营性科目计算（现金不进 NWC）。
- 净债务（估值日）：`NetDebt = Interest-bearing Debt − Cash & Equivalents`。
- WACC：
  - `Ke = Rf + β × (MRP + CRP) + Size + IRP`
  - `Kd_AT = Kd × (1 − Tax)`（建议用边际税率；系统当前用目标有效税率）
  - `WACC = w_e × Ke + w_d × Kd_AT`
  - 权重模式：
    - 目标结构：`w_d = target_debt_ratio`，`w_e = 1 − w_d`。
    - 市场价值：`w_d = D/(D+E)`（`E=市值=股价×总股本`；`D≈有息债账面值近似市场价值`）。
- 折现时点：可启用年中折现（Mid-Year），等价于指数用 `(year − 0.5)`。

## 在当前系统如何操作
- 估值日：在 UI 选择中报/季报之后的日期（或你指定的估值日）。
- 勾选“使用年中折现（mid_year_convention）”。
- NWC 基准口径：选择 `component`（分项口径，推荐），并设置“天数过渡年数”=3（可调）。
- WACC 权重模式：
  - 若强调贴近现状，选“使用最新市场价值计算权重（market）”。
  - 若结构预计回归常态/目标，选“使用目标债务比例（target）”。
- CRP/IRP：根据公司市场/行业设置（.env 或 UI 输入），Ke 将自动纳入。
- 终值方法：退出乘数或永续增长率按最新判断设定（后续将加入名义 GDP 上限校验与常驻提示）。
- 调试区检查：
  - 请求载荷摘录（含 `nwc_baseline_mode`、`nwc_days_transition_years`、`mid_year_convention`）
  - 映射后假设、历史中位数（含 `last_component_nwc`、`last_*_days`）
  - 首年预测关键项（`delta_nwc`、`ufcf` 等）

## 基期现金的选取（重评必读）
- 取“估值日时点”的现金及现金等价物（剔除受限现金）。
- 若只有年末现金，可用：`期初现金 + 自由现金流（YTD 经营/投资/筹资净额） ± 重大事项` 回滚至估值日。
- 与 NWC 一致性：现金不进 NWC，使用分项 NWC 基期即可消除 ΔNWC 的口径跳变。

## LTM（过去12个月）支持
- end_type 口径映射：`1=Q1`、`2=H1`、`3=Q3`、`4=年度`（均为当年 YTD 口径）。
  - 新增 UI/API 开关：`ltm_baseline_enabled`（默认 False）。开启后：
  - 经营基期（last_actual_revenue）采用 LTM 换算：
    - 若最新为年报（end_type=4）：LTM=当年年报；
    - 若为 1/2/3：`LTM = YTD_curr + (Annual_prev − YTD_prev)`（分别对 `revenue` 与 `total_revenue` 计算，优先采用 `revenue`）。
  - 资本结构与 NWC 基期：`latest_balance_sheet` 指向估值日之前最近一期；`last_component_nwc` 基于该行重算，避免首年口径跳变。
- 回退策略：如缺上年同口径（YTD_prev）或上年年报（Annual_prev），则回退为“年报基期”；调试区会提示。
- 保持一致性：
  - 历史中位数（比率/周转）仍基于年报历史序列提取；
  - 首年 `last_*_days` 使用历史中位数并按设置过渡年数从“最新一期实际值”平滑（如可得）。

备注：不开启 LTM 时，系统沿用“年报基期 + 最新资产负债表 + 年中折现 + NWC 平滑”的规范重评。

## 口径与显示（2025-09-10 更新）
- 基本信息 caption 显示：收入基期（LTM/年报）+ 资产负债表基准日
- 最新价格标签：带 as-of（估值日或小于等于估值日的最近交易日）
- 每股收益标签：显示“年报YYYY”（≤估值日的最近年报年份）
- 股息标签：依据来源显示“(TTM)/(TTM·预案)/(年度)”

## 何时选哪种 WACC 权重更科学
- 市场价值权重（market）：贴近现状的即期估值；但受市价波动影响大。
- 目标债务比例（target）：适合结构将回归目标/行业均衡、或现状明显偏离的情况。
- 最佳实践：两者交叉核对；若差异显著，建议做敏感性分析并说明理由。

## 常见陷阱与建议
- 不要用“资产负债率”（账面口径）代替 WACC 中的“债务比例”（市值口径的有息债权重）。
- 不要用平均现金或年末现金代替“估值日现金”；股权桥中的净债务必须是估值日口径。
- 年中不改估值日而直接插入中报数据，会造成时间错配；建议切估值日或使用年中折现/短期分段（stub）。
- 纳入租赁负债（IFRS16）时，要与 UFCF/EBITDA 口径配套，避免双重调整或遗漏。

## 重评清单（Checklist）
- [ ] 估值日选择并对齐所有数据
- [ ] mid_year_convention 勾选
- [ ] nwc_baseline_mode=component；nwc_days_transition_years=3（建议）
- [ ] WACC 权重模式与 Ke 参数（Rf/β/MRP/CRP/IRP）更新
- [ ] 终值参数（退出乘数/PGR）更新；（后续：名义 GDP 上限校验）
- [ ] 净债务：估值日有息债务与现金（剔除受限）口径一致
- [ ] 调试区：请求/映射/历史中位数/首年预测核对 ΔNWC 与 UFCF 合理性

## API 字段速查（与 UI 对应）
- `mid_year_convention: bool`
- `ltm_baseline_enabled: bool`
- `nwc_baseline_mode: component|aggregate`
- `nwc_days_transition_years: int`
- `wacc_weight_mode: market|target`, `target_debt_ratio`
- `cost_of_debt`, `risk_free_rate`, `beta`, `market_risk_premium`, `country_risk_premium`, `industry_risk_premium`
- `terminal_value_method: exit_multiple|perpetual_growth`, `exit_multiple`, `perpetual_growth_rate`

---

最后更新：2025-09-10
