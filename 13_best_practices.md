# 第十三章：最佳实践

## 13.1 项目结构

### 推荐的项目结构

```
project/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI 应用入口
│   ├── config.py            # 配置管理
│   ├── dependencies.py      # 全局依赖
│   ├── models/              # 数据库模型
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── schemas/             # Pydantic 模型
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── routers/             # 路由模块
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── items.py
│   ├── services/            # 业务逻辑
│   │   ├── __init__.py
│   │   └── user_service.py
│   ├── repositories/        # 数据访问层
│   │   ├── __init__.py
│   │   └── user_repo.py
│   ├── core/                # 核心功能
│   │   ├── __init__.py
│   │   ├── security.py
│   │   └── exceptions.py
│   └── utils/               # 工具函数
│       ├── __init__.py
│       └── helpers.py
├── tests/                   # 测试
│   ├── __init__.py
│   ├── conftest.py
│   └── test_main.py
├── alembic/                 # 数据库迁移
├── .env                     # 环境变量
├── .gitignore
├── requirements.txt
├── Dockerfile
└── README.md
```

### 分层架构

```python
# 分层架构示例

# 1. Router 层（处理 HTTP 请求）
# app/routers/users.py
from fastapi import APIRouter, Depends
from app.schemas.user import UserCreate, User
from app.services.user_service import UserService
from app.dependencies import get_user_service

router = APIRouter()

@router.post("/users", response_model=User)
def create_user(
    user_data: UserCreate,
    service: UserService = Depends(get_user_service)
):
    return service.create_user(user_data)

# 2. Service 层（业务逻辑）
# app/services/user_service.py
class UserService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo
    
    def create_user(self, user_data: UserCreate) -> User:
        # 业务逻辑验证
        if self.user_repo.exists_by_email(user_data.email):
            raise ValueError("Email already exists")
        
        # 创建用户
        user = self.user_repo.create(user_data)
        return user

# 3. Repository 层（数据访问）
# app/repositories/user_repo.py
class UserRepository:
    def __init__(self, db: Session):
        self.db = db
    
    def create(self, user_data: UserCreate) -> User:
        user = User(**user_data.dict())
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user
    
    def exists_by_email(self, email: str) -> bool:
        return self.db.query(User).filter(User.email == email).first() is not None
```

## 13.2 配置管理

### 使用环境变量

```python
# app/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # 应用配置
    APP_NAME: str = "My API"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False
    
    # 数据库配置
    DATABASE_URL: str
    
    # 安全配置
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    
    # CORS 配置
    ALLOWED_ORIGINS: str = "*"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    
    class Config:
        env_file = ".env"
        case_sensitive = True
    
    @property
    def allowed_origins_list(self) -> List[str]:
        if self.ALLOWED_ORIGINS == "*":
            return ["*"]
        return self.ALLOWED_ORIGINS.split(",")

settings = Settings()
```

### 环境变量文件

```bash
# .env.example
APP_NAME=My API
APP_VERSION=1.0.0
DEBUG=False

DATABASE_URL=postgresql://user:password@localhost/dbname
SECRET_KEY=your-secret-key-change-in-production
ACCESS_TOKEN_EXPIRE_MINUTES=30

ALLOWED_ORIGINS=http://localhost:3000,https://myapp.com
REDIS_URL=redis://localhost:6379
```

## 13.3 数据库最佳实践

### 使用连接池

```python
# app/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from app.config import settings

engine = create_engine(
    settings.DATABASE_URL,
    pool_size=10,          # 连接池大小
    max_overflow=20,       # 最大溢出连接数
    pool_pre_ping=True,    # 连接健康检查
    pool_recycle=3600,     # 连接回收时间（秒）
    echo=settings.DEBUG    # 打印 SQL 语句
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

### 数据库会话管理

```python
# app/dependencies.py
from sqlalchemy.orm import Session
from app.database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 异步数据库

