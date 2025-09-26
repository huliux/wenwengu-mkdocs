# 测试策略与质量保障指南
*最后更新: 2025-09-22*

## 概述

本文档详细说明了股票估值系统的测试策略、质量保障流程和持续集成实践。项目目前拥有27个测试文件，覆盖单元测试、集成测试、API测试等多个层次。

## 测试架构

### 测试分层策略

```
┌─────────────────────────────────────┐
│           E2E Tests                  │  ← 端到端测试
├─────────────────────────────────────┤
│          API Tests                   │  ← API接口测试
├─────────────────────────────────────┤
│       Integration Tests              │  ← 集成测试
├─────────────────────────────────────┤
│         Unit Tests                   │  ← 单元测试
└─────────────────────────────────────┘
```

### 测试目录结构
```
tests/
├── unit/                    # 单元测试
│   ├── core/               # 核心模块测试
│   ├── data/               # 数据层测试
│   └── services/           # 服务层测试
├── integration/            # 集成测试
├── api/                    # API测试
│   ├── test_main.py       # 主要API端点
│   ├── test_sensitivity.py # 敏感性分析
│   └── test_gdp_cap.py    # GDP上限测试
├── fixtures/               # 测试数据
└── conftest.py            # pytest配置
```

## 单元测试

### 核心计算器测试

#### WACC计算器测试
**文件**: `tests/test_wacc_calculator.py`

**测试覆盖**:
- 基本WACC计算
- 边界条件(零值、负值)
- 不同资本结构场景
- 市场价值vs账面价值权重

```python
def test_wacc_calculation_basic():
    """测试基本WACC计算"""
    calculator = WaccCalculator()
    result = calculator.calculate_wacc(
        market_value_equity=Decimal('100'),
        market_value_debt=Decimal('50'),
        cost_of_equity=Decimal('0.12'),
        cost_of_debt=Decimal('0.06'),
        tax_rate=Decimal('0.25')
    )
    expected_wacc = Decimal('0.1')  # 10%
    assert abs(result.wacc - expected_wacc) < Decimal('0.001')
```

#### 现值计算器测试
**文件**: `tests/test_present_value_calculator.py`

**测试覆盖**:
- 现金流现值计算
- 终值现值计算
- 不同折现率场景
- Decimal精度验证

#### 其他计算器测试
- `tests/test_fcf_calculator.py`: 自由现金流计算
- `tests/test_terminal_value_calculator.py`: 终值计算
- `tests/test_nwc_calculator.py`: 净营运资本计算
- `tests/test_equity_bridge_calculator.py`: 股权价值桥接

### 数据处理测试

#### 财务预测器测试
**文件**: `tests/test_financial_forecaster.py`

**测试覆盖**:
- 收入预测算法
- 历史比率计算
- CAGR衰减逻辑
- 边界条件处理

#### 数据处理器测试
**文件**: `tests/test_data_processor.py`

**测试覆盖**:
- 数据清洗逻辑
- 异常值检测
- 缺失值填充
- 历史比率计算

### 专项算法测试

#### 收入预测算法测试
**文件群**: `tests/test_enhanced_revenue_forecast*.py`

**测试场景**:
- 简单CAGR预测
- 带衰减的CAGR预测
- 过渡到目标值预测
- 数据库集成测试

```python
def test_revenue_forecast_with_decay():
    """测试带衰减的收入预测"""
    historical_revenues = [100, 120, 150, 180]
    forecaster = EnhancedRevenueForecast()

    projections = forecaster.project_revenue(
        historical_revenues=historical_revenues,
        forecast_years=5,
        decay_rate=0.1
    )

    # 验证预测逻辑
    assert len(projections) == 5
    assert all(p > 0 for p in projections)
    # 验证衰减效应
    growth_rates = [projections[i]/projections[i-1] - 1
                   for i in range(1, len(projections))]
    assert all(growth_rates[i] <= growth_rates[i-1]
              for i in range(1, len(growth_rates)))
```

## 集成测试

