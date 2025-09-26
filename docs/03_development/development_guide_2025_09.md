# 股票估值系统开发指南
*最后更新: 2025-09-22*

## 概述

本指南面向项目的新开发者和维护者，提供完整的开发环境设置、代码规范、测试策略和部署流程。项目当前版本(3.0)已达到生产级标准。

## 快速开始

### 环境要求
- Python 3.9+
- PostgreSQL 12+ (可选，用于数据验证)
- Redis (可选，用于缓存)

### 开发环境设置

```bash
# 1. 克隆仓库
git clone <repository-url>
cd stock_vale_valuation_3.0

# 2. 使用uv创建虚拟环境 (推荐)
uv venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# 3. 安装依赖
uv pip install -e ".[dev,test]"

# 4. 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入必要的配置

# 5. 安装pre-commit钩子
pre-commit install

# 6. 验证安装
pytest tests/unit/ -v
```

### 核心配置项

编辑 `.env` 文件：

```bash
# 数据源配置
DATA_SOURCE=tushare  # tushare/postgres/hybrid
TUSHARE_TOKEN=your_token_here
OCL_AGGREGATION_MODE=standard  # standard/pg

# LLM配置
LLM_PROVIDER=deepseek  # deepseek/custom_openai
DEEPSEEK_API_KEY=your_key_here
DEEPSEEK_MODEL_NAME=deepseek-chat

# WACC默认参数
RISK_FREE_RATE=0.03
MARKET_RISK_PREMIUM=0.05
DEFAULT_BETA=1.0
```

## 项目架构

### 目录结构
```
src/
├── api/                    # FastAPI接口层
│   ├── main.py            # API入口
│   ├── models.py          # Pydantic模型
│   ├── llm_utils.py       # LLM集成
│   └── utils.py           # API工具函数
├── core/                  # 核心业务逻辑
│   ├── calculators/       # DCF计算器集群
│   │   ├── dcf/          # DCF专用计算器
│   │   ├── base.py       # 计算器基类
│   │   └── nwc_calculator.py
│   ├── financial/        # 财务分析模块
│   │   ├── processor.py  # 数据处理引擎
│   │   └── forecaster.py # 财务预测引擎
│   └── screener/         # 股票筛选器
├── data/                 # 数据层
│   ├── fetchers/         # 数据获取器
│   └── processors/       # 数据处理器
├── services/             # 服务层
└── config/               # 配置文件

frontend-streamlit/       # Streamlit前端
tests/                   # 测试套件
dev_docs/               # 开发文档
```

### 核心设计模式

#### 1. 模块化计算器模式
每个DCF计算步骤都是独立的计算器类：

```python
# 例: WACC计算器
class WaccCalculator:
    def calculate_wacc(self,
                      market_value_equity: Decimal,
                      market_value_debt: Decimal,
                      cost_of_equity: Decimal,
                      cost_of_debt: Decimal,
                      tax_rate: Decimal) -> WaccResult:
        # 计算逻辑
        pass
```

#### 2. 数据源抽象模式
统一的数据获取接口支持多数据源：

```python
class BaseFetcher:
    def get_stock_info(self) -> dict: pass
    def get_latest_price(self, date: str) -> float: pass
    def get_raw_financial_data(self, years: int) -> dict: pass

class TushareAshareFetcher(BaseFetcher): pass
class AshareDataFetcher(BaseFetcher): pass  # PostgreSQL
```

#### 3. 服务层编排模式
ValuationService协调各个计算器：

```python
class ValuationService:
    def perform_valuation(self, request: StockValuationRequest) -> StockValuationResponse:
        # 1. 数据获取
        # 2. 数据处理
        # 3. DCF计算链路
        # 4. LLM分析
        # 5. 结果封装
```

## 开发工作流

### 代码提交流程

1. **创建功能分支**
```bash
git checkout -b feature/your-feature-name
```

2. **开发与测试**
```bash
# 代码开发
# ...

# 运行代码质量检查
pre-commit run --all-files

# 运行相关测试
pytest tests/test_your_module.py -v

# 运行覆盖率检查
pytest --cov=src tests/ --cov-report=html
```

3. **提交代码**
```bash
git add .
git commit -m "feat(module): add new feature description"
```

4. **推送与合并**
```bash
git push origin feature/your-feature-name
# 创建Pull Request
```

### 代码规范

#### Python代码风格
- **格式化**: Black (line-length=88)
- **导入排序**: isort (profile=black)
- **静态检查**: Ruff + MyPy(逐步启用)
- **安全检查**: Bandit

#### 提交信息规范
```bash
feat(module): add new feature
fix(module): fix bug description
docs(module): update documentation
test(module): add tests for feature
refactor(module): refactor code
style(module): format code
chore: update dependencies
```

