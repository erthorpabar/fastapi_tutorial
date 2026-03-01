# 第十一章：部署

## 11.1 部署概述

### 部署方式

- **Docker**: 容器化部署
- **云平台**: AWS, GCP, Azure
- **PaaS**: Heroku, Railway, Render
- **传统服务器**: VPS, 专用服务器

## 11.2 生产环境配置

### 环境变量

创建 `.env` 文件：

```bash
# 应用配置
APP_NAME=My API
APP_VERSION=1.0.0
DEBUG=False
SECRET_KEY=your-secret-key-change-in-production

# 数据库配置
DATABASE_URL=postgresql://user:password@localhost/dbname

# Redis 配置
REDIS_URL=redis://localhost:6379

# JWT 配置
JWT_SECRET_KEY=your-jwt-secret-key
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=30

# CORS 配置
ALLOWED_ORIGINS=http://localhost:3000,https://myapp.com

# 服务器配置
HOST=0.0.0.0
PORT=8000
```

### 配置类

```python
# config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # 应用配置
    APP_NAME: str = "My API"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False
    SECRET_KEY: str
    
    # 数据库配置
    DATABASE_URL: str
    
    # JWT 配置
    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRATION_MINUTES: int = 30
    
    # CORS 配置
    ALLOWED_ORIGINS: str
    
    # 服务器配置
    HOST: str = "0.0.0.0"
    PORT: int = 8000
    
    @property
    def allowed_origins_list(self) -> List[str]:
        return self.ALLOWED_ORIGINS.split(",")
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### 使用配置

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from config import settings

app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    debug=settings.DEBUG
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## 11.3 Docker 部署

### 创建 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 设置环境变量
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 创建非 root 用户
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
      - redis
    volumes:
      - ./app:/app
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### 多阶段构建 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11 as builder

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# 最终镜像
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache-dir /wheels/*

# 复制应用
COPY . .

# 非root用户
RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### .dockerignore

```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv/
pip-log.txt
pip-delete-this-directory.txt
.pytest_cache/
.coverage
htmlcov/
.git/
.gitignore
.env
*.db
*.sqlite3
.DS_Store
```

## 11.4 Gunicorn + Uvicorn

### 安装

```bash
pip install gunicorn uvicorn[standard]
```

### 运行命令

```bash
# 开发环境
gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --reload

# 生产环境
gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# 使用配置文件
gunicorn main:app -c gunicorn.conf.py
```

### gunicorn.conf.py

```python
# gunicorn.conf.py
import multiprocessing

# 绑定地址
bind = "0.0.0.0:8000"

# Worker 配置
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"

# 超时设置
timeout = 120
keepalive = 5

# 日志配置
accesslog = "-"
errorlog = "-"
loglevel = "info"

# 进程配置
daemon = False
pidfile = None
user = None
group = None
tmp_upload_dir = None

# 优雅重启
graceful_timeout = 30
max_requests = 1000
max_requests_jitter = 50

# 钩子函数
def on_starting(server):
    print("Starting Gunicorn")

def on_exit(server):
    print("Shutting down Gunicorn")

def when_ready(server):
    print("Server is ready. Spawning workers")
```

## 11.5 Nginx 反向代理

### Nginx 配置

```nginx
# /etc/nginx/sites-available/myapi
upstream fastapi_backend {
    server 127.0.0.1:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name myapi.com www.myapi.com;
    
    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapi.com www.myapi.com;
    
    # SSL 配置
    ssl_certificate /etc/letsencrypt/live/myapi.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapi.com/privkey.pem;
    
    # SSL 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # 请求限制
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    # 代理配置
    location / {
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 支持
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 缓冲
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # 静态文件
    location /static/ {
        alias /var/www/myapi/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # 健康检查
    location /health {
        access_log off;
        proxy_pass http://fastapi_backend/health;
    }
    
    # 日志
    access_log /var/log/nginx/myapi_access.log;
    error_log /var/log/nginx/myapi_error.log;
}
```

## 11.6 Systemd 服务

### 创建服务文件

```bash
sudo nano /etc/systemd/system/fastapi.service
```

```ini
[Unit]
Description=FastAPI Application
After=network.target

[Service]
Type=notify
User=www-data
Group=www-data
RuntimeDirectory=fastapi
RuntimeDirectoryMode=0755

# 环境变量
Environment="PATH=/var/www/myapi/venv/bin"
EnvironmentFile=/var/www/myapi/.env

# 工作目录
WorkingDirectory=/var/www/myapi

# 启动命令
ExecStart=/var/www/myapi/venv/bin/gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --access-logfile - \
    --error-logfile - \
    --log-level info

# 重启策略
Restart=always
RestartSec=10

# 安全设置
NoNewPrivileges=true
PrivateTmp=true

# 标准输出
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=fastapi

[Install]
WantedBy=multi-user.target
```

### 管理服务

```bash
# 启动服务
sudo systemctl start fastapi

# 停止服务
sudo systemctl stop fastapi

# 重启服务
sudo systemctl restart fastapi

# 查看状态
sudo systemctl status fastapi

# 开机自启
sudo systemctl enable fastapi

# 查看日志
sudo journalctl -u fastapi -f
```

## 11.7 云平台部署

### Heroku 部署

创建 `Procfile`:

```
web: gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT
```

部署步骤：

```bash
# 安装 Heroku CLI
# 登录
heroku login

# 创建应用
heroku create my-fastapi-app

# 添加数据库
heroku addons:create heroku-postgresql:hobby-dev

# 设置环境变量
heroku config:set SECRET_KEY=your-secret-key

# 部署
git push heroku main

# 运行迁移
heroku run alembic upgrade head
```

### Railway 部署

创建 `railway.toml`:

```toml
[build]
builder = "DOCKERFILE"

[deploy]
startCommand = "gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT"
healthcheckPath = "/health"
healthcheckTimeout = 100
restartPolicyType = "ON_FAILURE"
```

### AWS Elastic Beanstalk

创建 `.ebextensions/python.config`:

```yaml
option_settings:
  aws:elasticbeanstalk:container:python:
    WSGIPath: main:app
    NumProcesses: 4
    NumThreads: 15
  aws:elasticbeanstalk:application:environment:
    DEBUG: "False"
```

## 11.8 监控和日志

### 日志配置

```python
# logging_config.py
import logging
import sys
from pathlib import Path

LOG_FORMAT = "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
LOG_DIR = Path("logs")

def setup_logging(log_level: str = "INFO"):
    LOG_DIR.mkdir(exist_ok=True)
    
    # 根日志配置
    logging.basicConfig(
        level=getattr(logging, log_level),
        format=LOG_FORMAT,
        handlers=[
            # 控制台输出
            logging.StreamHandler(sys.stdout),
            # 文件输出
            logging.FileHandler(LOG_DIR / "app.log"),
        ]
    )
    
    # 设置第三方库日志级别
    logging.getLogger("uvicorn").setLevel(logging.INFO)
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)

# 使用
setup_logging("INFO")
logger = logging.getLogger(__name__)
```

### 健康检查端点

```python
from fastapi import FastAPI
from datetime import datetime
import psutil

app = FastAPI()

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "cpu_percent": psutil.cpu_percent(),
        "memory_percent": psutil.virtual_memory().percent,
    }

