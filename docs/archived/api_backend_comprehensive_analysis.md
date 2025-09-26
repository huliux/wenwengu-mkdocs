# Stock Valuation API 后端接口详细分析文档
*最后更新: 2025-09-22*

## 📋 目录

- [API概览](#api概览)
- [接口详情](#接口详情)
  - [1. 根路径接口](#1-根路径接口)
  - [2. 股票估值接口](#2-股票估值接口)
  - [3. 敏感性分析接口](#3-敏感性分析接口)
  - [4. 股票筛选接口](#4-股票筛选接口)
- [数据模型](#数据模型)
- [错误处理](#错误处理)
- [部署和测试](#部署和测试)
- [使用示例](#使用示例)
- [最新更新记录](#最新更新记录)

---

## 🌐 API概览

### 基本信息
- **服务名称**: Stock Valuation API
- **版本**: 3.0.0
- **框架**: FastAPI
- **端口**: 8124
- **协议**: HTTP/HTTPS
- **数据格式**: JSON
- **数据源**: TuShare(主要) + PostgreSQL(验证)

### 核心功能
1. **DCF估值计算**: 基于折现现金流的企业估值分析，支持退出倍数法和永续增长法
2. **敏感性分析**: 多维度参数敏感性分析，生成sensitivity table
3. **股票筛选**: 基于财务指标的多条件股票筛选器
4. **LLM智能分析**: 集成DeepSeek/OpenAI兼容模型生成投资分析报告
5. **LTM基线**: 支持Last Twelve Months收入基期，提升时效性
3. **LLM智能分析**: 集成大语言模型的投资建议
4. **数据管理**: 股票基本信息和行情数据更新
5. **股票筛选**: 基于财务指标的股票筛选器

### 技术架构
```
FastAPI App
├── CORS Middleware (跨域支持)
├── Routes (路由层)
│   ├── Main Routes (主要业务接口)
│   ├── Data Update Routes (数据更新接口)
│   └── Stock Screener Routes (股票筛选接口)
├── Services (业务服务层)
│   ├── ValuationService (估值服务)
│   ├── DataProcessor (数据处理服务)
│   └── LLM Utils (LLM调用工具)
├── Core Calculators (核心计算器)
│   ├── WACC Calculator (资本成本计算)
│   ├── DCF Calculators (DCF计算器集群)
│   └── Financial Forecaster (财务预测器)
└── Data Layer (数据层)
    ├── AshareDataFetcher (数据获取器)
    └── PostgreSQL (数据库)
```

### 2025-09-10 变更要点（新增/调整字段与行为）
- 请求新增/强调
  - `industry_preset_key`：行业模板键（模板加载，不限编辑）
  - `use_gdp_cap` / `gdp_nominal_cap`：名义 GDP 上限控制（PGR clamp）
  - `ltm_baseline_enabled`：启用 LTM/Interim 经营基期
- 响应新增（StockBasicInfoModel）
  - `latest_price_as_of_date`：价格 as-of（trade_date ≤ 估值日）
  - `eps_annual_as_of_year`：EPS（年报）年份（end_date ≤ 估值日）
  - `ttm_dps_source`：TTM 股息来源（implemented/proposal/latest_annual）
  - `ttm_window_start`/`ttm_window_end`：TTM 窗口（若适用）
- 响应 Debug（ValuationResultsContainer.debug_request_slice）
  - `baseline_debug`：Annual/LTM 模式与 LTM 组成
  - `applied_preset` 与 `applied_preset_diff`：已应用行业模板与偏离情况（within/warn/alert）
- 终值与隐含指标
  - 永续增长：按 `min(pg_rate, GDP_cap)` 夹持（来源提示）
  - 退出乘数：计算隐含 PGR 并与 GDP 上限对比提示
- 数据层行为
  - 最新价格按估值日回退（`trade_date ≤ 估值日`）
  - 股息 TTM：实施/完成 → 无则 TTM·预案 → 仍无则 年度最近一次

### 2025-09-11 变更要点（敏感性与稳健性增强）
- 敏感性输出扩展
  - 新增 `result_tables.ev_ebitda_terminal`：基于“末期预测 EBITDA”的 EV/EBITDA（与 LTM 口径区分）。
  - 新增 `result_tables.implied_pgr`：当采用退出乘数法时，逐格计算隐含永续增长率 `g = (TV*WACC − FCF_T) / (TV + FCF_T)`。
  - `result_tables.dcf_implied_pe` 支持按格回退：若单元格未返回 `dcf_implied_diluted_pe`，将用基准 EPS（最近年报）和该格每股价值计算回填。
- 物理可行性约束
  - 当单元格采用永续增长法且 `g ≥ WACC` 时直接跳过计算，单元格置为 `null`，同时在 `data_warnings` 中追加提示。
- WACC 计算稳健性
  - 当 `wacc_weight_mode = market` 且市场权重路径失败时，自动回退至 `target` 权重并记录警告，不再导致 500。
- 文档澄清
  - 明确区分 `result_tables.ev_ebitda`（LTM 口径）与 `result_tables.ev_ebitda_terminal`（末期口径）；默认继续输出 LTM 以便对标现实口径，末期口径为补充表。
  - 轴再生策略：WACC 轴在提供 `step+points` 时总是围绕基准再生；其他轴仅在 `values` 空且提供 `step+points` 时再生（后续可统一）。

---

## 🔗 接口详情

### 1. 根路径接口

#### GET `/`
**接口描述**: API健康检查和基本信息

**请求参数**: 无

**响应格式**:
```json
{
    "message": "Welcome to the Stock Valuation API (Streamlit Backend)"
}
```

注（口径说明，2025-09-10 更新）：
- LTM/基期收入选择统一“优先营业收入 revenue，回退 total_revenue”。
- `baseline_debug` 中仍保留 `ytd_curr_total_revenue`、`ytd_prev_total_revenue`、`prev_annual_total_revenue` 等字段以便审计；但实际用于基期选择的优先级为 `revenue` 对应字段，其次才是 `total_revenue` 对应字段。

**状态码**:
- `200`: 成功

**使用场景**: 
- API健康检查
- 服务可用性验证

---

### 2. 股票估值接口

#### POST `/api/v1/valuation`
**接口描述**: 计算股票DCF估值，支持敏感性分析和LLM智能分析

**请求模型**: StockValuationRequest

**请求参数详解**:

##### 基础参数
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `ts_code` | string | 是 | 无 | 股票代码 | 符合A股格式(6数字.SH/SZ) | "600519.SH" |
| `market` | string | 否 | "A" | 市场标识 | 枚举值: "A", "HK" | "A" |
| `valuation_date` | string | 否 | 最新日期 | 估值基准日期 | YYYY-MM-DD格式 | "2024-12-31" |
| `ltm_baseline_enabled` | bool | 否 | false | 是否使用LTM/Interim基期 | 布尔值 | true, false |

##### DCF核心假设参数
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `forecast_years` | int | 否 | 5 | 预测期年数 | 1-20年整数 | 5, 10, 15 |
| `cagr_decay_rate` | float | 否 | 系统计算 | 历史CAGR年衰减率 | 0-1之间 | 0.1, 0.2 |
| `op_margin_forecast_mode` | string | 否 | "historical_median" | 营业利润率预测模式 | 枚举值: "historical_median", "transition_to_target" | "historical_median" |
| `target_operating_margin` | float | 否 | 无 | 目标营业利润率 | 0-1之间，transition模式必需 | 0.25, 0.30 |
| `op_margin_transition_years` | int | 否 | 无 | 利润率过渡年数 | >=1，transition模式必需 | 3, 5 |

##### WACC计算参数
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `wacc_weight_mode` | string | 否 | "target" | WACC权重计算模式 | 正则: `^(target\|market)$` | "target" |
| `target_debt_ratio` | float | 否 | 系统默认 | 目标债务比例 | 0-1之间，target模式必需 | 0.3, 0.4 |
| `cost_of_debt` | float | 否 | 系统默认 | 税前债务成本 | >=0 | 0.05, 0.08 |
| `risk_free_rate` | float | 否 | 系统默认 | 无风险利率 | >=0 | 0.03, 0.035 |
| `beta` | float | 否 | 系统默认 | 贝塔系数 | 任意实数 | 1.2, 0.8 |
| `market_risk_premium` | float | 否 | 系统默认 | 市场风险溢价 | >=0 | 0.05, 0.06 |
| `size_premium` | float | 否 | 系统默认 | 规模溢价 | 任意实数 | 0.02, -0.01 |

##### 终值计算参数
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `terminal_value_method` | string | 否 | "exit_multiple" | 终值计算方法 | 枚举值: "exit_multiple", "perpetual_growth" | "exit_multiple" |
| `exit_multiple` | float | 否 | 系统默认 | 退出乘数 | >=0，exit_multiple模式必需 | 15.0, 18.0 |
| `perpetual_growth_rate` | float | 否 | 系统默认 | 永续增长率 | >=0，perpetual_growth模式必需 | 0.03, 0.035 |

##### LLM控制参数
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `request_llm_summary` | bool | 否 | false | 是否请求LLM分析 | 布尔值 | true, false |
| `llm_provider` | string | 否 | 系统默认 | LLM提供商 | 字符串 | "deepseek", "openai" |
| `llm_model_id` | string | 否 | 系统默认 | LLM模型ID | 字符串 | "deepseek-chat", "gpt-4" |
| `llm_api_base_url` | string | 否 | 系统默认 | 自定义API地址 | URL格式 | "https://api.deepseek.com" |
| `llm_temperature` | float | 否 | 0.7 | 生成温度参数 | 0.0-2.0之间 | 0.5, 1.0, 1.5 |
| `llm_top_p` | float | 否 | 0.9 | Top-P参数 | 0.0-1.0之间 | 0.8, 0.95 |
| `llm_max_tokens` | int | 否 | 4000 | 最大Token数 | >=1 | 2000, 4096 |

##### 敏感性分析参数
| 参数名 | 类型 | 必填 | 说明 | 约束条件 |
|--------|------|------|------|----------|
| `sensitivity_analysis` | SensitivityAnalysisRequest | 否 | 敏感性分析配置 | 复杂对象结构 |

**详细请求体示例**:

##### 1. 基础估值请求
```json
{
    "ts_code": "600519.SH",
    "forecast_years": 5,
    "request_llm_summary": false,
    "ltm_baseline_enabled": false
}
```

##### 2. 完整参数请求（包含所有约束条件示例）
```json
{
    "ts_code": "600519.SH",
    "market": "A",
    "valuation_date": "2024-12-31",
    
    "forecast_years": 10,
    "cagr_decay_rate": 0.15,
    "op_margin_forecast_mode": "transition_to_target",
    "target_operating_margin": 0.30,
    "op_margin_transition_years": 5,
    
    "wacc_weight_mode": "target",
    "target_debt_ratio": 0.35,
    "cost_of_debt": 0.045,
    "risk_free_rate": 0.03,
    "beta": 1.1,
    "market_risk_premium": 0.055,
    "size_premium": 0.015,
    
    "terminal_value_method": "exit_multiple",
    "exit_multiple": 16.5,
    "ltm_baseline_enabled": true,
    
    "request_llm_summary": true,
    "llm_provider": "deepseek",
    "llm_model_id": "deepseek-chat",
    "llm_api_base_url": "https://api.deepseek.com",
    "llm_temperature": 0.8,
    "llm_top_p": 0.9,
    "llm_max_tokens": 3000
}
```

##### 3. 边界值测试请求
```json
{
    "ts_code": "600519.SH",
    "forecast_years": 20,
    "cagr_decay_rate": 0.99,
    "target_operating_margin": 0.99,
    "op_margin_transition_years": 1,
    "target_debt_ratio": 0.99,
    "cost_of_debt": 0.20,
    "risk_free_rate": 0.15,
    "llm_temperature": 2.0,
    "llm_top_p": 1.0,
    "llm_max_tokens": 1
}
```

##### 4. 敏感性分析请求
```json
{
    "ts_code": "600519.SH",
    "forecast_years": 5,
    "sensitivity_analysis": {
        "row_axis": {
            "parameter_name": "wacc",
            "values": [0.08, 0.09, 0.10, 0.11],
            "step": 0.01,
            "points": 4
        },
        "column_axis": {
            "parameter_name": "exit_multiple",
            "values": [14, 16, 18, 20],
            "step": 2,
            "points": 4
        }
    }
}
```

##### 5. 永续增长法请求
```json
{
    "ts_code": "600519.SH",
    "forecast_years": 8,
    "terminal_value_method": "perpetual_growth",
    "perpetual_growth_rate": 0.035,
    "wacc_weight_mode": "market",
    "ltm_baseline_enabled": true,
    "request_llm_summary": true
}
```

##### 6. 错误请求示例（违反约束条件）
```json
{
    "ts_code": "600519.SH",
    "forecast_years": 25,  // 错误：超过最大值20
    "cagr_decay_rate": 1.5,  // 错误：超过最大值1
    "target_operating_margin": -0.1,  // 错误：负值
    "llm_temperature": 3.0,  // 错误：超过最大值2.0
    "llm_top_p": 1.5,  // 错误：超过最大值1.0
    "wacc_weight_mode": "invalid_mode"  // 错误：不在枚举值中
}
```

**响应模型**: StockValuationResponse

**响应格式详解**:

##### 1. 成功响应 - 基础估值
```json
{
    "stock_info": {
        "ts_code": "600519.SH",
        "name": "贵州茅台",
        "industry": "白酒",
        "list_date": "2001-08-27",
        "exchange": "上交所",
        "currency": "CNY",
        "market": "A",
        "latest_pe_ttm": 45.2,
        "latest_pb_mrq": 12.8,
        "total_shares": 1256197800,
        "free_float_shares": 1256197800,
        "ttm_dps": 25.91,
        "dividend_yield": 0.0158,
        "act_name": "贵州省人民政府国有资产监督管理委员会",
        "act_ent_type": "国企",
        "latest_annual_diluted_eps": 49.32,
        "base_report_date": "2024-09-30"
    },
    "valuation_results": {
        "latest_price": 1725.0,
        "current_pe": 45.2,
        "current_pb": 12.8,
        "dcf_forecast_details": {
            "enterprise_value": 2150000000000,
            "equity_value": 2100000000000,
            "value_per_share": 1675.5,
            "net_debt": 50000000000,
            "pv_forecast_ufcf": 800000000000,
            "pv_terminal_value": 1350000000000,
            "terminal_value": 2500000000000,
            "wacc_used": 0.085,
            "cost_of_equity_used": 0.12,
            "terminal_value_method_used": "exit_multiple",
            "exit_multiple_used": 15.2,
            "perpetual_growth_rate_used": null,
            "forecast_period_years": 5,
            "dcf_implied_diluted_pe": 33.97,
            "base_ev_ebitda": 18.5,
            "implied_perpetual_growth_rate": 0.035
        },
        "other_analysis": {
            "dividend_analysis": {
                "current_yield_pct": 1.58,
                "avg_dividend_3y": 22.5,
                "payout_ratio_pct": 52.5
            },
            "growth_analysis": {
                "net_income_cagr_3y": 15.2,
                "revenue_cagr_3y": 18.7
            }
        },
        "detailed_forecast_table": [
            {
                "year": 2024,
                "revenue": 150000000000,
                "revenue_growth": 0.18,
                "gross_profit": 120000000000,
                "operating_profit": 75000000000,
                "operating_margin": 0.50,
                "ebit": 75000000000,
                "tax_rate": 0.25,
                "nopat": 56250000000,
                "da": 8000000000,
                "capex": 12000000000,
                "nwc_change": 5000000000,
                "ufcf": 47250000000,
                "nwc": 30000000000,
                "nwc_days": 73
            },
            {
                "year": 2025,
                "revenue": 168000000000,
                "revenue_growth": 0.12,
                "gross_profit": 134400000000,
                "operating_profit": 84000000000,
                "operating_margin": 0.50,
                "ebit": 84000000000,
                "tax_rate": 0.25,
                "nopat": 63000000000,
                "da": 8960000000,
                "capex": 13440000000,
                "nwc_change": 5600000000,
                "ufcf": 52960000000,
                "nwc": 35600000000,
                "nwc_days": 73
            }
        ],
        "sensitivity_analysis_result": null,
        "llm_analysis_summary": null,
        "data_warnings": [
            "2024年数据为基于前三个季度的预测值",
            "部分历史数据缺失，使用行业平均值替代"
        ],
        "historical_financial_summary": [
            {
                "year": 2023,
                "revenue": 129700000000,
                "operating_profit": 64850000000,
                "net_income": 48550000000,
                "total_assets": 185000000000,
                "total_liabilities": 45000000000
            }
        ],
        "historical_ratios_summary": [
            {
                "year": 2023,
                "operating_margin": 0.50,
                "roic": 0.28,
                "asset_turnover": 0.70,
                "debt_to_equity": 0.32
            }
        ],
        "special_industry_warning": null
    },
    "error": null
}
```
##### 调试字段说明（片段）
响应中还包含用于审计/调试的片段：
```json
{
  "valuation_results": {
    "debug_request_slice": {
      "forecast_years": 5,
      "ltm_baseline_enabled": true,
      "nwc_baseline_mode": "component",
      "baseline_debug": { "mode": "LTM", "curr_end_date": "2025-06-30", "curr_end_type": "2" }
    }
  }
}
```
其中 `baseline_debug` 会在 LTM 模式下包含 YTD/LTM 的分解；在 Annual 模式为 `{ "mode": "Annual" }`。

###### 对照说明（最小节选 vs 完整版）
- 最小节选：仅返回前端最常用的快取字段（`value_per_share`、`wacc_used`、`exit_multiple_used`、价格与TTM分红口径等），`debug_request_slice` 仅含 `baseline_debug`。
- 完整版：在最小节选基础上，补充 `detailed_forecast_table`、`historical_*`、LLM摘要、以及 `debug_request_slice.applied_preset` 与 `applied_preset_diff`（含区间与偏离状态）。
- 该对照用于 UI 快速加载与审计深挖两种路径，接口一致，字段可选返回。

###### 1.1 成功响应 - 最小节选（便于前端快速取用）
```json
{
  "stock_info": {
    "ts_code": "600519.SH",
    "name": "贵州茅台",
    "market": "主板",
    "latest_annual_diluted_eps": 68.64,
    "base_report_date": "2023-12-31",
    "latest_price_as_of_date": "2024-10-15",
    "eps_annual_as_of_year": 2023,
    "ttm_dps": 49.982,
    "dividend_yield": 0.0349,
    "ttm_dps_source": "implemented"
  },
  "valuation_results": {
    "latest_price": 1476.1,
    "current_pe": 28.08,
    "current_pb": 9.73,
    "dcf_forecast_details": {
      "value_per_share": 2318.05,
      "wacc_used": 0.0609,
      "terminal_value_method_used": "exit_multiple",
      "exit_multiple_used": 15.0,
      "implied_perpetual_growth_rate": 0.0169
    },
    "debug_request_slice": {
      "baseline_debug": { "mode": "LTM", "curr_end_date": "2024-09-30" }
    }
  }
}
```

###### 1.2 成功响应 - 完整版（含 debug/applied_preset/diff）
```json
{
  "stock_info": {
    "ts_code": "601717.SH",
    "name": "郑煤机",
    "market": "主板",
    "latest_annual_diluted_eps": 2.2120,
    "base_report_date": "2023-12-31",
    "latest_price_as_of_date": "2025-09-10",
    "eps_annual_as_of_year": 2024,
    "ttm_dps": 1.12,
    "dividend_yield": 0.0516,
    "ttm_dps_source": "proposal",
    "ttm_window_start": "2024-09-10",
    "ttm_window_end": "2025-09-10"
  },
  "valuation_results": {
    "latest_price": 21.71,
    "current_pe": 9.23,
    "current_pb": 1.48,
    "dcf_forecast_details": {
      "value_per_share": 26.86,
      "wacc_used": 0.0609,
      "terminal_value_method_used": "exit_multiple",
      "exit_multiple_used": 8.0,
      "implied_perpetual_growth_rate": 0.0045
    },
    "detailed_forecast_table": [ { "year": 1, "revenue": 3.70e10, "ufcf": 1.23e9 } ],
    "data_warnings": [],
    "debug_request_slice": {
      "baseline_debug": {
        "mode": "LTM",
        "curr_end_date": "2025-06-30",
        "curr_end_type": "2",
        "ytd_curr_total_revenue": 2.786e10,
        "ytd_prev_total_revenue": 2.726e10,
        "prev_annual_total_revenue": 3.642e10
      },
      "applied_preset": {
        "key": "industrials",
        "beta": { "default": 1.0, "range": [0.8, 1.3] },
        "forecast_years": { "default": 5, "range": [4, 7] },
        "pgr_cap": { "default": 0.045, "range": [0.03, 0.05] },
        "exit_multiple_ev_ebitda": { "default": 9.0, "range": [6.0, 12.0] },
        "turnover_days": {
          "ar": { "median": 60, "range": [20, 120] },
          "inv": { "median": 80, "range": [40, 180] },
          "ap": { "median": 55, "range": [25, 110] }
        }
      },
      "applied_preset_diff": {
        "beta": { "value": 1.0, "default": 1.0, "range_status": "within", "modified": false },
        "forecast_years": { "value": 5.0, "default": 5.0, "range_status": "within", "modified": false },
        "pgr_cap": { "value": 0.045, "default": 0.045, "range_status": "within", "modified": false },
        "exit_multiple_ev_ebitda": { "value": 9.0, "default": 9.0, "range_status": "within", "modified": false },
        "turnover_days": {
          "ar": { "value": 60.0, "default": 60.0, "range_status": "within", "modified": false },
          "inv": { "value": 80.0, "default": 80.0, "range_status": "within", "modified": false },
          "ap": { "value": 55.0, "default": 55.0, "range_status": "within", "modified": false }
        }
      }
    }
  },
  "error": null
}
```

##### 1.x 三情景分析（乐观/基准/悲观）（提案）
为补足“区间输出、关键驱动排序”的需求，建议在现有估值接口中引入可选字段 `scenarios`，由客户端提交三套参数调整，服务端按三套假设各自跑一遍估值，并给出汇总区间与驱动排序。

请求模型扩展（新增，可选）：
```json
{
  "ts_code": "600519.SH",
  "forecast_years": 5,
  "terminal_value_method": "exit_multiple",
  "exit_multiple": 15,
  "scenarios": {
    "base": {
      "label": "基准",
      "weight": 0.5,
      "overrides": {},
      "multipliers": {}
    },
    "bull": {
      "label": "乐观",
      "weight": 0.25,
      "overrides": { "wacc": 0.07, "exit_multiple": 17 },
      "multipliers": { "target_operating_margin": 1.05, "target_inventory_days": 0.9 }
    },
    "bear": {
      "label": "悲观",
      "weight": 0.25,
      "overrides": { "wacc": 0.10, "exit_multiple": 13 },
      "multipliers": { "target_operating_margin": 0.95, "target_inventory_days": 1.1 }
    }
  }
}
```

字段约定：
- `overrides`：按请求模型同名字段进行替换（如 `wacc`、`exit_multiple`、`perpetual_growth_rate`、`target_operating_margin`、`target_*_days` 等）。
- `multipliers`：按请求模型同名字段进行乘数调整（先乘后截断到有效区间，优先级低于 `overrides`）。
- `weight`：用于加权均值/分位的权重（可选，默认各 1/3）。

响应扩展（新增，可选）：
```json
{
  "valuation_results": {
    "scenario_results": {
      "base": {
        "dcf_forecast_details": { "value_per_share": 1675.5, "wacc_used": 0.085 },
        "ev_ebitda": 18.5
      },
      "bull": {
        "dcf_forecast_details": { "value_per_share": 1850.2, "wacc_used": 0.078 },
        "ev_ebitda": 19.8
      },
      "bear": {
        "dcf_forecast_details": { "value_per_share": 1498.3, "wacc_used": 0.094 },
        "ev_ebitda": 17.2
      },
      "summary": {
        "value_per_share": { "min": 1498.3, "p10": 1510.0, "p50": 1675.5, "p90": 1830.0, "max": 1850.2, "weighted_mean": 1674.9 },
        "enterprise_value": { "min": 1.76e12, "p50": 2.10e12, "max": 2.32e12 },
        "key_drivers_ranking": [
          { "name": "wacc", "contribution": 0.46 },
          { "name": "exit_multiple", "contribution": 0.31 },
          { "name": "target_operating_margin", "contribution": 0.15 },
          { "name": "nwc_days", "contribution": 0.08 }
        ],
        "note": "排名基于基准点周边一因子弹性/方差贡献的近似分解（提案）。"
      }
    }
  }
}
```

实现备注：
- 当前代码尚未返回 `scenario_results` 字段，此为接口提案；落地需在 `StockValuationRequest` 增加 `scenarios`，并在 `ValuationService` 逐情景重跑并汇总。
- `key_drivers_ranking` 可基于一因子扰动（OFAT）或 Sobol 近似（轻量）计算相对贡献，先实现 OFAT 版以保证性能与可解释性。
- UI 建议：
  - 情景面板：为“收入增速/利润率/NWC/WACC/终值参数”提供“覆盖/乘数”两种输入，支持权重；提供“重置为行业预设”的按钮。
  - 结果汇总：展示估值区间（条形/胡须图）、权重均值、与基准差异；绘制“关键驱动排序”（Tornado chart）。

##### 2. 成功响应 - 包含敏感性分析
```json
{
    "stock_info": {
        "ts_code": "600519.SH",
        "name": "贵州茅台",
        "industry": "白酒",
        "market": "A",
        "latest_pe_ttm": 45.2,
        "latest_pb_mrq": 12.8,
        "total_shares": 1256197800,
        "base_report_date": "2024-09-30"
    },
    "valuation_results": {
        "latest_price": 1725.0,
        "current_pe": 45.2,
        "current_pb": 12.8,
        "dcf_forecast_details": {
            "enterprise_value": 2150000000000,
            "equity_value": 2100000000000,
            "value_per_share": 1675.5,
            "wacc_used": 0.085,
            "terminal_value_method_used": "exit_multiple",
            "exit_multiple_used": 15.2,
            "forecast_period_years": 5,
            "dcf_implied_diluted_pe": 33.97
        },
        "sensitivity_analysis_result": {
            "row_parameter": "wacc",
            "column_parameter": "exit_multiple",
            "row_values": [0.08, 0.09, 0.10, 0.11],
            "column_values": [14, 16, 18, 20],
            "result_tables": {
                "value_per_share": [
                    [1850.5, 1675.5, 1520.3, 1380.2],
                    [1680.3, 1520.8, 1380.5, 1255.2],
                    [1535.2, 1388.6, 1260.4, 1148.3],
                    [1408.1, 1274.2, 1157.8, 1054.6]
                ],
                "enterprise_value": [
                    [2325000000000, 2100000000000, 1900000000000, 1720000000000],
                    [2110000000000, 1910000000000, 1730000000000, 1570000000000],
                    [1925000000000, 1740000000000, 1580000000000, 1430000000000],
                    [1760000000000, 1590000000000, 1440000000000, 1310000000000]
                ],
                "dcf_implied_pe": [
                    [37.5, 33.97, 30.8, 27.9],
                    [34.0, 30.8, 27.9, 25.3],
                    [31.1, 28.1, 25.5, 23.2],
                    [28.5, 25.8, 23.4, 21.3]
                ],
                "tv_ev_ratio": [
                    [0.68, 0.65, 0.62, 0.60],
                    [0.70, 0.66, 0.63, 0.61],
                    [0.71, 0.67, 0.64, 0.62],
                    [0.72, 0.68, 0.65, 0.63]
                ],
                "ev_ebitda": [
                    [18.9, 17.8, 16.7, 15.9],
                    [18.3, 17.2, 16.3, 15.5],
                    [17.7, 16.7, 15.8, 15.1],
                    [17.2, 16.2, 15.4, 14.8]
                ],
                "ev_ebitda_terminal": [
                    [16.2, 15.1, 14.3, 13.7],
                    [15.7, 14.7, 13.9, 13.3],
                    [15.3, 14.3, 13.6, 13.0],
                    [14.9, 14.0, 13.3, 12.8]
                ],
                "implied_pgr": [
                    [0.017, 0.013, 0.011, 0.009],
                    [0.019, 0.015, 0.012, 0.010],
                    [0.021, 0.017, 0.014, 0.011],
                    [0.022, 0.018, 0.015, 0.012]
                ]
            }
        },
        "data_warnings": [],
        "historical_financial_summary": [...],
        "historical_ratios_summary": [...]
    },
    "error": null
}
```

说明：
- 若列参数为 `perpetual_growth_rate` 且出现 `g ≥ WACC` 的行/列组合，该单元格返回 `null`；同时 `valuation_results.data_warnings` 会包含“组合被跳过”的提示，便于 UI 标注空白的合理性。
- `dcf_implied_pe` 的单元格数值优先取模型返回；缺失时以基准 EPS（最近年报）与该格每股价值计算回填。
- `ev_ebitda` 以 LTM 实际 EBITDA 为分母；`ev_ebitda_terminal` 以末期预测 EBITDA 为分母。两者口径不同，使用场景不同（LTM 便于与市场口径对标，Terminal 便于校验退出倍数假设）。

##### 3. 成功响应 - 包含LLM分析
```json
{
    "stock_info": {
        "ts_code": "600519.SH",
        "name": "贵州茅台",
        "industry": "白酒",
        "market": "A",
        "latest_pe_ttm": 45.2,
        "base_report_date": "2024-09-30"
    },
    "valuation_results": {
        "latest_price": 1725.0,
        "current_pe": 45.2,
        "dcf_forecast_details": {
            "value_per_share": 1675.5,
            "wacc_used": 0.085,
            "terminal_value_method_used": "exit_multiple",
            "exit_multiple_used": 15.2,
            "dcf_implied_diluted_pe": 33.97
        },
        "llm_analysis_summary": "# 贵州茅台(600519.SH)投资分析报告\n\n## 估值摘要\n- **当前股价**: 1,725.0元\n- **DCF估值**: 1,675.5元\n- **估值差异**: -2.9% (略微低估)\n- **建议评级**: 持有\n\n## 投资亮点\n1. **品牌护城河**: 茅台拥有强大的品牌价值和定价权\n2. **财务表现**: 毛利率高达50%，ROE超过25%\n3. **增长稳定**: 收入和利润保持稳健增长\n4. **现金流**: 产生强劲的自由现金流\n\n## 风险因素\n1. **估值偏高**: 当前PE_TTM为45.2倍，高于历史平均\n2. **政策风险**: 白酒行业面临消费政策变化风险\n3. **竞争加剧**: 高端白酒市场竞争加剧\n\n## 投资建议\n基于DCF估值结果，当前股价略微高估，建议投资者等待更好的买入时机。长期来看，茅台依然是优质的核心资产。\n\n> *分析基于DCF估值模型，存在模型假设风险，投资需谨慎。*",
        "data_warnings": [
            "LLM分析基于历史数据，未来表现可能存在差异"
        ]
    },
    "error": null
}
```

##### 4. 错误响应 - 参数验证失败
```json
{
    "detail": [
        {
            "loc": ["body", "forecast_years"],
            "msg": "ensure this value is less than or equal to 20",
            "type": "value_error.number.not_le",
            "ctx": {
                "limit_value": 20
            }
        },
        {
            "loc": ["body", "cagr_decay_rate"],
            "msg": "ensure this value is less than or equal to 1",
            "type": "value_error.number.not_le",
            "ctx": {
                "limit_value": 1
            }
        }
    ]
}
```

##### 5. 错误响应 - 股票数据未找到
```json
{
    "detail": "无法获取股票基本信息: 999999.SH"
}
```

##### 6. 错误响应 - 业务逻辑错误
```json
{
    "detail": "预测期年数必须在1-20之间"
}
```

##### 7. 错误响应 - 服务器内部错误
```json
{
    "detail": "服务器内部错误: 计算WACC失败"
}
```

##### 8. 成功响应 - 金融行业特殊警告
```json
{
    "stock_info": {
        "ts_code": "600036.SH",
        "name": "招商银行",
        "industry": "银行",
        "market": "A",
        "latest_pe_ttm": 8.5,
        "base_report_date": "2024-09-30"
    },
    "valuation_results": {
        "latest_price": 35.6,
        "current_pe": 8.5,
        "dcf_forecast_details": {
            "value_per_share": 38.2,
            "wacc_used": 0.075,
            "terminal_value_method_used": "perpetual_growth",
            "perpetual_growth_rate_used": 0.03
        },
        "special_industry_warning": "⚠️ 金融行业警告：DCF模型对金融企业适用性有限。建议使用以下方法辅助估值：\n1. 股息折现模型(DDM)\n2. 市净率(P/B)相对估值\n3. 净资产收益率(ROE)分析\n4. 不良贷款率和拨备覆盖率分析\n\n当前DCF估值结果仅供参考，投资决策请结合专业金融分析。"
    },
    "error": null
}
```

##### 9. 成功响应 - 批量处理（无LLM）
```json
{
    "stock_info": {
        "ts_code": "600519.SH",
        "name": "贵州茅台",
        "industry": "白酒",
        "market": "A",
        "latest_pe_ttm": 45.2,
        "total_shares": 1256197800,
        "base_report_date": "2024-09-30"
    },
    "valuation_results": {
        "latest_price": 1725.0,
        "current_pe": 45.2,
        "dcf_forecast_details": {
            "enterprise_value": 2150000000000,
            "equity_value": 2100000000000,
            "value_per_share": 1675.5,
            "wacc_used": 0.085,
            "terminal_value_method_used": "exit_multiple",
            "exit_multiple_used": 15.2,
            "forecast_period_years": 5,
            "dcf_implied_diluted_pe": 33.97
        },
        "llm_analysis_summary": null,
        "data_warnings": ["批量处理模式，已关闭LLM分析以提高性能"],
        "detailed_forecast_table": [...],
        "historical_financial_summary": [...]
    },
    "error": null
}
```

**业务逻辑流程**:
1. **数据获取阶段**: 获取股票基本信息、价格、财务报表数据
2. **数据处理阶段**: 清洗数据、计算财务指标、生成历史比率
3. **估值计算阶段**: 
   - 初始化WACC计算器
   - 运行财务预测
   - 计算DCF估值
   - 计算终值
4. **敏感性分析阶段**: 多维度参数敏感性测试
5. **LLM分析阶段**: 调用大语言模型生成投资建议
6. **结果整合阶段**: 整合所有计算结果和警告信息

**状态码**:
- `200`: 成功
- `400`: 请求参数错误
- `404`: 股票数据未找到
- `500`: 服务器内部错误

---

### 3. 数据更新接口

#### POST `/update_stock_basic`
**接口描述**: 更新股票基本信息数据

**请求参数**: 无

**请求体示例**:
```json
{}
```

**详细响应体示例**:

##### 1. 成功响应 - 正常更新
```json
{
    "status": "success",
    "message": "股票基本信息更新成功，共5,234条记录",
    "count": 5234,
    "details": {
        "updated_count": 5234,
        "new_count": 156,
        "execution_time": 45.2,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:30:00Z"
    }
}
```

##### 2. 成功响应 - 无新数据
```json
{
    "status": "success",
    "message": "股票基本信息已是最新，无需更新",
    "count": 0,
    "details": {
        "updated_count": 0,
        "new_count": 0,
        "execution_time": 5.1,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:30:00Z"
    }
}
```

##### 3. 成功响应 - 部分更新
```json
{
    "status": "partial_success",
    "message": "股票基本信息部分更新成功，共更新3,200条记录，1,500条记录失败",
    "count": 3200,
    "details": {
        "updated_count": 3200,
        "failed_count": 1500,
        "new_count": 89,
        "execution_time": 38.7,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:30:00Z",
        "errors": [
            "API调用频率限制",
            "部分股票代码格式错误"
        ]
    }
}
```

##### 4. 错误响应 - API调用失败
```json
{
    "status": "error",
    "message": "更新失败：TuShare API调用失败",
    "error": "API connection timeout",
    "details": {
        "error_code": "API_TIMEOUT",
        "error_message": "Connection to TuShare API timed out after 30 seconds",
        "execution_time": 30.0,
        "failed_count": 0
    }
}
```

##### 5. 错误响应 - 数据库错误
```json
{
    "status": "error",
    "message": "更新失败：数据库连接错误",
    "error": "Database connection failed",
    "details": {
        "error_code": "DB_CONNECTION_ERROR",
        "error_message": "Unable to connect to PostgreSQL database",
        "execution_time": 0.5,
        "failed_count": 0
    }
}
```

**业务逻辑**:
1. 调用数据处理器更新股票基本信息
2. 强制从数据源获取最新数据
3. 更新数据库记录
4. 返回更新结果统计

**状态码**:
- `200`: 更新成功
- `500`: 更新失败

---

#### POST `/update_daily_basic`
**接口描述**: 更新每日行情指标数据

**请求参数**: 无

**请求体示例**:
```json
{}
```

**详细响应体示例**:

##### 1. 成功响应 - 正常更新
```json
{
    "status": "success",
    "message": "每日行情指标更新成功，共4,856条记录",
    "count": 4856,
    "details": {
        "updated_count": 4856,
        "new_count": 45,
        "execution_time": 32.8,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:35:00Z",
        "trade_date": "2024-12-30"
    }
}
```

##### 2. 成功响应 - 市场休市
```json
{
    "status": "success",
    "message": "当前日期为周末或节假日，无需更新行情数据",
    "count": 0,
    "details": {
        "updated_count": 0,
        "new_count": 0,
        "execution_time": 2.1,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:35:00Z",
        "trade_date": "2024-12-29",
        "note": "周末休市"
    }
}
```

##### 3. 成功响应 - 部分更新
```json
{
    "status": "partial_success",
    "message": "每日行情指标部分更新成功，共更新4,000条记录，500条记录失败",
    "count": 4000,
    "details": {
        "updated_count": 4000,
        "failed_count": 500,
        "new_count": 23,
        "execution_time": 28.5,
        "data_source": "tushare",
        "update_time": "2024-12-31T10:35:00Z",
        "trade_date": "2024-12-30",
        "errors": [
            "部分股票停牌",
            "API调用频率限制"
        ]
    }
}
```

##### 4. 错误响应 - 网络错误
```json
{
    "status": "error",
    "message": "更新失败：网络连接错误",
    "error": "Network connection failed",
    "details": {
        "error_code": "NETWORK_ERROR",
        "error_message": "Unable to connect to TuShare API server",
        "execution_time": 10.0,
        "failed_count": 0
    }
}
```

##### 5. 错误响应 - 数据格式错误
```json
{
    "status": "error",
    "message": "更新失败：数据格式错误",
    "error": "Data format error",
    "details": {
        "error_code": "DATA_FORMAT_ERROR",
        "error_message": "Received data format does not match expected schema",
        "execution_time": 15.2,
        "failed_count": 0,
        "sample_error": "Field 'pe_ttm' is missing in some records"
    }
}
```

**业务逻辑**:
1. 调用数据处理器更新每日行情数据
2. 获取最新PE、PB、市值等指标
3. 更新数据库记录
4. 返回更新结果统计

**状态码**:
- `200`: 更新成功
- `500`: 更新失败

---

### 4. 股票筛选接口

#### POST `/screen_stocks`
**接口描述**: 根据PE、PB、市值等条件筛选股票

**请求参数详解**:
| 参数名 | 类型 | 必填 | 默认值 | 说明 | 约束条件 | 示例 |
|--------|------|------|--------|------|----------|------|
| `pe_min` | float | 否 | 无 | 最小市盈率 | >=0 | 10, 15 |
| `pe_max` | float | 否 | 无 | 最大市盈率 | >=pe_min | 30, 50 |
| `pb_min` | float | 否 | 无 | 最小市净率 | >=0 | 1, 2 |
| `pb_max` | float | 否 | 无 | 最大市净率 | >=pb_min | 5, 8 |
| `market_cap_min` | float | 否 | 无 | 最小市值(亿元) | >=0 | 100, 200 |
| `market_cap_max` | float | 否 | 无 | 最大市值(亿元) | >=market_cap_min | 1000, 2000 |

**详细请求体示例**:

##### 1. 基础筛选请求
```json
{
    "pe_min": 10,
    "pe_max": 30,
    "market_cap_min": 100
}
```

##### 2. 完整筛选条件
```json
{
    "pe_min": 5,
    "pe_max": 25,
    "pb_min": 0.5,
    "pb_max": 3,
    "market_cap_min": 50,
    "market_cap_max": 500
}
```

##### 3. 单一条件筛选
```json
{
    "pe_max": 20
}
```

##### 4. 大盘股筛选
```json
{
    "market_cap_min": 1000,
    "pe_max": 25,
    "pb_max": 5
}
```

##### 5. 小盘价值股筛选
```json
{
    "market_cap_max": 200,
    "pe_max": 15,
    "pb_max": 2
}
```

##### 6. 边界值测试请求
```json
{
    "pe_min": 0,
    "pe_max": 9999,
    "pb_min": 0,
    "pb_max": 9999,
    "market_cap_min": 0,
    "market_cap_max": 999999
}
```

##### 7. 错误请求示例
```json
{
    "pe_min": 30,  // 错误：pe_min > pe_max
    "pe_max": 10,
    "pb_min": -5,  // 错误：负值
    "market_cap_min": 5000,  // 错误：market_cap_min > market_cap_max
    "market_cap_max": 1000
}
```

**详细响应体示例**:

##### 1. 成功响应 - 基础筛选
```json
{
    "status": "success",
    "message": "筛选完成，共找到150只符合条件的股票",
    "count": 150,
    "data": [
        {
            "ts_code": "600519.SH",
            "name": "贵州茅台",
            "industry": "白酒",
            "area": "贵州",
            "market": "主板",
            "list_date": "2001-08-27",
            "close": 1725.0,
            "pe": 45.2,
            "pb": 12.8,
            "total_mv": 21500.0,
            "float_mv": 21500.0,
            "turnover_rate": 0.85,
            "volume_ratio": 1.2,
            "price_change": 2.5
        },
        {
            "ts_code": "000858.SZ",
            "name": "五粮液",
            "industry": "白酒",
            "area": "四川",
            "market": "主板",
            "list_date": "1998-04-27",
            "close": 125.8,
            "pe": 28.5,
            "pb": 8.2,
            "total_mv": 5150.0,
            "float_mv": 5150.0,
            "turnover_rate": 1.2,
            "volume_ratio": 1.5,
            "price_change": 1.8
        }
    ]
}
```

##### 2. 成功响应 - 无结果
```json
{
    "status": "success",
    "message": "筛选完成，未找到符合条件的股票",
    "count": 0,
    "data": []
}
```

##### 3. 成功响应 - 单一结果
```json
{
    "status": "success",
    "message": "筛选完成，共找到1只符合条件的股票",
    "count": 1,
    "data": [
        {
            "ts_code": "600519.SH",
            "name": "贵州茅台",
            "industry": "白酒",
            "area": "贵州",
            "market": "主板",
            "list_date": "2001-08-27",
            "close": 1725.0,
            "pe": 45.2,
            "pb": 12.8,
            "total_mv": 21500.0,
            "float_mv": 21500.0,
            "turnover_rate": 0.85,
            "volume_ratio": 1.2,
            "price_change": 2.5
        }
    ]
}
```

##### 4. 错误响应 - 参数验证失败
```json
{
    "detail": [
        {
            "loc": ["body", "pe_min"],
            "msg": "ensure this value is greater than or equal to 0",
            "type": "value_error.number.not_ge",
            "ctx": {
                "limit_value": 0
            }
        },
        {
            "loc": ["body", "market_cap_min"],
            "msg": "ensure this value is less than or equal to market_cap_max",
            "type": "value_error.number.not_le",
            "ctx": {
                "limit_value": 1000
            }
        }
    ]
}
```

##### 5. 错误响应 - 数据未找到
```json
{
    "detail": "无法获取股票数据，请先更新股票基本信息"
}
```

**业务逻辑**:
1. 获取合并后的股票数据
2. 应用筛选条件（PE、PB、市值范围）
3. 移除无效数据
4. 按市值降序排列
5. 返回筛选结果

**状态码**:
- `200`: 筛选成功
- `404`: 数据未找到
- `500`: 筛选失败

---

## 📋 参数约束条件总结表

### 股票估值接口参数约束

#### 基础参数约束
| 参数名 | 类型 | 约束条件 | 说明 | 有效示例 | 无效示例 |
|--------|------|----------|------|----------|----------|
| `ts_code` | string | 符合A股格式 | 6位数字+.SH/.SZ | "600519.SH" | "600519", "999999.SH" |
| `market` | string | 枚举值 | "A", "HK" | "A" | "US", "NASDAQ" |
| `valuation_date` | string | YYYY-MM-DD格式 | 有效的日期格式 | "2024-12-31" | "2024/12/31", "invalid" |

#### 数值范围约束参数
| 参数名 | 类型 | 最小值 | 最大值 | 说明 | 有效示例 | 无效示例 |
|--------|------|--------|--------|------|----------|----------|
| `forecast_years` | int | 1 | 20 | 预测期年数 | 5, 10, 20 | 0, 25, -1 |
| `cagr_decay_rate` | float | 0 | 1 | CAGR衰减率 | 0.1, 0.5, 0.99 | -0.1, 1.5 |
| `target_operating_margin` | float | 0 | 1 | 目标营业利润率 | 0.25, 0.5 | -0.1, 1.5 |
| `op_margin_transition_years` | int | 1 | 无限制 | 过渡年数 | 3, 5, 10 | 0, -1 |
| `target_debt_ratio` | float | 0 | 1 | 目标债务比例 | 0.3, 0.5 | -0.1, 1.5 |
| `llm_temperature` | float | 0.0 | 2.0 | LLM温度参数 | 0.0, 1.0, 2.0 | -0.1, 2.1 |
| `llm_top_p` | float | 0.0 | 1.0 | LLM Top-P参数 | 0.0, 0.5, 1.0 | -0.1, 1.1 |

#### 非负约束参数
| 参数名 | 类型 | 约束条件 | 说明 | 有效示例 | 无效示例 |
|--------|------|----------|------|----------|----------|
| `cost_of_debt` | float | >=0 | 税前债务成本 | 0.05, 0.08 | -0.01 |
| `risk_free_rate` | float | >=0 | 无风险利率 | 0.03, 0.035 | -0.01 |
| `market_risk_premium` | float | >=0 | 市场风险溢价 | 0.05, 0.06 | -0.01 |
| `exit_multiple` | float | >=0 | 退出乘数 | 15.0, 18.0 | -1.0 |
| `perpetual_growth_rate` | float | >=0 | 永续增长率 | 0.03, 0.035 | -0.01 |
| `llm_max_tokens` | int | >=1 | 最大Token数 | 1000, 4000 | 0, -100 |

#### 枚举值约束参数
| 参数名 | 类型 | 可选值 | 说明 | 有效示例 | 无效示例 |
|--------|------|--------|------|----------|----------|
| `op_margin_forecast_mode` | string | "historical_median", "transition_to_target" | 利润率预测模式 | "historical_median" | "invalid_mode" |
| `wacc_weight_mode` | string | "target", "market" | WACC权重模式 | "target" | "invalid_mode" |
| `terminal_value_method` | string | "exit_multiple", "perpetual_growth" | 终值计算方法 | "exit_multiple" | "invalid_method" |

#### 条件约束参数
| 参数名 | 类型 | 依赖条件 | 约束说明 | 有效示例 | 无效示例 |
|--------|------|----------|----------|----------|----------|
| `target_operating_margin` | float | op_margin_forecast_mode="transition_to_target" | 必须提供目标利润率 | 0.30 | null |
| `op_margin_transition_years` | int | op_margin_forecast_mode="transition_to_target" | 必须提供过渡年数 | 5 | null |
| `exit_multiple` | float | terminal_value_method="exit_multiple" | 必须提供退出乘数 | 15.0 | null |
| `perpetual_growth_rate` | float | terminal_value_method="perpetual_growth" | 必须提供永续增长率 | 0.03 | null |
| `target_debt_ratio` | float | wacc_weight_mode="target" | 建议提供目标债务比例 | 0.35 | null |

### 股票筛选接口参数约束

#### 相对约束参数
| 参数名 | 类型 | 约束条件 | 说明 | 有效示例 | 无效示例 |
|--------|------|----------|------|----------|----------|
| `pe_max` | float | >=pe_min | 最大市盈率 | 30 (当pe_min=10) | 10 (当pe_min=30) |
| `pb_max` | float | >=pb_min | 最大市净率 | 5 (当pb_min=1) | 1 (当pb_min=5) |
| `market_cap_max` | float | >=market_cap_min | 最大市值 | 1000 (当market_cap_min=100) | 100 (当market_cap_min=1000) |

#### 非负约束参数
| 参数名 | 类型 | 约束条件 | 说明 | 有效示例 | 无效示例 |
|--------|------|----------|------|----------|----------|
| `pe_min` | float | >=0 | 最小市盈率 | 10, 0 | -5 |
| `pe_max` | float | >=0 | 最大市盈率 | 30, 9999 | -10 |
| `pb_min` | float | >=0 | 最小市净率 | 1, 0 | -2 |
| `pb_max` | float | >=0 | 最大市净率 | 5, 9999 | -3 |
| `market_cap_min` | float | >=0 | 最小市值 | 100, 0 | -50 |
| `market_cap_max` | float | >=0 | 最大市值 | 1000, 999999 | -100 |

### 参数验证错误码对照表

| 错误类型 | HTTP状态码 | 错误码示例 | 说明 |
|----------|------------|------------|------|
| 参数缺失 | 400 | `field required` | 必填参数未提供 |
| 数值超出范围 | 400 | `ensure this value is less than or equal to 20` | 数值大于最大允许值 |
| 数值小于最小值 | 400 | `ensure this value is greater than or equal to 0` | 数值小于最小允许值 |
| 枚举值无效 | 400 | `value is not a valid enumeration member` | 提供的值不在枚举范围内 |
| 格式错误 | 400 | `invalid date format` | 日期格式不正确 |
| 相对约束违反 | 400 | `ensure this value is less than or equal to maximum` | 最大值小于最小值 |
| 条件约束违反 | 422 | `target_operating_margin is required when transition mode` | 条件满足时未提供必需参数 |

---

## 📊 数据模型

### 请求模型详解

#### StockValuationRequest
```python
class StockValuationRequest(BaseModel):
    # 基础参数
    ts_code: str                                    # 股票代码
    market: Optional[str] = "A"                     # 市场标识
    valuation_date: Optional[str] = None           # 估值日期
    ltm_baseline_enabled: Optional[bool] = False    # 是否使用LTM/Interim基期
    
    # DCF假设参数
    forecast_years: int = 5                        # 预测年数
    cagr_decay_rate: Optional[float] = None        # CAGR衰减率
    op_margin_forecast_mode: str = "historical_median"  # 利润率预测模式
    
    # WACC参数
    wacc_weight_mode: str = "target"               # WACC权重模式
    target_debt_ratio: Optional[float] = None      # 目标债务比例
    risk_free_rate: Optional[float] = None         # 无风险利率
    beta: Optional[float] = None                   # 贝塔系数
    
    # 终值参数
    terminal_value_method: str = "exit_multiple"   # 终值计算方法
    exit_multiple: Optional[float] = None         # 退出乘数
    perpetual_growth_rate: Optional[float] = None  # 永续增长率
    
    # LLM参数
    request_llm_summary: bool = False              # LLM分析开关
    llm_provider: Optional[str] = None            # LLM提供商
    llm_temperature: Optional[float] = None       # 温度参数
    
    # 敏感性分析
    sensitivity_analysis: Optional[SensitivityAnalysisRequest] = None
```

#### SensitivityAnalysisRequest
```python
class SensitivityAnalysisRequest(BaseModel):
    row_axis: SensitivityAxisInput      # 行轴配置
    column_axis: SensitivityAxisInput   # 列轴配置

class SensitivityAxisInput(BaseModel):
    parameter_name: str                  # 参数名
    values: list[float]                 # 测试值列表
    step: Optional[float] = None        # 步长
    points: Optional[int] = None        # 点数
```

### 响应模型详解

#### StockValuationResponse
```python
class StockValuationResponse(BaseModel):
    stock_info: StockBasicInfoModel                    # 股票基本信息
    valuation_results: ValuationResultsContainer       # 估值结果容器
    error: Optional[str] = None                      # 错误信息
```

#### ValuationResultsContainer
```python
class ValuationResultsContainer(BaseModel):
    latest_price: Optional[float]                      # 最新价格
    current_pe: Optional[float]                        # 当前PE
    current_pb: Optional[float]                        # 当前PB
    dcf_forecast_details: Optional[DcfForecastDetails] # DCF详情
    detailed_forecast_table: Optional[list[dict]]      # 预测表格
    sensitivity_analysis_result: Optional[SensitivityAnalysisResult]  # 敏感性分析
    llm_analysis_summary: Optional[str]               # LLM分析
    data_warnings: Optional[list[str]]                # 数据警告
    historical_financial_summary: Optional[list[dict]] # 历史财务摘要
    historical_ratios_summary: Optional[list[dict]]   # 历史比率摘要
    special_industry_warning: Optional[str]          # 行业警告
```

#### DcfForecastDetails
```python
class DcfForecastDetails(BaseModel):
    enterprise_value: Optional[float]          # 企业价值
    equity_value: Optional[float]              # 股权价值
    value_per_share: Optional[float]           # 每股价值
    wacc_used: Optional[float]                 # 使用的WACC
    terminal_value: Optional[float]             # 终值
    dcf_implied_diluted_pe: Optional[float]    # DCF隐含PE
    base_ev_ebitda: Optional[float]            # EV/EBITDA
    implied_perpetual_growth_rate: Optional[float]  # 隐含永续增长率
    # ... 其他字段
```

---

## ⚠️ 错误处理

### 错误类型分类

#### 1. 参数验证错误 (400)
- **触发条件**: 请求参数格式错误或缺失必需参数
- **错误示例**:
```json
{
    "detail": "field required (type=value_error.missing)"
}
```

#### 2. 数据未找到错误 (404)
- **触发条件**: 股票代码不存在或数据不可用
- **错误示例**:
```json
{
    "detail": "无法获取股票基本信息: 999999.SH"
}
```

#### 3. 业务逻辑错误 (422)
- **触发条件**: 参数值不符合业务规则
- **错误示例**:
```json
{
    "detail": "预测期年数必须在1-20之间"
}
```

#### 4. 服务器内部错误 (500)
- **触发条件**: 计算过程异常、外部服务调用失败
- **错误示例**:
```json
{
    "detail": "服务器内部错误: 计算WACC失败"
}
```

注意：
- `target_debt_ratio` 指 WACC 权重用的债务比例 D/(D+E)（市值口径），请勿与“资产负债率”（账面口径）混淆。
- 当启用 `ltm_baseline_enabled` 且缺少必要的上年同口径或上年年报导致 LTM 计算失败时，系统自动回退为“年报基期”，并在 `debug_request_slice.baseline_debug` 给出提示。

### 错误处理机制

#### 1. 输入验证
```python
# Pydantic模型自动验证
class StockValuationRequest(BaseModel):
    forecast_years: int = Field(default=5, ge=1, le=20)
    # 自动验证范围约束
```

#### 2. 业务逻辑验证
```python
# 业务规则检查
if latest_price is None or latest_price <= 0:
    raise HTTPException(
        status_code=404, 
        detail=f"无法获取有效的最新价格: {request.ts_code}"
    )
```

#### 3. 异常捕获和转换
```python
try:
    # 业务逻辑
    result = valuation_service.run_single_valuation(...)
except Exception as e:
    logger.error(f"Unexpected error: {e}")
    raise HTTPException(status_code=500, detail=f"服务器内部错误: {str(e)}")
```

#### 4. 警告信息处理
```python
# 非致命警告收集
all_warnings = base_data_warnings + base_run_warnings
data_warnings = list(set(all_warnings)) if all_warnings else None
```

### 数据验证规则

#### 数值范围验证
- `forecast_years`: 1-20年
- `target_operating_margin`: 0-1之间
- `llm_temperature`: 0.0-2.0之间
- `llm_top_p`: 0.0-1.0之间

#### 格式验证
- `ts_code`: 符合股票代码格式 (如 600519.SH)
- `valuation_date`: YYYY-MM-DD格式
- `wacc_weight_mode`: 枚举值 (target/market)

#### 业务规则验证
- 历史数据完整性检查
- 财务数据有效性验证
- 计算结果合理性验证

---

## 🚀 部署和测试

### 环境要求
- Python 3.8+
- FastAPI 0.100+
- uvicorn 0.20+
- pandas 2.0+
- pydantic 2.0+

### 本地部署
```bash
# 安装依赖
pip install -e .

# 启动服务
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8124

# 或使用项目脚本
python -m uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8124
```

### 生产部署
```bash
# 使用gunicorn启动
gunicorn src.api.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8124

# 使用docker
docker build -t stock-valuation-api .
docker run -p 8124:8124 stock-valuation-api
```

### API文档
启动服务后访问：
- Swagger UI: http://localhost:8124/docs
- ReDoc: http://localhost:8124/redoc
- OpenAPI Schema: http://localhost:8124/openapi.json

### 测试脚本

#### 1. 健康检查测试
```bash
# 基础健康检查
curl http://localhost:8124/

# 预期响应
# {"message": "Welcome to the Stock Valuation API (Streamlit Backend)"}
```

#### 2. 股票估值测试

##### 基础估值测试
```bash
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 5,
    "request_llm_summary": false
  }'
```

##### 完整参数估值测试
```bash
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "market": "A",
    "valuation_date": "2024-12-31",
    "forecast_years": 10,
    "cagr_decay_rate": 0.15,
    "op_margin_forecast_mode": "transition_to_target",
    "target_operating_margin": 0.30,
    "op_margin_transition_years": 5,
    "wacc_weight_mode": "target",
    "target_debt_ratio": 0.35,
    "cost_of_debt": 0.045,
    "risk_free_rate": 0.03,
    "beta": 1.1,
    "market_risk_premium": 0.055,
    "size_premium": 0.015,
    "terminal_value_method": "exit_multiple",
    "exit_multiple": 16.5,
    "request_llm_summary": true,
    "llm_provider": "deepseek",
    "llm_model_id": "deepseek-chat",
    "llm_temperature": 0.8,
    "llm_top_p": 0.9,
    "llm_max_tokens": 3000
  }'
```

##### 敏感性分析测试
```bash
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 5,
    "sensitivity_analysis": {
      "row_axis": {
        "parameter_name": "wacc",
        "values": [0.08, 0.09, 0.10, 0.11],
        "step": 0.01,
        "points": 4
      },
      "column_axis": {
        "parameter_name": "exit_multiple",
        "values": [14, 16, 18, 20],
        "step": 2,
        "points": 4
      }
    }
  }'
```

##### 边界值测试
```bash
# 测试最小预测年数
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 1
  }'

# 测试最大预测年数
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 20
  }'

# 测试LLM参数边界值
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 5,
    "request_llm_summary": true,
    "llm_temperature": 0.0,
    "llm_top_p": 1.0,
    "llm_max_tokens": 100
  }'
```

##### 错误请求测试
```bash
# 测试无效股票代码
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "999999.SH",
    "forecast_years": 5
  }'

# 测试参数超出范围
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{
    "ts_code": "600519.SH",
    "forecast_years": 25,
    "cagr_decay_rate": 1.5,
    "llm_temperature": 3.0
  }'
```

#### 3. 数据更新测试

##### 股票基本信息更新测试
```bash
# 基础更新
curl -X POST "http://localhost:8124/update_stock_basic" \
  -H "Content-Type: application/json" \
  -d '{}'

# 预期成功响应
# {
#   "status": "success",
#   "message": "股票基本信息更新成功，共5,234条记录",
#   "count": 5234
# }
```

##### 每日行情更新测试
```bash
# 更新每日行情
curl -X POST "http://localhost:8124/update_daily_basic" \
  -H "Content-Type: application/json" \
  -d '{}'

# 预期成功响应
# {
#   "status": "success",
#   "message": "每日行情指标更新成功，共4,856条记录",
#   "count": 4856
# }
```

#### 4. 股票筛选测试

##### 基础筛选测试
```bash
# 低PE股票筛选
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "pe_min": 0,
    "pe_max": 15,
    "market_cap_min": 100
  }'

# 大盘蓝筹股筛选
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "market_cap_min": 1000,
    "pe_max": 25,
    "pb_max": 5
  }'

# 价值股筛选
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "pe_max": 15,
    "pb_max": 2,
    "market_cap_max": 500
  }'
```

##### 边界值测试
```bash
# 测试极值筛选
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "pe_min": 0,
    "pe_max": 9999,
    "pb_min": 0,
    "pb_max": 9999,
    "market_cap_min": 0,
    "market_cap_max": 999999
  }'

# 测试空筛选条件
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{}'
```

##### 错误请求测试
```bash
# 测试无效参数组合
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "pe_min": 30,
    "pe_max": 10,
    "market_cap_min": 5000,
    "market_cap_max": 1000
  }'

# 测试负值
curl -X POST "http://localhost:8124/screen_stocks" \
  -H "Content-Type: application/json" \
  -d '{
    "pe_min": -5,
    "pb_min": -1
  }'
```

### 性能测试

#### 并发测试
```bash
# 使用ab测试
ab -n 100 -c 10 -p test_request.json -T application/json http://localhost:8124/api/v1/valuation

# 使用wrk测试
wrk -t12 -c400 -d30s -s post.lua http://localhost:8124/api/v1/valuation
```

#### 响应时间监控
- 单次估值请求: 3-10秒 (取决于LLM调用)
- 数据更新请求: 10-60秒
- 股票筛选请求: 1-3秒

---

## 💡 使用示例

### Python客户端示例

```python
import requests
import json

# API配置
API_BASE_URL = "http://localhost:8124"

def calculate_stock_valuation(ts_code, forecast_years=5, request_llm=True):
    """计算股票估值"""
    
    # 构建请求参数
    request_data = {
        "ts_code": ts_code,
        "forecast_years": forecast_years,
        "request_llm_summary": request_llm,
        "terminal_value_method": "exit_multiple",
        "wacc_weight_mode": "target",
        # 可选：自定义WACC参数
        "risk_free_rate": 0.03,
        "market_risk_premium": 0.05,
        "beta": 1.0
    }
    
    try:
        # 发送请求
        response = requests.post(
            f"{API_BASE_URL}/api/v1/valuation",
            headers={"Content-Type": "application/json"},
            json=request_data,
            timeout=60  # 60秒超时
        )
        
        response.raise_for_status()
        result = response.json()
        
        # 解析结果
        stock_info = result["stock_info"]
        valuation = result["valuation_results"]
        dcf_details = valuation["dcf_forecast_details"]
        
        print(f"股票: {stock_info['name']} ({stock_info['ts_code']})")
        print(f"当前价格: {valuation['latest_price']:.2f}")
        print(f"DCF估值: {dcf_details['value_per_share']:.2f}")
        print(f"估值差异: {((dcf_details['value_per_share'] / valuation['latest_price'] - 1) * 100):.1f}%")
        
        if valuation["llm_analysis_summary"]:
            print("\n=== LLM分析 ===")
            print(valuation["llm_analysis_summary"])
            
        return result
        
    except requests.exceptions.RequestException as e:
        print(f"请求失败: {e}")
        return None

def screen_stocks(pe_min=None, pe_max=None, market_cap_min=None):
    """筛选股票"""
    
    request_data = {}
    if pe_min is not None:
        request_data["pe_min"] = pe_min
    if pe_max is not None:
        request_data["pe_max"] = pe_max
    if market_cap_min is not None:
        request_data["market_cap_min"] = market_cap_min
    
    try:
        response = requests.post(
            f"{API_BASE_URL}/screen_stocks",
            headers={"Content-Type": "application/json"},
            json=request_data
        )
        
        response.raise_for_status()
        result = response.json()
        
        print(f"找到 {result['count']} 只符合条件的股票:")
        for stock in result["data"][:10]:  # 显示前10只
            print(f"{stock['name']} ({stock['ts_code']}): PE={stock['pe']:.1f}, 市值={stock['total_mv']:.0f}亿")
            
        return result
        
    except requests.exceptions.RequestException as e:
        print(f"筛选失败: {e}")
        return None

# 使用示例
if __name__ == "__main__":
    # 计算贵州茅台估值
    result = calculate_stock_valuation("600519.SH", forecast_years=5)
    
    # 筛选低PE股票
    screen_stocks(pe_min=0, pe_max=20, market_cap_min=100)
```

### JavaScript客户端示例

```javascript
// API配置
const API_BASE_URL = 'http://localhost:8124';

// 计算股票估值
async function calculateStockValuation(tsCode, options = {}) {
    const requestData = {
        ts_code: tsCode,
        forecast_years: options.forecastYears || 5,
        request_llm_summary: options.requestLLM || false,
        ...options
    };
    
    try {
        const response = await fetch(`${API_BASE_URL}/api/v1/valuation`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(requestData)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const result = await response.json();
        
        console.log(`股票: ${result.stock_info.name} (${result.stock_info.ts_code})`);
        console.log(`当前价格: ${result.valuation_results.latest_price}`);
        console.log(`DCF估值: ${result.valuation_results.dcf_forecast_details.value_per_share}`);
        
        return result;
        
    } catch (error) {
        console.error('请求失败:', error);
        throw error;
    }
}

// 股票筛选
async function screenStocks(filters = {}) {
    const requestData = {};
    
    if (filters.peMin !== undefined) requestData.pe_min = filters.peMin;
    if (filters.peMax !== undefined) requestData.pe_max = filters.peMax;
    if (filters.marketCapMin !== undefined) requestData.market_cap_min = filters.marketCapMin;
    
    try {
        const response = await fetch(`${API_BASE_URL}/screen_stocks`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(requestData)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const result = await response.json();
        console.log(`找到 ${result.count} 只符合条件的股票`);
        
        return result;
        
    } catch (error) {
        console.error('筛选失败:', error);
        throw error;
    }
}

// 使用示例
(async () => {
    try {
        // 计算估值
        const valuation = await calculateStockValuation('600519.SH', {
            forecastYears: 5,
            requestLLM: true
        });
        
        // 筛选股票
        const screened = await screenStocks({
            peMin: 0,
            peMax: 20,
            marketCapMin: 100
        });
        
    } catch (error) {
        console.error('API调用失败:', error);
    }
})();
```

### 批量处理示例

```python
import asyncio
import aiohttp
import json

async def batch_valuation(session, stock_codes):
    """批量计算股票估值"""
    
    tasks = []
    for ts_code in stock_codes:
        task = calculate_single_valuation(session, ts_code)
        tasks.append(task)
    
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results

async def calculate_single_valuation(session, ts_code):
    """单只股票估值"""
    
    request_data = {
        "ts_code": ts_code,
        "forecast_years": 5,
        "request_llm_summary": False  # 批量时关闭LLM以提高速度
    }
    
    try:
        async with session.post(
            f"{API_BASE_URL}/api/v1/valuation",
            json=request_data,
            timeout=aiohttp.ClientTimeout(total=30)
        ) as response:
            result = await response.json()
            return {
                "ts_code": ts_code,
                "success": True,
                "data": result
            }
    except Exception as e:
        return {
            "ts_code": ts_code,
            "success": False,
            "error": str(e)
        }

# 使用示例
async def main():
    stock_codes = ["600519.SH", "000858.SZ", "600036.SH"]
    
    async with aiohttp.ClientSession() as session:
        results = await batch_valuation(session, stock_codes)
        
        for result in results:
            if result["success"]:
                stock_info = result["data"]["stock_info"]
                dcf_details = result["data"]["valuation_results"]["dcf_forecast_details"]
                print(f"{stock_info['name']}: {dcf_details['value_per_share']:.2f}")
            else:
                print(f"{result['ts_code']}: 失败 - {result['error']}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📝 最佳实践

### 1. 请求优化
- **批量处理**: 使用异步请求批量处理多只股票
- **缓存策略**: 对相同参数的估值结果进行缓存
- **超时设置**: 根据网络环境合理设置超时时间
- **重试机制**: 对临时性错误实现指数退避重试

### 2. 错误处理
- **参数验证**: 在发送请求前验证参数格式
- **状态码检查**: 检查HTTP状态码和业务错误码
- **异常捕获**: 捕获网络异常和解析异常
- **日志记录**: 记录请求和响应信息用于调试

### 3. 性能优化
- **并发控制**: 合理控制并发请求数量
- **LLM调用**: 批量处理时关闭LLM分析以提高速度
- **数据分页**: 对大量结果进行分页处理
- **连接池**: 使用HTTP连接池减少连接开销

### 4. 安全考虑
- **HTTPS**: 生产环境使用HTTPS协议
- **认证授权**: 实现API密钥或OAuth认证
- **限流控制**: 实现请求频率限制
- **输入验证**: 严格验证所有输入参数

---

## 🔧 故障排除

### 常见问题

#### 1. 连接超时
**问题**: 请求超时失败
**解决**: 
- 增加超时时间到60秒
- 检查网络连接
- 验证服务是否正常运行

#### 2. 数据未找到
**问题**: 返回404错误
**解决**:
- 验证股票代码格式
- 检查数据是否已更新
- 确认股票是否存在

#### 3. 计算失败
**问题**: 返回500错误
**解决**:
- 检查请求参数是否合理
- 查看服务器日志
- 验证财务数据完整性

#### 4. LLM调用失败
**问题**: LLM分析返回错误
**解决**:
- 检查LLM服务配置
- 验证API密钥和网络连接
- 尝试使用不同的LLM提供商

### 调试技巧

#### 1. 启用详细日志
```bash
# 设置日志级别为DEBUG
export LOG_LEVEL=DEBUG
uvicorn src.api.main:app --reload
```

#### 2. 使用Swagger UI
访问 http://localhost:8124/docs 进行交互式测试

#### 3. 检查请求/响应
```python
import logging

# 启用请求日志
logging.basicConfig(level=logging.DEBUG)
```

#### 4. 验证数据格式
```python
# 使用Pydantic验证数据
from pydantic import ValidationError

try:
    request = StockValuationRequest(**data)
except ValidationError as e:
    print(f"验证错误: {e}")
```

---

## 📊 监控和运维

### 关键指标
- **响应时间**: 单次请求处理时间
- **错误率**: HTTP错误码分布
- **并发数**: 同时处理的请求数
- **内存使用**: 服务内存占用
- **CPU使用**: 服务CPU占用率

### 日志监控
```python
# 关键日志模式
INFO - Received valuation request for: 600519.SH
INFO - Step 1: Fetching data...
INFO - Step 2: Processing data...
INFO - Base case valuation successful.
ERROR - Error during LLM call: API timeout
```

### 健康检查
```bash
# 基础健康检查
curl http://localhost:8124/

# 详细健康检查
curl http://localhost:8124/health
```

### 性能优化建议
1. **数据库优化**: 添加适当的索引
2. **缓存策略**: 实现Redis缓存
3. **异步处理**: 使用异步I/O提高并发
4. **负载均衡**: 部署多实例分担负载

---

## 📅 最新更新记录

### v3.0.0 (2025-09-22)
**重大功能更新**:
- ✅ **TuShare数据源**: 完全集成TuShare API，支持数据源热切换
- ✅ **LTM基线功能**: 新增Last Twelve Months收入基期支持
- ✅ **数据清洗优化**: 修复资产负债表基准日回退问题，保护关键科目
- ✅ **敏感性分析增强**: 新增EV/EBITDA倍数、隐含PE/PGR等指标
- ✅ **生产级部署**: 完整的错误处理、兜底机制和监控能力

**API接口更新**:
- 新增 `ltm_baseline_enabled` 参数启用LTM基线
- 增强估值响应字段，包含更多诊断信息
- 优化错误处理和用户友好的错误消息

**数据质量提升**:
- TuShare与PostgreSQL数据源端到端验证完成
- 关键估值指标(EV/VPS/CAGR/EBITDA)对齐确认
- 按年择优数据选择策略，确保使用最新年报

**性能优化**:
- 数据获取链路优化，减少API调用次数
- Decimal精度计算，避免浮点误差
- 异步处理支持，提高并发处理能力

### v2.x.x (2025-09之前)
**历史功能**:
- 基础DCF估值计算
- PostgreSQL数据源集成
- Streamlit前端
- 基础敏感性分析
- 股票筛选器

### 下一版本计划 (v3.1.0)
**计划功能**:
- 性能缓存机制
- 批量估值API
- 更多行业模型支持
- 实时数据流处理

---

这份API文档提供了完整的技术规格和使用指南，方便其他项目进行对接和测试。项目已达到生产级标准，具备完整的估值分析能力。如有任何问题，请参考故障排除部分或联系技术支持。
