# 第十四章：常见问题

## 14.1 安装和配置问题

### 问题：安装 FastAPI 时遇到依赖冲突

**解决方案：**

```bash
# 创建新的虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 升级 pip
pip install --upgrade pip

# 安装 FastAPI
pip install fastapi uvicorn[standard]
```

### 问题：ModuleNotFoundError: No module named 'fastapi'

**原因：** 虚拟环境未激活或未安装 FastAPI

**解决方案：**

```bash
# 检查虚拟环境
which python  # Linux/Mac
where python  # Windows

# 激活虚拟环境
source venv/bin/activate

# 确认安装
pip list | grep fastapi
```

### 问题：如何选择 Python 版本？

**建议：** 使用 Python 3.8 或更高版本

```bash
# 检查 Python 版本
python --version

# 推荐版本
Python 3.11.x
```

## 14.2 路由问题

### 问题：路由顺序导致的问题

**问题代码：**

```python
# ❌ 错误
@app.get("/users/me")
async def get_current_user():
    return {"user": "current"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

访问 `/users/me` 会返回 `{"user_id": "me"}` 而不是 `{"user": "current"}`。

**解决方案：**

```python
# ✅ 正确 - 特定路径在前
@app.get("/users/me")
async def get_current_user():
    return {"user": "current"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

### 问题：如何处理尾部斜杠？

**解决方案：**

```python
from fastapi import FastAPI

app = FastAPI()

# 两个路由都会匹配
@app.get("/items")  # 匹配 /items
@app.get("/items/")  # 匹配 /items/
async def read_items():
    return {"items": []}
```

### 问题：如何创建动态路由？

**解决方案：**

```python
from fastapi import FastAPI, Path

app = FastAPI()

# 路径参数验证
@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(..., ge=1, le=1000)
):
    return {"item_id": item_id}

# 路径包含斜杠
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
```

## 14.3 数据验证问题

### 问题：如何处理可选字段？

**解决方案：**

```python
from pydantic import BaseModel
from typing import Optional

class Item(BaseModel):
    name: str
    description: Optional[str] = None  # 可选，默认 None
    price: float
    tax: Optional[float] = None  # 可选，默认 None

# 或使用 Pydantic v2
class ItemV2(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
```

### 问题：如何自定义验证错误消息？

**解决方案：**

```python
from pydantic import BaseModel, validator, Field

class User(BaseModel):
    username: str = Field(..., min_length=3)
    age: int = Field(..., ge=0, le=120)
    
    @validator('username')
    def username_must_contain_letter(cls, v):
        if not any(c.isalpha() for c in v):
            raise ValueError('用户名必须包含至少一个字母')
        return v
    
    class Config:
        # 自定义错误消息
        error_msg_templates = {
            'value_error.missing': '此字段为必填项',
            'value_error.number.not_gt': '数值必须大于 {limit_value}',
        }
```

### 问题：如何验证嵌套模型？

**解决方案：**

```python
from pydantic import BaseModel
from typing import List

class Address(BaseModel):
    street: str
    city: str
    country: str

class User(BaseModel):
    name: str
    addresses: List[Address]

# 使用
user = User(
    name="John",
    addresses=[
        {"street": "123 Main St", "city": "New York", "country": "USA"}
    ]
)
```

## 14.4 数据库问题

### 问题：数据库会话未正确关闭

**原因：** 会话管理不当导致连接泄漏

**解决方案：**

```python
from fastapi import Depends
from sqlalchemy.orm import Session

# ✅ 正确 - 使用依赖注入
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
def read_items(db: Session = Depends(get_db)):
    items = db.query(Item).all()
    return items  # 会话会自动关闭
```

### 问题：如何处理数据库迁移？

**解决方案：**

```bash
# 1. 安装 Alembic
pip install alembic

# 2. 初始化
alembic init alembic

# 3. 配置 alembic/env.py
from models import Base
target_metadata = Base.metadata

# 4. 创建迁移
alembic revision --autogenerate -m "Initial migration"

# 5. 执行迁移
alembic upgrade head
```

### 问题：异步数据库如何使用？

**解决方案：**

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from fastapi import Depends

# 配置
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession)