@app.get("/ready")
async def readiness_check():
    # 检查数据库连接
    # 检查 Redis 连接
    # 检查其他依赖
    return {"status": "ready"}
```

### Prometheus 监控

```python
from prometheus_client import Counter, Histogram, generate_latest
from fastapi import FastAPI, Response

app = FastAPI()

# 指标
REQUEST_COUNT = Counter(
    'request_count',
    'Total request count',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'request_latency_seconds',
    'Request latency',
    ['method', 'endpoint']
)

@app.middleware("http")
async def add_metrics(request, call_next):
    start_time = time.time()
    response = await call_next(request)
    
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(time.time() - start_time)
    
    return response

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type="text/plain"
    )
```

## 11.9 安全配置

### HTTPS 配置

```python
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# 强制 HTTPS
app.add_middleware(HTTPSRedirectMiddleware)
```

### 安全头

```python
from fastapi import FastAPI
from starlette.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# 信任的域名
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["myapi.com", "*.myapi.com"]
)
```

### 速率限制

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter
import redis.asyncio as redis

app = FastAPI()

@app.on_event("startup")
async def startup():
    redis_client = redis.from_url("redis://localhost:6379")
    await FastAPILimiter.init(redis_client)

@app.get(
    "/limited",
    dependencies=[Depends(RateLimiter(times=2, seconds=5))]
)
async def limited_endpoint():
    return {"message": "This endpoint is rate limited"}
```

## 11.10 完整部署示例

### 项目结构

```
myapi/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── models/
│   ├── routers/
│   └── utils/
├── tests/
├── alembic/
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env
├── gunicorn.conf.py
└── .dockerignore
```

### requirements.txt

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
gunicorn==21.2.0
python-multipart==0.0.6
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
pydantic==2.5.0
pydantic-settings==2.1.0
sqlalchemy==2.0.23
asyncpg==0.29.0
alembic==1.12.1
redis==5.0.1
prometheus-client==0.19.0
psutil==5.9.6
```

### 生产环境启动脚本

```bash
#!/bin/bash
# start.sh

set -e

# 激活虚拟环境
source venv/bin/activate

# 运行数据库迁移
alembic upgrade head

# 启动应用
gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --access-logfile /var/log/myapi/access.log \
    --error-logfile /var/log/myapi/error.log \
    --log-level info \
    --preload
```

## 11.11 部署清单

- [ ] 环境变量配置完成
- [ ] 数据库迁移完成
- [ ] SSL 证书配置
- [ ] Nginx 反向代理配置
- [ ] Systemd 服务配置
- [ ] 日志收集配置
- [ ] 监控和告警配置
- [ ] 备份策略配置
- [ ] 安全头配置
- [ ] 速率限制配置
- [ ] 健康检查端点
- [ ] 文档更新

## 11.12 练习

1. **基础练习**
   - 使用 Docker 部署应用
   - 配置环境变量
   - 设置 Nginx 反向代理

2. **进阶练习**
   - 部署到云平台
   - 配置 HTTPS
   - 实现监控和日志

3. **挑战练习**
   - 配置 CI/CD 流水线
   - 实现零停机部署
   - 配置自动伸缩

## 11.13 小结

- 使用环境变量管理配置
- Docker 提供一致的开发和生产环境
- Gunicorn + Uvicorn 提供高性能生产服务器
- Nginx 作为反向代理提供安全和性能优化
- Systemd 管理服务自动启动和重启
- 监控和日志对生产环境至关重要

## 下一章

[第十二章：实战项目](./12_project.md) - 综合运用所学知识构建完整项目！
