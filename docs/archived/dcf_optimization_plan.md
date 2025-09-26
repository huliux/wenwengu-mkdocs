# DCF 核心业务逻辑优化方案（v1）

## 目标与范围
- 目标：将模型从“参数可调的标准 DCF”升级为“带行业校准、宏观约束、口径统一、可审计”的企业级 DCF。
- 范围：折现时点、WACC 结构、终值治理、利润与成本路径、营运资本口径、透明度与审计、敏感性与情景、数据质量、测试与 UX。

## 核心优化（高优先级）
- 折现时点（Mid-Year Convention）
  - 默认支持年中折现，结果页标注折现惯例。
  - 已落地：`PresentValueCalculator(mid_year_convention)`；Streamlit 复选项：`mid_year_convention`。
- WACC/Ke 结构（CRP / IRP）
  - Ke = Rf + β×(MRP + CRP) + Size + IRP；Rf/β/MRP 支持 UI 覆盖与 ENV。
  - 已落地：`COUNTRY_RISK_PREMIUM`、`INDUSTRY_RISK_PREMIUM`（ENV+API+UI）。
- 终值治理（TV）
  - 永续增长法：以“名义 GDP（通胀+实际）”为上限校验（min(PGR, GDP_cap)），超限给出来源与被更正值（已落地：ENV/请求控制）。
  - 退出乘数法：提供行业区间参考与偏离警示；同步展示隐含 PGR 与 EV/EBITDA（已落地：隐含 PGR 计算与提示）。
- 利润率与 COGS 双模式
  - 反推法（默认）：Revenue − EBIT − SGA − RD，保持损益恒等式，匹配目标利润率/费用率。
  - 直接比率法（可选）：COGS = Revenue × 历史 COGS/Revenue 中位数，适用于跟踪毛利趋势。
  - 已落地：`cogs_forecast_mode = residual|direct_ratio`（API+UI）。
- 营运资本（NWC）与 ΔNWC 基期
  - “纯历史中位数”场景：DIO/DPO 用损益口径的 COGS；“目标/过渡”场景用历史 COGS 比率，降低扭曲。
  - 统一并兼容 ΔNWC 基期键名：`last_historical_nwc/last_actual_nwc`。
  - 结果页固定展示首年 ΔNWC 构成（AR/Inv/AP/OCA/OCL）及基期。
  - LTM/Interim 基期（新增）：支持 `ltm_baseline_enabled`（默认关闭）以 LTM 口径确定 `last_actual_revenue`；估值日滚动最新资产负债表并重算 `last_component_nwc`（已落地）。
- 税率与 NOPAT
  - 有效税率从历史中位数过渡到目标，设合理上下限（如 5%–35%），越界告警。
- Capex 与 D&A 联动
  - 维持性投资约束（Capex ≥ D&A）+ 收入驱动的扩张项，长期趋于稳态（后续落地）。
- 股权桥口径
  - Net Debt = 有息负债 − 现金等价物；在桥接中扣除少数股东权益与优先股；UI 标注口径。

## 预测增强（中优先级）
- 收入路径：历史 CAGR ×（名义 GDP/行业基准）×（份额因子），并提供“行业预设”快捷项。
- 利润率路径：线性→逻辑/指数平滑可选；自动夹持在历史/行业上下限。
- 行业标尺库：β、预测期、PGR 上限、EV/EBITDA 参考、DSO/DIO/DPO 典型值。

## 透明度与可审计
- 关键假设摘要卡片（常驻）：
  - WACC 构成（Rf/β/MRP/CRP/IRP/权重/Ke/Kd(AT)）
  - TV 方法与参数、隐含 PGR 与 EV/EBITDA
  - 收入/利润率/COGS 模式与路径、首年 ΔNWC 构成、净债务口径
- 审计导出：请求载荷、映射后假设、历史中位数、逐年预测表、DCF 分解（JSON/CSV）。
- 基期调试：在 `debug_request_slice` 中注入 `baseline_debug`（Annual/LTM 模式、YTD/LTM 构成），使滚动重评可追溯。

## 敏感性与情景
- 二维敏感性保持；新增轻量蒙特卡洛（1000 次）：输出估值分布、置信区间、关键驱动排序。
- 三情景（悲观/基准/乐观）联动调整增长/利润率/WACC/PGR，并合并展示区间与驱动。