### 完整估值流程测试
**文件**: `tests/test_sensitivity_analysis_integration.py`

**测试覆盖**:
- 数据获取→处理→计算→输出完整链路
- 多个股票代码测试
- 不同参数组合验证
- 错误处理流程

### 数据源集成测试
**测试场景**:
- TuShare API集成
- PostgreSQL数据库集成
- 数据源切换测试
- 缓存机制测试

## API测试

### 主要API端点测试
**文件**: `tests/api/test_main.py`

**测试覆盖**:
- `/api/v1/valuation` 估值接口
- 请求参数验证
- 响应格式验证
- 错误处理测试

```python
@pytest.mark.asyncio
async def test_valuation_endpoint():
    """测试估值API端点"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post("/api/v1/valuation", json={
            "ts_code": "600519.SH",
            "valuation_date": "2025-09-22",
            "forecast_years": 5
        })

    assert response.status_code == 200
    data = response.json()
    assert "valuation_summary" in data
    assert "enterprise_value" in data["valuation_summary"]
    assert data["valuation_summary"]["enterprise_value"] > 0
```

### 敏感性分析API测试
**文件**: `tests/api/test_sensitivity.py`

**测试覆盖**:
- 敏感性分析参数验证
- 多维度分析结果
- 输出格式验证
- 性能测试

### 特殊功能测试
**文件**: `tests/api/test_gdp_cap.py`

**测试覆盖**:
- GDP上限约束功能
- LTM基线功能
- 数据源切换功能

## 端到端测试

### E2E测试脚本
**工具**: `scripts/e2e_compare_sources.py`

**测试流程**:
1. 配置测试环境
2. 运行完整估值流程
3. 对比不同数据源结果
4. 验证关键指标一致性
5. 生成测试报告

**使用方法**:
```bash
# 单股票E2E测试
python scripts/e2e_compare_sources.py --ts 600519.SH --date 2025-09-22

# 批量E2E测试
python scripts/e2e_compare_sources.py --batch --config test_stocks.json
```

## 测试数据管理

### 测试夹具(Fixtures)
**位置**: `tests/fixtures/`

**数据类型**:
- 模拟财务报表数据
- 标准测试参数集
- 预期计算结果
- 错误场景数据

```python
# conftest.py
@pytest.fixture
def sample_financial_data():
    """提供标准测试用财务数据"""
    return {
        'income_statement': pd.DataFrame({
            'end_date': ['2021-12-31', '2022-12-31', '2023-12-31'],
            'revenue': [1000, 1200, 1500],
            'operate_profit': [150, 180, 225]
        }),
        'balance_sheet': pd.DataFrame({
            'end_date': ['2021-12-31', '2022-12-31', '2023-12-31'],
            'total_assets': [5000, 6000, 7500],
            'inventories': [200, 250, 300]
        })
    }
```

### Mock数据策略
- **外部API**: Mock TuShare API调用
- **数据库**: 使用内存SQLite数据库
- **文件系统**: 使用临时目录
- **网络请求**: Mock HTTP响应

## 性能测试

### 基准测试
**测试维度**:
- 单次估值耗时
- 敏感性分析耗时
- 数据获取耗时
- 内存使用情况

### 压力测试
**测试场景**:
- 并发API请求
- 大批量股票筛选
- 长时间运行稳定性
- 内存泄漏检测

```python
def test_valuation_performance():
    """测试估值性能"""
    import time

    start_time = time.time()
    result = run_valuation("600519.SH", "2025-09-22")
    end_time = time.time()

    execution_time = end_time - start_time
    assert execution_time < 10.0  # 10秒内完成
    assert result is not None
```

## 质量保障流程

### Pre-commit钩子
**配置**: `.pre-commit-config.yaml`

**检查项目**:
- 代码格式化(Black, isort)
- 静态分析(Ruff)
- 安全检查(Bandit)
- 测试执行(pytest)

### 持续集成
**触发条件**:
- 代码推送到主分支
- Pull Request创建/更新
- 定期调度(每日)