#### 代码注释规范
```python
def calculate_dcf_value(
    self,
    ufcf_projections: List[Decimal],
    terminal_value: Decimal,
    wacc: Decimal
) -> Decimal:
    """
    计算DCF企业价值

    Args:
        ufcf_projections: 预测期无杠杆自由现金流列表
        terminal_value: 终值
        wacc: 加权平均资本成本

    Returns:
        Decimal: 企业价值

    Raises:
        ValueError: 当WACC为负数或UFCF为空时
    """
    # 实现逻辑
```

### 测试策略

#### 测试分类
1. **单元测试**: 测试单个函数/类的逻辑
2. **集成测试**: 测试模块间交互
3. **API测试**: 测试FastAPI端点
4. **端到端测试**: 测试完整估值流程

#### 测试命令
```bash
# 运行所有测试
pytest

# 运行特定模块测试
pytest tests/test_wacc_calculator.py -v

# 运行API测试
pytest tests/api/ -v

# 生成覆盖率报告
pytest --cov=src --cov-report=html

# 只运行快速测试(排除慢测试)
pytest -m "not slow"
```

#### 测试数据
- 使用`tests/fixtures/`目录存放测试数据
- Mock外部API调用避免依赖外部服务
- 参数化测试覆盖边界情况

### 调试技巧

#### 1. 日志配置
```python
import logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

#### 2. 环境变量调试
```bash
# 启用详细日志
export LOG_LEVEL=DEBUG

# 使用测试数据源
export DATA_SOURCE=postgres

# 跳过LLM调用
export LLM_PROVIDER=mock
```

#### 3. API调试
```bash
# 启动API服务器
uvicorn src.api.main:app --reload --port 8124

# 访问Swagger文档
open http://localhost:8124/docs

# 测试API端点
curl -X POST "http://localhost:8124/api/v1/valuation" \
  -H "Content-Type: application/json" \
  -d '{"ts_code": "600519.SH", "valuation_date": "2025-09-22"}'
```

## 性能优化

### 1. 数据获取优化
- 使用缓存减少重复API调用
- 批量获取数据避免单次调用
- 异步处理提高并发性能

### 2. 计算优化
- 使用Decimal避免浮点精度问题
- 向量化计算处理大量数据
- 惰性计算减少不必要计算

### 3. 内存优化
- 及时释放大型DataFrame
- 使用生成器处理大数据集
- 避免重复创建大对象

## 部署指南

### 本地部署

#### Streamlit前端
```bash
streamlit run frontend-streamlit/streamlit_app.py --server.port 8501
```

#### API服务
```bash
uvicorn src.api.main:app --host 0.0.0.0 --port 8124 --reload
```

### Docker部署
```bash
# 构建镜像
docker build -t stock-valuation:latest .

# 运行容器
docker run -p 8124:8124 -p 8501:8501 \
  --env-file .env \
  stock-valuation:latest
```

### 生产部署注意事项
1. **环境变量**: 使用Secret管理敏感信息
2. **数据库连接**: 配置连接池和超时
3. **API限流**: 配置TuShare API调用频率限制
4. **监控**: 设置健康检查和性能监控
5. **备份**: 定期备份配置和重要数据

## 故障排查

### 常见问题

#### 1. TuShare API调用失败
```bash
# 检查token配置
echo $TUSHARE_TOKEN

# 检查网络连接
ping api.tushare.pro

# 查看API调用日志
grep "tushare" logs/app.log
```

#### 2. 数据清洗异常
```bash
# 检查数据质量
python scripts/data_quality_check.py

# 查看清洗警告
grep "警告" logs/processor.log
```

#### 3. DCF计算错误
```bash
# 运行单元测试
pytest tests/test_dcf_calculator.py -v

# 检查输入数据
python scripts/debug_dcf_inputs.py --ts_code 600519.SH
```

### 调试工具
1. **E2E对比脚本**: `scripts/e2e_compare_sources.py`
2. **数据探测脚本**: `scripts/tushare_alignment_probe.py`
3. **性能分析**: `cProfile` + `snakeviz`

## 贡献指南

### 代码贡献流程
1. Fork项目到个人仓库
2. 创建功能分支
3. 实现功能并添加测试
4. 确保所有测试通过
5. 提交Pull Request
6. 代码审查与合并

### 文档贡献
- 更新相关的dev_docs文档
- 确保代码注释完整
- 添加示例代码
- 更新CLAUDE.md如有架构变更

### 问题报告
使用GitHub Issues报告问题，包含：
- 问题描述
- 重现步骤
- 环境信息
- 期望行为
- 实际行为

## 学习资源

### 项目相关
- DCF估值模型理论
- 财务报表分析
- Python异步编程
- FastAPI框架
- Streamlit框架

### 推荐阅读
- 《估值：难点、解决方案及相关案例》
- 《财务报表分析与证券估值》
- 《Clean Code》(代码规范)
- 《Effective Python》(Python最佳实践)

## 联系方式

- **技术问题**: 创建GitHub Issue
- **功能建议**: 通过Pull Request提交
- **紧急问题**: 联系项目维护者

---

*本指南会随着项目发展持续更新，欢迎提供反馈和改进建议。*