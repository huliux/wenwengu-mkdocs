# 部署运维指南
*最后更新: 2025-09-26*

## 概述

本指南提供股票估值系统的完整部署、运维和监控方案。项目已达到生产级标准，支持多种部署方式和运维策略。

### v3.0.0重要更新(2025-09-26)
- **用户认证系统**: 数据库架构从SQLite迁移至PostgreSQL/Supabase，支持高并发生产环境
- **智能缓存系统**: 估值结果持久化缓存，大幅提升用户体验和系统性能
- **会话管理**: 用户登录状态和数据会话恢复功能
- **安全增强**: 认证中间件、密码哈希、会话令牌等完整安全体系
- **UI/UX优化**: 前端界面和用户体验重大改进

## 部署架构

### 系统组件
```
┌─────────────────────────────────────────────────────┐
│                  Load Balancer                      │
├─────────────────────────────────────────────────────┤
│              Frontend Layer                         │
│  ┌─────────────────────────────────────────────────┐│
│  │         Streamlit App                           ││
│  │         Port: 8501                              ││
│  └─────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────┤
│              API Layer                              │
│  ┌─────────────────────────────────────────────────┐│
│  │         FastAPI Service                         ││
│  │         Port: 8124                              ││
│  └─────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────┤
│              Data Layer                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐│
│  │ TuShare API │ │ PostgreSQL  │ │ Redis (Cache)   ││
│  │ (External)  │ │ Port: 5432  │ │ Port: 6379      ││
│  └─────────────┘ └─────────────┘ └─────────────────┘│
└─────────────────────────────────────────────────────┘
```

## 环境要求

### 硬件要求
**生产环境**:
- CPU: 4核心以上
- 内存: 8GB以上
- 磁盘: 50GB SSD
- 网络: 100Mbps带宽

**开发环境**:
- CPU: 2核心以上
- 内存: 4GB以上
- 磁盘: 20GB可用空间

### 软件依赖
- Python 3.9+
- uv包管理器 (推荐) 或 pip
- PostgreSQL 12+ (可选)
- Redis 6+ (可选)
- Nginx (生产环境)
- Docker & Docker Compose (容器化部署)

## 部署方式

### 1. 本地开发部署

#### 快速启动
```bash
# 1. 环境准备
git clone <repository-url>
cd stock_vale_valuation_3.0
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

# 2. 配置环境
cp .env.example .env
# 编辑.env文件配置必要参数

# 3. 启动服务
# 启动API服务
uvicorn src.api.main:app --reload --port 8124 &

# 启动Streamlit前端
streamlit run frontend-streamlit/streamlit_app.py --server.port 8501 &
```

#### 服务验证
```bash
# 检查API服务
curl http://localhost:8124/

# 检查Streamlit前端
curl http://localhost:8501/

# 运行健康检查
curl http://localhost:8124/health
```

### 2. Docker容器化部署

#### Dockerfile示例
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# 安装uv包管理器
RUN pip install uv

# 复制项目文件
COPY . .

# 安装Python依赖
RUN uv pip install --system -e ".[prod]"

# 暴露端口
EXPOSE 8124 8501

# 启动脚本
CMD ["bash", "scripts/start.sh"]
```

#### Docker Compose配置
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8124:8124"
    environment:
      - DATA_SOURCE=tushare
      - TUSHARE_TOKEN=${TUSHARE_TOKEN}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
    depends_on:
      - redis
    restart: unless-stopped

  streamlit:
    build: .
    ports:
      - "8501:8501"
    environment:
      - API_BASE_URL=http://api:8124
    depends_on:
      - api
    restart: unless-stopped
    command: ["streamlit", "run", "frontend-streamlit/streamlit_app.py", "--server.port", "8501", "--server.address", "0.0.0.0"]

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    restart: unless-stopped

  postgres:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=stock_valuation
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api
      - streamlit
    restart: unless-stopped

volumes:
  postgres_data:
```

#### 启动容器服务
```bash
# 构建并启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f api

# 停止服务
docker-compose down
```

### 3. Kubernetes部署