**CI流程**:
```yaml
# 示例GitHub Actions配置
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Install dependencies
        run: |
          pip install uv
          uv pip install -e ".[dev,test]"
      - name: Run tests
        run: pytest --cov=src --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v1
```

## 测试执行指南

### 本地测试
```bash
# 运行所有测试
pytest

# 运行特定模块测试
pytest tests/test_wacc_calculator.py -v

# 运行API测试(需要异步支持)
pytest tests/api/ -v

# 运行覆盖率测试
pytest --cov=src --cov-report=html

# 运行性能测试
pytest tests/performance/ -v

# 跳过慢测试
pytest -m "not slow"
```

### 测试标记
```python
# 标记慢测试
@pytest.mark.slow
def test_long_running_process():
    pass

# 标记集成测试
@pytest.mark.integration
def test_database_integration():
    pass

# 标记API测试
@pytest.mark.api
def test_api_endpoint():
    pass
```

### 调试测试
```bash
# 详细输出
pytest -v -s

# 停在第一个失败
pytest -x

# 运行特定测试函数
pytest tests/test_wacc_calculator.py::test_basic_calculation -v

# 使用pdb调试
pytest --pdb
```

## 测试最佳实践

### 1. 测试命名规范
- 测试函数: `test_<功能>_<场景>`
- 测试类: `Test<模块名>`
- 测试文件: `test_<模块名>.py`

### 2. 测试组织原则
- 一个测试只验证一个功能点
- 测试之间相互独立
- 使用描述性的断言消息
- 避免测试实现细节

### 3. 测试数据原则
- 使用最小化测试数据
- 避免硬编码魔法数字
- 使用工厂函数生成测试数据
- 清理测试产生的副作用

### 4. Mock使用原则
- Mock外部依赖而非内部逻辑
- 验证Mock调用参数
- 使用合理的Mock返回值
- 避免过度Mock

## 测试覆盖率目标

### 当前覆盖率状态
- **总体覆盖率**: ~47%
- **核心计算模块**: >80%
- **API层**: >70%
- **数据层**: >60%

### 覆盖率提升计划
1. **第一阶段**(目标80%):
   - 补充数据处理模块测试
   - 完善异常处理测试
   - 增加边界条件测试

2. **第二阶段**(目标90%):
   - 补充集成测试
   - 增加性能测试
   - 完善E2E测试

### 覆盖率监控
```bash
# 生成覆盖率报告
pytest --cov=src --cov-report=html --cov-report=term

# 查看详细覆盖率
coverage report -m

# 查看未覆盖代码
coverage html
open htmlcov/index.html
```

## 故障排查

### 常见测试问题

#### 1. 异步测试失败
```bash
# 确保安装pytest-asyncio
pip install pytest-asyncio

# 检查async标记
@pytest.mark.asyncio
async def test_async_function():
    pass
```

#### 2. 数据库测试失败
```bash
# 检查测试数据库配置
export TEST_DATABASE_URL="sqlite:///test.db"

# 清理测试数据
pytest --setup-show
```

#### 3. Mock未生效
```python
# 确保Mock路径正确
@patch('src.data.fetchers.tushare_fetcher.TushareAshareFetcher')
def test_with_mock(mock_fetcher):
    pass
```

### 测试环境隔离
- 使用独立的测试数据库
- 清理测试产生的文件
- 重置全局状态
- 避免测试间依赖

## 未来规划

### 短期目标(1个月)
- 提升覆盖率到80%
- 完善API测试套件
- 增加性能基准测试
- 优化测试执行速度

### 中期目标(3个月)
- 实现自动化回归测试
- 建立测试数据管理平台
- 增加契约测试
- 完善监控和告警

### 长期目标(6个月)
- 实现测试驱动开发(TDD)
- 建立测试质量度量体系
- 实现智能测试选择
- 完善测试文档和培训

---

*测试是保证代码质量的重要手段，建议所有开发者严格遵循本测试策略进行开发。*