```python
# app/database_async.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    pool_size=10,
    max_overflow=20
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_async_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

## 13.4 安全最佳实践

### 密码哈希

```python
# app/core/security.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)
```

### JWT 配置

```python
# app/core/jwt.py
from datetime import datetime, timedelta
from jose import JWTError, jwt
from app.config import settings

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm="HS256"
    )
    return encoded_jwt

def decode_token(token: str):
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=["HS256"]
        )
        return payload
    except JWTError:
        return None
```

### 输入验证

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, Field, validator
import re

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=100)
    
    @validator('username')
    def username_alphanumeric(cls, v):
        if not re.match(r'^[a-zA-Z0-9_-]+$', v):
            raise ValueError('Username must be alphanumeric')
        return v
    
    @validator('password')
    def password_strength(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'[0-9]', v):
            raise ValueError('Password must contain number')
        return v
```

### SQL 注入防护

```python
# ❌ 不安全
query = f"SELECT * FROM users WHERE username = '{username}'"

# ✅ 安全（使用 ORM）
user = db.query(User).filter(User.username == username).first()

# ✅ 安全（使用参数化查询）
from sqlalchemy import text
query = text("SELECT * FROM users WHERE username = :username")
result = db.execute(query, {"username": username})
```

## 13.5 错误处理最佳实践

### 自定义异常

```python
# app/core/exceptions.py
from fastapi import HTTPException, status

class AppException(Exception):
    def __init__(
        self,
        status_code: int,
        code: str,
        message: str,
        details: dict = None
    ):
        self.status_code = status_code
        self.code = code
        self.message = message
        self.details = details

class NotFoundException(AppException):
    def __init__(self, resource: str, resource_id: int):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            code="NOT_FOUND",
            message=f"{resource} not found",
            details={"resource": resource, "id": resource_id}
        )

class ValidationException(AppException):
    def __init__(self, message: str, field: str = None):
        super().__init__(
            status_code=status.HTTP_400_BAD_REQUEST,
            code="VALIDATION_ERROR",
            message=message,
            details={"field": field}
        )

class UnauthorizedException(AppException):
    def __init__(self, message: str = "Unauthorized"):
        super().__init__(
            status_code=status.HTTP_401_UNAUTHORIZED,
            code="UNAUTHORIZED",
            message=message
        )
```

### 全局异常处理

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from app.core.exceptions import AppException

app = FastAPI()

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details
            }
        }
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # 记录错误日志
    logger.exception(f"Unhandled exception: {exc}")
    
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred"
            }
        }
    )
```

## 13.6 性能优化

### 数据库查询优化

```python
# 使用索引
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)  # 添加索引
    username = Column(String, unique=True, index=True)  # 添加索引

# 只查询需要的列
users = db.query(User.id, User.username).all()

# 使用 join 优化关联查询
from sqlalchemy.orm import joinedload
posts = db.query(Post).options(joinedload(Post.author)).all()

# 批量插入
users = [User(username=f"user{i}") for i in range(1000)]
db.bulk_save_objects(users)
db.commit()
```

### 缓存策略

```python
# app/utils/cache.py
import redis
import json
from functools import wraps
from app.config import settings

redis_client = redis.from_url(settings.REDIS_URL)

def cache(expire: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 生成缓存 key
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"
            
            # 尝试从缓存获取
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # 执行函数
            result = await func(*args, **kwargs)
            
            # 存入缓存
            redis_client.setex(
                cache_key,
                expire,
                json.dumps(result)
            )
            
            return result
        return wrapper
    return decorator

# 使用缓存
@cache(expire=300)
async def get_user(user_id: int):
    return db.query(User).filter(User.id == user_id).first()
```

### 异步操作

```python
# 使用异步 I/O
@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()

# 并发请求
import asyncio
import httpx

async def fetch_multiple_urls(urls: list):
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
    return responses
```

## 13.7 日志最佳实践

### 结构化日志

```python
# app/utils/logging.py
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }
        
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_data)

# 配置日志
def setup_logging():
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    logger.addHandler(handler)
    
    return logger