#### 命名空间和配置
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: stock-valuation

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: stock-valuation
data:
  DATA_SOURCE: "tushare"
  OCL_AGGREGATION_MODE: "standard"
  API_BASE_URL: "http://api-service:8124"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: stock-valuation
type: Opaque
data:
  TUSHARE_TOKEN: <base64-encoded-token>
  DEEPSEEK_API_KEY: <base64-encoded-key>
```

#### 应用部署
```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: stock-valuation
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: stock-valuation:latest
        ports:
        - containerPort: 8124
        env:
        - name: DATA_SOURCE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATA_SOURCE
        - name: TUSHARE_TOKEN
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: TUSHARE_TOKEN
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8124
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8124
          initialDelaySeconds: 5
          periodSeconds: 5

---
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: stock-valuation
spec:
  selector:
    app: api
  ports:
  - protocol: TCP
    port: 8124
    targetPort: 8124
  type: ClusterIP
```

### 4. 云服务部署

#### AWS部署
```bash
# 使用AWS ECS
aws ecs create-cluster --cluster-name stock-valuation

# 创建任务定义
aws ecs register-task-definition --cli-input-json file://task-definition.json

# 创建服务
aws ecs create-service \
  --cluster stock-valuation \
  --service-name api-service \
  --task-definition stock-valuation-api:1 \
  --desired-count 2
```

#### Azure部署
```bash
# 创建容器实例
az container create \
  --resource-group myResourceGroup \
  --name stock-valuation-api \
  --image stock-valuation:latest \
  --ports 8124 \
  --environment-variables DATA_SOURCE=tushare
```

## 配置管理

### 环境变量配置

#### 核心配置
```bash
# 数据源配置
DATA_SOURCE=tushare  # tushare/postgres/hybrid
TUSHARE_TOKEN=your_token
OCL_AGGREGATION_MODE=standard

# LLM配置
LLM_PROVIDER=deepseek
DEEPSEEK_API_KEY=your_key
DEEPSEEK_MODEL_NAME=deepseek-chat

# 数据库配置(可选)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=stock_valuation
DB_USER=admin
DB_PASSWORD=password

# Redis配置(可选)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# API配置
API_BASE_URL=http://localhost:8124
CORS_ORIGINS=["http://localhost:8501"]

# 日志配置
LOG_LEVEL=INFO
LOG_FILE=logs/app.log
```

#### 环境特定配置
```bash
# 开发环境
cp .env.example .env.dev
# 编辑开发配置

# 测试环境
cp .env.example .env.test
# 编辑测试配置

# 生产环境
cp .env.example .env.prod
# 编辑生产配置
```

### 配置验证
```bash
# 验证配置完整性
python scripts/validate_config.py

# 测试数据源连接
python scripts/test_data_source.py

# 验证API可达性
python scripts/health_check.py
```

## 监控和日志

### 健康检查

#### API健康检查
```python
# 自定义健康检查端点
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "version": "3.0.0",
        "data_source": os.getenv("DATA_SOURCE"),
        "dependencies": {
            "tushare": check_tushare_connection(),
            "database": check_database_connection(),
            "redis": check_redis_connection()
        }
    }
```

#### 基础监控脚本
```bash
#!/bin/bash
# health_monitor.sh

API_URL="http://localhost:8124"
STREAMLIT_URL="http://localhost:8501"

# 检查API服务
api_status=$(curl -s -o /dev/null -w "%{http_code}" $API_URL/health)
if [ $api_status -eq 200 ]; then
    echo "✅ API service is healthy"
else
    echo "❌ API service is down (HTTP: $api_status)"
fi

# 检查Streamlit服务
streamlit_status=$(curl -s -o /dev/null -w "%{http_code}" $STREAMLIT_URL)
if [ $streamlit_status -eq 200 ]; then
    echo "✅ Streamlit service is healthy"
else
    echo "❌ Streamlit service is down (HTTP: $streamlit_status)"
fi
```

### 日志管理

#### 日志配置
```python
import logging
import logging.handlers