# 依赖
async def get_async_db():
    async with AsyncSessionLocal() as session:
        yield session

# 使用
@app.get("/items/")
async def read_items(db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(select(Item))
    return result.scalars().all()
```

## 14.5 认证问题

### 问题：JWT token 过期如何处理？

**解决方案：**

```python
from fastapi import HTTPException, status
from jose import JWTError, jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError as e:
        if "expired" in str(e).lower():
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token has expired",
                headers={"WWW-Authenticate": "Bearer"},
            )
        raise credentials_exception
    
    return get_user(user_id)

# 客户端可以使用 refresh token 获取新的 access token
@app.post("/refresh")
async def refresh_token(refresh_token: str):
    # 验证 refresh token 并返回新的 access token
    pass
```

### 问题：如何实现权限控制？

**解决方案：**

```python
from functools import wraps

def require_roles(*roles):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            if current_user.role not in roles:
                raise HTTPException(
                    status_code=403,
                    detail="Not enough permissions"
                )
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator

# 使用
@app.delete("/admin/users/{user_id}")
@require_roles("admin")
async def delete_user(user_id: int, current_user: User):
    return {"message": "User deleted"}
```

### 问题：CORS 错误如何解决？

**解决方案：**

```python
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 配置 CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://myapp.com"
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## 14.6 性能问题

### 问题：API 响应慢如何优化？

**解决方案：**

1. **数据库查询优化：**

```python
# 使用 select_related 或 joinedload
from sqlalchemy.orm import joinedload

# ❌ 慢 - N+1 查询问题
posts = db.query(Post).all()
for post in posts:
    print(post.author.name)  # 每次都会查询数据库

# ✅ 快 - 使用 join
posts = db.query(Post).options(joinedload(Post.author)).all()
```

2. **使用缓存：**

```python
from fastapi_cache import FastAPICache
from fastapi_cache.decorator import cache

@app.get("/items/")
@cache(expire=60)  # 缓存 60 秒
async def read_items():
    return db.query(Item).all()
```

3. **分页：**

```python
@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    items = db.query(Item).offset(skip).limit(limit).all()
    return items
```

### 问题：如何处理大文件上传？

**解决方案：**

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    # 流式读取，不会占用太多内存
    async def chunk_generator():
        while chunk := await file.read(1024 * 1024):  # 1MB chunks
            yield chunk
    
    async for chunk in chunk_generator():
        # 处理每个块
        pass
    
    return {"filename": file.filename}
```

## 14.7 测试问题

### 问题：如何测试需要认证的端点？

**解决方案：**

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def get_auth_header():
    # 先登录获取 token
    response = client.post(
        "/token",
        data={"username": "testuser", "password": "password123"}
    )
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

def test_protected_route():
    response = client.get("/protected", headers=get_auth_header())
    assert response.status_code == 200
```

### 问题：测试数据库如何隔离？

**解决方案：**

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="function")
def db_session():
    # 创建内存数据库
    engine = create_engine("sqlite:///:memory:")
    
    # 创建表
    Base.metadata.create_all(engine)
    
    Session = sessionmaker(bind=engine)
    session = Session()
    
    yield session
    
    # 清理
    session.close()
    Base.metadata.drop_all(engine)
```

## 14.8 部署问题

### 问题：生产环境如何运行？

**解决方案：**

```bash
# 使用 Gunicorn + Uvicorn
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --access-logfile - \
  --error-logfile -