```

### 请求追踪

```python
# app/middleware/logging.py
import uuid
from fastapi import Request
import logging

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    request_id = str(uuid.uuid4())
    
    # 记录请求
    logger.info(
        f"Request started",
        extra={
            "request_id": request_id,
            "method": request.method,
            "url": str(request.url),
        }
    )
    
    # 处理请求
    response = await call_next(request)
    
    # 记录响应
    logger.info(
        f"Request completed",
        extra={
            "request_id": request_id,
            "status_code": response.status_code,
        }
    )
    
    response.headers["X-Request-ID"] = request_id
    return response
```

## 13.8 测试最佳实践

### 测试组织

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import Base, get_db

# 测试数据库
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="function")
def db_session():
    # 创建表
    Base.metadata.create_all(bind=engine)
    
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.close()
        # 删除表
        Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db_session):
    def override_get_db():
        try:
            yield db_session
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()

@pytest.fixture
def test_user(db_session):
    user = User(
        username="testuser",
        email="test@example.com",
        hashed_password=get_password_hash("password123")
    )
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user
```

### 参数化测试

```python
import pytest

@pytest.mark.parametrize("username,email,expected_status", [
    ("validuser", "valid@email.com", 201),
    ("", "valid@email.com", 422),  # 空用户名
    ("user", "invalid-email", 422),  # 无效邮箱
    ("a"*100, "test@email.com", 422),  # 用户名太长
])
def test_register_validation(client, username, email, expected_status):
    response = client.post(
        "/register",
        json={"username": username, "email": email, "password": "password123"}
    )
    assert response.status_code == expected_status
```

## 13.9 API 文档最佳实践

### 详细的文档字符串

```python
@app.post(
    "/users/",
    response_model=User,
    status_code=status.HTTP_201_CREATED,
    summary="创建新用户",
    description="创建一个新用户账户",
    response_description="创建的用户信息",
    responses={
        201: {
            "description": "用户创建成功",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "username": "johndoe",
                        "email": "john@example.com"
                    }
                }
            }
        },
        400: {
            "description": "用户名或邮箱已存在",
        }
    }
)
async def create_user(
    user: UserCreate,
    db: Session = Depends(get_db)
):
    """
    创建新用户
    
    参数:
    - **username**: 用户名（3-50字符）
    - **email**: 邮箱地址
    - **password**: 密码（至少8字符）
    
    返回:
    - 创建的用户对象
    """
    return UserService(db).create_user(user)
```

## 13.10 部署最佳实践

### 健康检查

```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": settings.APP_VERSION
    }

@app.get("/ready")
async def readiness_check(db: Session = Depends(get_db)):
    # 检查数据库连接
    try:
        db.execute("SELECT 1")
        db_status = "healthy"
    except Exception as e:
        db_status = f"unhealthy: {str(e)}"
    
    return {
        "status": "ready" if db_status == "healthy" else "not ready",
        "database": db_status
    }
```

### 优雅关闭

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动时执行
    print("Starting up...")
    yield
    # 关闭时执行
    print("Shutting down...")

app = FastAPI(lifespan=lifespan)
```

## 13.11 代码质量

### 使用 Linter

```bash
# 安装
pip install flake8 black mypy

# 运行
flake8 app/
black app/
mypy app/
```

### 配置文件

```ini
# .flake8
[flake8]
max-line-length = 88
exclude = 
    .git,
    __pycache__,
    migrations,
    venv

# mypy.ini
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_ignores = True
disallow_untyped_defs = True
```

## 13.12 练习

1. **基础练习**
   - 重构项目结构
   - 添加配置管理
   - 实现日志记录

2. **进阶练习**
   - 实现缓存策略
   - 添加监控和追踪
   - 优化数据库查询

3. **挑战练习**
   - 实现微服务架构
   - 添加性能监控
   - 实现自动化测试和部署

## 13.13 小结

遵循最佳实践可以：

- 提高代码质量和可维护性
- 增强应用安全性
- 优化性能
- 便于团队协作
- 简化部署和运维

## 下一章

[第十四章：常见问题](./14_faq.md) - 解答 FastAPI 开发中的常见问题！