# 配置结构化日志
def setup_logging():
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )

    # 文件日志
    file_handler = logging.handlers.RotatingFileHandler(
        'logs/app.log',
        maxBytes=10*1024*1024,  # 10MB
        backupCount=5
    )
    file_handler.setFormatter(formatter)

    # 控制台日志
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(formatter)

    # 根日志器
    root_logger = logging.getLogger()
    root_logger.setLevel(logging.INFO)
    root_logger.addHandler(file_handler)
    root_logger.addHandler(console_handler)
```

#### 日志聚合(ELK Stack)
```yaml
# logstash.conf
input {
  file {
    path => "/app/logs/*.log"
    start_position => "beginning"
  }
}

filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} - %{DATA:logger} - %{LOGLEVEL:level} - %{GREEDYDATA:message}"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "stock-valuation-%{+YYYY.MM.dd}"
  }
}
```

### 性能监控

#### 关键指标
```python
# 使用Prometheus监控
from prometheus_client import Counter, Histogram, generate_latest

# 请求计数器
REQUEST_COUNT = Counter('api_requests_total', 'Total API requests', ['method', 'endpoint'])

# 响应时间直方图
REQUEST_LATENCY = Histogram('api_request_duration_seconds', 'API request latency')

# 业务指标
VALUATION_SUCCESS = Counter('valuations_successful_total', 'Successful valuations')
VALUATION_ERRORS = Counter('valuations_failed_total', 'Failed valuations', ['error_type'])

@app.middleware("http")
async def monitor_requests(request: Request, call_next):
    start_time = time.time()

    response = await call_next(request)

    # 记录指标
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path
    ).inc()

    REQUEST_LATENCY.observe(time.time() - start_time)

    return response
```

#### Grafana仪表板
```json
{
  "dashboard": {
    "title": "Stock Valuation System",
    "panels": [
      {
        "title": "API Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(api_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, api_request_duration_seconds_bucket)",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "singlestat",
        "targets": [
          {
            "expr": "rate(valuations_failed_total[5m]) / rate(api_requests_total[5m])"
          }
        ]
      }
    ]
  }
}
```

## 备份和恢复

### 数据备份策略

#### 配置文件备份
```bash
#!/bin/bash
# backup_config.sh

BACKUP_DIR="/backup/config"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份目录
mkdir -p $BACKUP_DIR/$DATE

# 备份环境配置
cp .env* $BACKUP_DIR/$DATE/
cp -r config/ $BACKUP_DIR/$DATE/
cp docker-compose.yml $BACKUP_DIR/$DATE/

# 压缩备份
tar -czf $BACKUP_DIR/config_backup_$DATE.tar.gz -C $BACKUP_DIR $DATE

# 清理旧备份(保留30天)
find $BACKUP_DIR -name "config_backup_*.tar.gz" -mtime +30 -delete
```

#### 数据库备份
```bash
#!/bin/bash
# backup_database.sh

DB_HOST="localhost"
DB_NAME="stock_valuation"
DB_USER="admin"
BACKUP_DIR="/backup/database"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME \
  | gzip > $BACKUP_DIR/db_backup_$DATE.sql.gz

# 上传到云存储(可选)
aws s3 cp $BACKUP_DIR/db_backup_$DATE.sql.gz \
  s3://your-backup-bucket/database/
```

### 恢复流程

#### 配置恢复
```bash
# 恢复配置文件
tar -xzf config_backup_20251222_140000.tar.gz
cp 20251222_140000/.env* ./
cp -r 20251222_140000/config/ ./
```

#### 数据库恢复
```bash
# 恢复数据库
gunzip -c db_backup_20251222_140000.sql.gz | \
  psql -h localhost -U admin -d stock_valuation
```

#### 服务恢复
```bash
# 重启服务
docker-compose down
docker-compose up -d