## 数据质量治理
- 指标覆盖率、异常值计数、时效性标识（可视化评分）；跨源一致性与基准校验。
- 口径一致性检查：损益恒等式、周转基底一致性、ΔNWC 基期有效性—触发提示与修正建议。

## 测试与基准
- 单元测试：
  - mid_year_convention、CRP/IRP 对 Ke/WACC 影响
  - `cogs_forecast_mode` 两模式边界与一致性
  - ΔNWC 基期键名与首年 ΔNWC 合理性
  - TV 上限校验（名义 GDP）与隐含指标
- 回归样例：建立锚定股票（不同行业）目标区间，防止模型漂移。

## Streamlit 体验
- 行业模板：一键加载“预测期/增长/PGR/WACC/周转/乘数区间”，并展示行业警示。
- 解释力：价值驱动分解图（增长、利润率、WACC、PGR、ΔNWC、Capex/D&A 贡献）。
- NWC 控件：新增“营运资本基准口径（component/aggregate）”与“天数过渡年数（默认3年）”。

## 实施路线图（2025-09-10）
- 里程碑 1（本月）
  - [x] Mid-Year Convention（已落地：API+UI）
  - [x] CRP/IRP（已落地：ENV+API+UI）
  - [x] COGS 双模式（已落地：API+UI）
  - [x] ΔNWC 基期统一与调试明细（已落地）
  - [x] LTM/Interim 基期与最新资产负债表滚动（已落地：API+Fetcher+UI）
- 里程碑 2（下月）
  - [x] TV 名义 GDP 上限校验与隐含 PGR 提示（已落地）
  - [ ] 关键假设摘要卡片常驻与审计导出（进行中：debug_* 已可用）
  - [x] 行业标尺库 v1（β/预测期/PGR/周转/乘数，GICS 11 + 白酒变体）（已落地）
  - [x] 股息口径回退链（TTM 实施/完成 → 预案 → 年度）与 UI 标签（已落地）
  - [x] as-of 一致性（价格≤估值日、EPS≤估值日）与 UI 标签（已落地）
- 里程碑 3（+1 月）
  - [ ] 收入多因子与利润平滑路径开关
  - [ ] 维持性/扩张性 Capex 与 D&A 联动
  - [ ] 蒙特卡洛情景与三情景联动

## 代码参考
- 预测引擎：`src/core/financial/forecaster.py`
- 现值：`src/core/calculators/dcf/present_value_calculator.py`
- 终值：`src/core/calculators/dcf/terminal_value_calculator.py`
- WACC：`src/core/calculators/dcf/wacc_calculator.py`
- 股权桥：`src/core/calculators/equity_bridge_calculator.py`
- Streamlit 参数：`frontend-streamlit/streamlit_app.py`

## 配置与字段（节选）
- ENV：`COUNTRY_RISK_PREMIUM`、`INDUSTRY_RISK_PREMIUM`、`RISK_FREE_RATE`、`MARKET_RISK_PREMIUM` …
- API（新增）：`mid_year_convention: bool`、`cogs_forecast_mode: residual|direct_ratio`、`nwc_baseline_mode: component|aggregate`、`nwc_days_transition_years: int`、`country_risk_premium`、`industry_risk_premium`、`ltm_baseline_enabled: bool`
- ENV（新增）：`DEFAULT_LTM_BASELINE_ENABLED=0`（默认关闭；前端可覆盖）

## 验收标准（关键）
- 开关默认关闭时估值回归一致；开启后影响方向与幅度可解释。
- `nwc_baseline_mode=component` 时，与 `aggregate` 相比，首年 ΔNWC 不出现非经济性跳变；当设置 `nwc_days_transition_years>0`，ΔNWC 在预测头两年平滑释放。
- 关键假设与输出在 UI 可视并可导出；调试面板包含“请求摘录/映射假设/历史中位数/首年构成”。
- 行业与宏观约束生效时，TV 与 Ke 更具合理性与稳定性。
- LTM/Interim：开启 `ltm_baseline_enabled` 时，`baseline_debug` 显示 YTD/LTM 分解；关闭时使用最新年报；两种基期下首年 ΔNWC 方向与幅度可解释、且与 NWC 基准口径一致。
