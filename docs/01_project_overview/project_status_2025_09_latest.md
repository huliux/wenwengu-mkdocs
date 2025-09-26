# 股票估值系统项目状态报告
*最后更新: 2025-09-26*

## 执行摘要

本项目是一个基于DCF模型的专业股票估值系统，当前已完成从Proof of Concept到Production Ready的转换。系统实现了混合数据源架构（TuShare + PostgreSQL），具备完整的估值链路、LLM智能分析、敏感性分析、股票筛选等功能。

**当前版本**: v3.0.0 (feature/user-auth分支)
**核心技术栈**: Python 3.9+, FastAPI, Streamlit, PostgreSQL, TuShare API
**架构模式**: 混合数据源 + 模块化DCF计算 + LLM智能分析
**部署状态**: 开发环境运行(Streamlit端口8501)，生产就绪

### 关键成就
- ✅ **生产级数据源**: TuShare接口完全对齐PostgreSQL标准，支持数据源热切换
- ✅ **模块化DCF引擎**: 6个独立计算器，覆盖从NWC到股权价值的完整链路
- ✅ **Streamlit前端架构**: 基于Streamlit的现代化Web界面，支持用户友好操作
- ✅ **用户认证系统**: 完整的用户注册/登录/会话管理，数据库从SQLite迁移至Postgres/Supabase
- ✅ **智能缓存系统**: 估值结果缓存与会话恢复，大幅提升用户体验和系统性能
- ✅ **LLM集成**: 支持DeepSeek/OpenAI兼容API，生成投资分析报告
- ✅ **敏感性分析**: 多维度估值sensitivity table，支持终值法/永续增长法
- ✅ **股票筛选器**: 基于财务指标的多条件筛选
- ✅ **LTM基线**: 支持Last Twelve Months收入基期，提升时效性

## 当前架构概览

### 系统架构（2025-09版本）
```
前端层 (Streamlit)
└── Streamlit: frontend-streamlit/streamlit_app.py

API服务层
└── FastAPI: src/api/main.py (端口8124)
    ├── 估值接口: /api/v1/valuation
    ├── 敏感性分析: /api/v1/sensitivity
    └── 股票筛选: /api/v1/screener

业务逻辑层
├── 核心计算引擎: src/core/
│   ├── calculators/dcf/: DCF计算器集群(6个模块)
│   ├── financial/: 财务预测与处理引擎
│   └── screener/: 股票筛选器
├── 数据服务: src/data/
│   ├── fetchers/: 数据获取器(TuShare/PostgreSQL)
│   └── processors/: 数据清洗与处理
└── 业务服务: src/services/valuation_service.py

数据层
├── TuShare API: 实时金融数据(主要)
├── PostgreSQL: 历史数据存储/校验基准
└── Redis: 缓存层(配置项)
```

### 核心模块统计
- **源代码文件**: 34个Python模块
- **测试文件**: 27个测试模块
- **前端文件**: 17个前端组件
- **配置文件**: 支持uv/pip双包管理，pre-commit代码质量保障

## 主要功能模块

### 1. 用户认证与会话管理
**位置**: `src/auth/`, `frontend-streamlit/auth/`

**功能特性**:
- 完整的用户注册、登录、会话管理系统
- 数据库架构迁移：SQLite → PostgreSQL/Supabase，支持高并发和云原生部署
- 会话持久化：用户登录状态和估值结果缓存，提升用户体验
- 认证中间件：保护敏感功能和数据访问
- 安全特性：密码哈希、会话令牌、防CSRF等

**关键改进**(2025-09):
- 解决SQLite并发锁问题，迁移至成熟的关系型数据库
- 智能缓存机制，估值结果在会话间持久保存
- 优化用户界面，简化认证流程

### 2. 数据获取与处理
**位置**: `src/data/fetchers/`, `src/core/financial/processor.py`

**功能特性**:
- 混合数据源架构: `DATA_SOURCE=tushare/postgres/hybrid`
- TuShare年报严格筛选: `end_type='4'`, `update_flag='1'`优先
- 字段映射统一: 应收账款+票据聚合、固定资产回退、EPS去重
- 数据清洗保护: 关键科目(收入/利润/资产/负债)免受异常值误判
- LTM支持: 季度+年报组装Last Twelve Months基线

**关键改进**(2025-09):
- 修复资产负债表基准日回退到2023的问题
- 保护列机制防止结构性跃迁被误判为异常
- 按年择优保留(不再丢失最新年度数据)

### 2. DCF估值计算引擎
**位置**: `src/core/calculators/dcf/`, `src/core/financial/forecaster.py`

**模块组成**:
1. **NwcCalculator**: 净营运资本计算与预测
2. **FcfCalculator**: 无杠杆自由现金流计算
3. **WaccCalculator**: 加权平均资本成本计算
4. **TerminalValueCalculator**: 终值计算(退出倍数/永续增长)
5. **PresentValueCalculator**: 现值折现计算
6. **EquityBridgeCalculator**: 企业价值到股权价值桥接

**估值链路**:
```
历史数据 → 清洗处理 → 比率中位数 → 财务预测 → UFCF计算 → 终值计算 → 现值折现 → 股权价值 → 每股价值
```