# 验证恢复
./scripts/health_check.sh
```

## 安全配置

### SSL/TLS配置

#### Nginx SSL配置
```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    location /api/ {
        proxy_pass http://api:8124/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://streamlit:8501/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 访问控制

#### API密钥认证
```python
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def verify_api_key(token: str = Depends(security)):
    if token.credentials != os.getenv("API_SECRET_KEY"):
        raise HTTPException(
            status_code=401,
            detail="Invalid API key"
        )
    return token
```

#### IP白名单
```python
from fastapi import Request, HTTPException

ALLOWED_IPS = ["127.0.0.1", "10.0.0.0/8", "192.168.0.0/16"]

async def ip_whitelist(request: Request):
    client_ip = request.client.host
    if not any(ipaddress.ip_address(client_ip) in ipaddress.ip_network(allowed)
              for allowed in ALLOWED_IPS):
        raise HTTPException(status_code=403, detail="IP not allowed")
```

## 故障排查

### 常见问题

#### 1. API服务无法启动
```bash
# 检查端口占用
lsof -i :8124

# 检查日志
tail -f logs/app.log

# 检查配置
python scripts/validate_config.py

# 重启服务
systemctl restart stock-valuation-api
```

#### 2. 数据源连接失败
```bash
# 检查TuShare连接
python -c "
import tushare as ts
ts.set_token('$TUSHARE_TOKEN')
pro = ts.pro_api()
print(pro.stock_basic().head())
"

# 检查数据库连接
pg_isready -h $DB_HOST -p $DB_PORT

# 检查网络连接
ping api.tushare.pro
```

#### 3. 内存使用过高
```bash
# 检查进程内存使用
ps aux | grep python | grep valuation

# 检查系统内存
free -h

# 重启服务释放内存
docker-compose restart api
```

#### 4. 估值计算错误
```bash
# 运行诊断脚本
python scripts/diagnose_valuation.py --ts_code 600519.SH

# 检查输入数据质量
python scripts/data_quality_check.py

# 对比数据源结果
python scripts/e2e_compare_sources.py --ts 600519.SH
```

### 日志分析

#### 错误模式识别
```bash
# 查找API错误
grep "ERROR" logs/app.log | tail -20

# 查找数据获取失败
grep "Failed to fetch" logs/app.log

# 查找计算异常
grep "Calculation error" logs/app.log

# 统计错误类型
grep "ERROR" logs/app.log | awk '{print $5}' | sort | uniq -c
```

## 扩展和优化

### 水平扩展

#### 负载均衡配置
```nginx
upstream api_backend {
    server api1:8124;
    server api2:8124;
    server api3:8124;
}

server {
    location /api/ {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 自动扩展(Kubernetes)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: stock-valuation
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 性能优化

#### 缓存策略
```python
import redis
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_result(expiration=3600):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 生成缓存键
            cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"

            # 尝试从缓存获取
            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)

            # 执行函数并缓存结果
            result = await func(*args, **kwargs)
            redis_client.setex(
                cache_key,
                expiration,
                json.dumps(result, default=str)
            )

            return result
        return wrapper
    return decorator
```

#### 数据库优化
```sql
-- 添加索引优化查询
CREATE INDEX idx_stock_basic_ts_code ON stock_basic(ts_code);
CREATE INDEX idx_income_end_date ON income_statement(end_date);
CREATE INDEX idx_balance_end_date ON balance_sheet(end_date);

-- 分区表优化大数据量
CREATE TABLE income_statement_2024 PARTITION OF income_statement
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

## 维护计划

### 日常维护

#### 每日任务
- 检查服务健康状态
- 监控API响应时间和错误率
- 检查磁盘空间使用情况
- 备份重要配置文件

#### 每周任务
- 清理过期日志文件
- 更新系统补丁
- 检查数据质量
- 性能报告生成

#### 每月任务
- 完整备份验证
- 安全审计
- 依赖包更新
- 容量规划评估

### 更新流程

#### 滚动更新
```bash
# 1. 构建新版本镜像
docker build -t stock-valuation:v3.1.0 .

# 2. 更新docker-compose.yml
sed -i 's/stock-valuation:latest/stock-valuation:v3.1.0/g' docker-compose.yml

# 3. 滚动更新服务
docker-compose up -d --no-deps api

# 4. 验证更新
./scripts/health_check.sh

# 5. 更新其他服务
docker-compose up -d
```

#### 回滚计划
```bash
# 如果更新失败，快速回滚
docker-compose down
git checkout previous-stable-tag
docker-compose up -d

# 恢复数据(如需要)
./scripts/restore_backup.sh
```

---

*本部署运维指南提供了完整的生产级部署方案，建议根据实际环境调整具体配置。*