```

### 问题：如何配置 HTTPS？

**解决方案：**

1. **使用 Nginx 反向代理：**

```nginx
server {
    listen 443 ssl;
    server_name example.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

2. **在 FastAPI 中强制 HTTPS：**

```python
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()
app.add_middleware(HTTPSRedirectMiddleware)
```

### 问题：Docker 容器如何优化大小？

**解决方案：**

```dockerfile
# 多阶段构建
FROM python:3.11-slim as builder

WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

FROM python:3.11-slim

WORKDIR /app
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache-dir /wheels/*

COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 14.9 其他常见问题

### 问题：如何在后台运行任务？

**解决方案：**

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, message: str):
    # 发送邮件逻辑
    pass

@app.post("/send-notification")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    # 在后台执行
    background_tasks.add_task(send_email, email, "Hello!")
    return {"message": "Notification sent"}
```

### 问题：如何使用 WebSocket？

**解决方案：**

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Message: {data}")
```

### 问题：如何处理时区？

**解决方案：**

```python
from datetime import datetime, timezone

# 使用 UTC 时间
created_at = datetime.now(timezone.utc)

# 转换为本地时间
from zoneinfo import ZoneInfo
local_time = created_at.astimezone(ZoneInfo("Asia/Shanghai"))
```

### 问题：如何记录请求日志？

**解决方案：**

```python
import logging
from fastapi import Request

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    logger.info(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Response status: {response.status_code}")
    return response
```

### 问题：如何限制请求大小？

**解决方案：**

```python
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.middleware("http")
async def limit_upload_size(request: Request, call_next):
    if request.method == "POST":
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > 10 * 1024 * 1024:  # 10MB
            raise HTTPException(status_code=413, detail="File too large")
    return await call_next(request)
```

## 14.10 调试技巧

### 使用日志调试

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    logger.debug(f"Fetching item {item_id}")
    item = get_item_from_db(item_id)
    logger.debug(f"Item: {item}")
    return item
```

### 使用调试器

```python
# 在代码中设置断点
import pdb; pdb.set_trace()

# 或使用 ipdb（更好的体验）
import ipdb; ipdb.set_trace()

# VS Code launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "python",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "main:app",
                "--reload"
            ],
            "jinja": true
        }
    ]
}
```

## 14.11 社区资源

### 官方资源

- **FastAPI 官方文档**: https://fastapi.tiangolo.com/
- **GitHub 仓库**: https://github.com/tiangolo/fastapi
- **FastAPI Discord**: https://discord.gg/fastapi

### 第三方资源

- **TestDriven.io**: FastAPI 教程
- **Real Python**: FastAPI 文章
- **YouTube**: FastAPI 视频教程

### 常用库

```bash
# 数据库
pip install sqlalchemy databases asyncpg

# 认证
pip install python-jose passlib python-multipart

# 测试
pip install pytest httpx pytest-asyncio

# 工具
pip install fastapi-utils fastapi-cache fastapi-limiter
```

## 14.12 总结

本章涵盖了 FastAPI 开发中的常见问题和解决方案：

- 安装和配置
- 路由问题
- 数据验证
- 数据库操作
- 认证和授权
- 性能优化
- 测试
- 部署
- 调试技巧

遇到问题时：

1. 查阅官方文档
2. 搜索 GitHub Issues
3. 在社区提问
4. 查看错误日志
5. 使用调试工具

## 结语

恭喜你完成了 FastAPI 完整教学手册的学习！

你现在已经掌握了：

- ✅ FastAPI 基础知识
- ✅ 路由和请求处理
- ✅ 数据验证和模型
- ✅ 依赖注入
- ✅ 数据库集成
- ✅ 认证和授权
- ✅ 错误处理
- ✅ 测试
- ✅ 部署
- ✅ 最佳实践
- ✅ 常见问题解决

继续学习：

- 深入学习异步编程
- 探索微服务架构
- 学习容器化和编排
- 了解 CI/CD 流程
- 参与开源项目

祝你使用 FastAPI 开发出优秀的应用！🚀