**关键特性**:
- 历史比率中位数驱动预测
- 支持CAGR衰减与过渡到目标值
- 名义GDP上限约束(中国市场)
- Decimal精度计算避免浮点误差

### 3. LLM智能分析
**位置**: `src/api/llm_utils.py`, `src/config/llm_prompt_template.md`

**支持模型**:
- DeepSeek Chat
- OpenAI兼容API(自定义)
- 环境变量配置: `LLM_PROVIDER`, `*_API_KEY`

**分析内容**:
- 估值结果解读与投资建议
- 关键假设敏感性分析
- 行业对比与风险提示
- 中文专业报告生成

### 4. 敏感性分析系统
**位置**: `src/api/sensitivity_models.py`

**分析维度**:
- WACC vs 永续增长率
- WACC vs 退出倍数
- 收入CAGR vs EBITDA利润率
- 收入CAGR vs 终值方法

**输出指标**:
- 企业价值(EV)
- 每股价值(VPS)
- EV/EBITDA倍数
- 隐含PE倍数
- 隐含永续增长率

### 5. 股票筛选器
**位置**: `src/core/screener/`

**筛选条件**:
- 市值范围
- 估值倍数(PE/PB/EV/EBITDA)
- 财务指标(ROE/ROA/收入增长率)
- 行业/板块过滤

## 开发工具链

### 代码质量保障
- **格式化**: Black + isort
- **静态检查**: Ruff + MyPy(逐步启用)
- **安全扫描**: Bandit + Safety
- **预提交钩子**: pre-commit配置
- **测试覆盖**: pytest + pytest-cov

### 包管理
- **主要**: uv(推荐,更快更可靠)
- **备选**: pip + requirements.txt
- **依赖组**: `[dev]`, `[test]`, `[docs]`

### 运行命令
```bash
# 环境设置
uv venv && source .venv/bin/activate
uv pip install -e ".[dev,test]"

# 代码质量
pre-commit run --all-files
black . && isort . && ruff check --fix .

# 测试
pytest --cov=. --cov-report=html

# 应用运行
streamlit run frontend-streamlit/streamlit_app.py
uvicorn src.api.main:app --reload --port 8124
```

## 数据源对比与验证

### TuShare vs PostgreSQL 对齐结果
**测试案例**: 000999.SZ, 600519.SH
**对齐维度**: 基准日期、历史CAGR、最新EBITDA、NWC组件、EV估值

**对齐状况**:
- ✅ **基准报表日期**: 2024-12-31 (一致)
- ✅ **历史收入CAGR**: 21.698%/17.184% (完全一致)
- ✅ **最新实际EBITDA**: 600519完全一致，000999接近
- ✅ **DCF计算链路**: WACC、终值、现值逻辑一致
- ⚠️ **VPS差异**: 主要由股本快照时间差异(TS更及时)

### 数据源选择建议
- **生产估值**: `DATA_SOURCE=tushare` (更及时、更完整)
- **历史复现**: `DATA_SOURCE=postgres` 或 TS的PG兼容模式
- **开发调试**: `DATA_SOURCE=hybrid` (PG优先,TS回退)

## 已知问题与限制

### 1. 行业适用性
- ✅ **非金融企业**: 完全适用
- ⚠️ **金融企业**: 部分指标不适用(银行/保险等)，会给出警告

### 2. 数据质量依赖
- TuShare数据质量直接影响估值结果
- 需要定期校验与PG基准的一致性
- 清洗策略需要持续优化

### 3. 性能考虑
- TuShare API有调用频率限制
- 建议增加本地缓存与重试机制
- 大批量筛选时可能较慢

## 下一步规划

### 短期优化(1-2周)
- [ ] 性能优化: 本地缓存机制
- [ ] 清洗策略: 软化NWC组件清洗规则
- [ ] 透明度: 市场快照trade_date透传到前端
- [ ] 测试: 提升覆盖率到80%+

### 中期增强(1个月)
- [ ] 行业专用模型: 金融企业估值模型
- [ ] 数据监控: 定期TS↔PG基线漂移检查
- [ ] 多市场支持: 港股/美股数据源
- [ ] 批量估值: 支持股票池批量处理

### 长期愿景(3个月)
- [ ] 实时流处理: 盘中实时估值更新
- [ ] 机器学习: 历史估值准确性反馈学习
- [ ] 云原生部署: Docker + K8s生产部署
- [ ] 开放API: 第三方集成接口

## 技术债务

### 高优先级
1. **类型注解**: MyPy覆盖率仍较低，需要逐步补强
2. **异常处理**: 部分模块缺乏完整的错误处理链路
3. **文档同步**: 代码变更后文档更新滞后

### 中优先级
1. **日志系统**: 统一的日志格式与级别管理
2. **配置管理**: 环境变量管理可以更加结构化
3. **数据库连接**: 连接池与事务管理优化

## 结论

项目已达到生产可用状态，核心功能完整且经过验证。TuShare数据源的引入显著提升了系统的时效性和数据完整性。当前版本(3.0)在科学性、准确性、实用性方面均满足专业投资分析需求。

下一阶段的重点是性能优化、用户体验提升和功能扩展，为更大规模的生产应用做准备。