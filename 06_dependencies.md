# 第六章：依赖注入

## 6.1 依赖注入基础

### 什么是依赖注入？

依赖注入（Dependency Injection, DI）是一种设计模式，允许你将依赖项（如服务、配置等）注入到函数或类中，而不是在内部创建它们。

### 第一个依赖

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def common_parameters(q: str = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users/")
async def read_users(commons: dict = Depends(common_parameters)):
    return commons
```

## 6.2 类作为依赖

### 使用类

```python
from fastapi import FastAPI, Depends
from typing import Optional

app = FastAPI()

class CommonParams:
    def __init__(self, q: Optional[str] = None, skip: int = 0, limit: int = 100):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(commons: CommonParams = Depends(CommonParams)):
    response = {}
    if commons.q:
        response.update({"q": commons.q})
    items = fake_items_db[commons.skip : commons.skip + commons.limit]
    response.update({"items": items})
    return response
```

### 单例依赖

```python
class Database:
    def __init__(self):
        self.connection = None
    
    def connect(self):
        self.connection = "Connected"
        return self.connection

# 每次请求创建新实例
@app.get("/items/")
async def read_items(db: Database = Depends(Database)):
    return {"db": db.connect()}

# 单例（整个应用共享一个实例）
@app.get("/users/")
async def read_users(db: Database = Depends()):
    return {"db": db.connect()}

# 或者在应用启动时创建
db_instance = Database()

@app.get("/products/")
async def read_products(db: Database = Depends(lambda: db_instance)):
    return {"db": db.connect()}
```

## 6.3 依赖链

### 嵌套依赖

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def query_extractor(q: str = None):
    return q

def query_or_cookie_extractor(
    q: str = Depends(query_extractor),
    last_query: str = None
):
    if not q:
        return last_query
    return q

@app.get("/items/")
async def read_query(
    query_or_default: str = Depends(query_or_cookie_extractor)
):
    return {"query_or_default": query_or_default}
```

### 多层依赖

```python
class DB:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

def get_db():
    return DB("postgresql://localhost/app")

def get_user_service(db: DB = Depends(get_db)):
    return UserService(db)

def get_auth_service(user_service: UserService = Depends(get_user_service)):
    return AuthService(user_service)

@app.get("/users/")
async def read_users(
    auth_service: AuthService = Depends(get_auth_service)
):
    return auth_service.get_users()
```

## 6.4 可调用实例

### __call__ 方法

```python
from fastapi import FastAPI, Depends

app = FastAPI()

class FixedContentQueryChecker:
    def __init__(self, fixed_content: str):
        self.fixed_content = fixed_content

    def __call__(self, q: str = ""):
        if q:
            return self.fixed_content in q
        return False

checker = FixedContentQueryChecker("bar")

@app.get("/items/")
async def read_items(is_fixed_content_included: bool = Depends(checker)):
    return {"is_fixed_content_included": is_fixed_content_included}
```

## 6.5 使用路径操作装饰器

### 全局依赖

```python
from fastapi import FastAPI, Depends, HTTPException, Header

app = FastAPI()

async def verify_token(x_token: str = Header(...)):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")

async def verify_key(x_key: str = Header(...)):
    if x_key != "fake-super-secret-key":
        raise HTTPException(status_code=400, detail="X-Key header invalid")
    return x_key

@app.get("/items/", dependencies=[Depends(verify_token), Depends(verify_key)])
async def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]
```

### 路由器依赖

```python
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/users",
    tags=["users"],
    dependencies=[Depends(verify_token), Depends(verify_key)],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
async def read_users():
    return [{"username": "john"}, {"username": "jane"}]

@router.get("/{user_id}")
async def read_user(user_id: int):
    return {"user_id": user_id}
```

### 全局应用依赖

```python
app = FastAPI(dependencies=[Depends(verify_token), Depends(verify_key)])

@app.get("/items/")
async def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]

@app.get("/users/")
async def read_users():
    return [{"username": "john"}, {"username": "jane"}]
```

## 6.6 依赖中的 yield

### 基本用法

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_db():
    db = Database()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
async def read_items(db: Database = Depends(get_db)):
    return db.query("SELECT * FROM items")
```

### 资源清理

```python
class Database:
    def __init__(self):
        print("Database connection created")
    
    def query(self, sql: str):
        return f"Result of: {sql}"
    
    def close(self):
        print("Database connection closed")

def get_db():
    db = Database()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
async def read_items(db: Database = Depends(get_db)):
    return db.query("SELECT * FROM items")
```

### 多个 yield 依赖

```python
async def dependency_a():
    print("Dependency A - Setup")
    try:
        yield "A"
    finally:
        print("Dependency A - Cleanup")

async def dependency_b(dep_a: str = Depends(dependency_a)):
    print(f"Dependency B - Setup (with {dep_a})")
    try:
        yield f"B({dep_a})"
    finally:
        print("Dependency B - Cleanup")

@app.get("/")
async def main(dep_b: str = Depends(dependency_b)):
    print(f"Main - Processing with {dep_b}")
    return {"dependency": dep_b}
```

执行顺序：
```
Dependency A - Setup
Dependency B - Setup (with A)
Main - Processing with B(A)
Dependency B - Cleanup
Dependency A - Cleanup
```

## 6.7 实际应用示例

### 数据库依赖

```python
from fastapi import FastAPI, Depends
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

app = FastAPI()

# 数据库配置
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 数据库依赖
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 使用
@app.get("/users/")
def read_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
```

### 认证依赖

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

class AuthService:
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def is_valid(self) -> bool:
        return self.api_key == "valid-api-key"
    
    def get_user(self) -> str:
        return "user123"

def get_current_user(
    authorization: Optional[str] = Header(None)
) -> AuthService:
    if not authorization:
        raise HTTPException(status_code=401, detail="未授权")
    
    auth = AuthService(authorization)
    if not auth.is_valid():
        raise HTTPException(status_code=401, detail="无效的 API Key")
    
    return auth

@app.get("/protected/")
async def protected_route(auth: AuthService = Depends(get_current_user)):
    return {"message": "这是受保护的路由", "user": auth.get_user()}
```

### 配置依赖

```python
from fastapi import FastAPI, Depends
from pydantic import BaseSettings

class Settings(BaseSettings):
    app_name: str = "My API"
    admin_email: str = "admin@example.com"
    version: str = "1.0.0"
    debug: bool = False
    
    class Config:
        env_file = ".env"

def get_settings():
    return Settings()

app = FastAPI()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {
        "app_name": settings.app_name,
        "admin_email": settings.admin_email,
        "version": settings.version,
    }
```

### 分页依赖

```python
from fastapi import FastAPI, Depends, Query
from typing import Optional

app = FastAPI()

class PaginationParams:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(10, ge=1, le=100),
        search: Optional[str] = None,
        sort_by: Optional[str] = None,
        sort_order: str = Query("asc", regex="^(asc|desc)$")
    ):
        self.page = page
        self.page_size = page_size
        self.search = search
        self.sort_by = sort_by
        self.sort_order = sort_order

@app.get("/items/")
async def read_items(pagination: PaginationParams = Depends()):
    return {
        "page": pagination.page,
        "page_size": pagination.page_size,
        "search": pagination.search,
        "sort_by": pagination.sort_by,
        "sort_order": pagination.sort_order
    }
```

## 6.8 高级用法

### 缓存依赖

```python
from functools import lru_cache

@lru_cache()
def get_settings():
    return Settings()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return settings
```

### 依赖覆盖（测试）

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient

app = FastAPI()

def get_db():
    # 生产数据库
    return "production_db"

@app.get("/items/")
async def read_items(db: str = Depends(get_db)):
    return {"db": db}

# 测试时覆盖
def override_get_db():
    return "test_db"

app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)

def test_read_items():
    response = client.get("/items/")
    assert response.json() == {"db": "test_db"}

# 清除覆盖
app.dependency_overrides = {}
```

### 可选依赖

```python
from typing import Optional

def get_current_user_optional(authorization: Optional[str] = Header(None)):
    if authorization:
        return AuthService(authorization)
    return None

@app.get("/items/")
async def read_items(
    auth: Optional[AuthService] = Depends(get_current_user_optional)
):
    if auth:
        return {"message": f"Hello {auth.get_user()}"}
    return {"message": "Hello Guest"}
```

## 6.9 完整示例：博客系统

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime

app = FastAPI(title="博客系统")

# 模型
class User(BaseModel):
    id: int
    username: str
    email: str

class Post(BaseModel):
    id: int
    title: str
    content: str
    author: User
    created_at: datetime

# 数据库模拟
fake_db = {
    "users": {},
    "posts": {}
}

# 依赖：数据库
def get_db():
    return fake_db

# 依赖：认证
def get_current_user(
    token: str = Header(..., alias="Authorization"),
    db: dict = Depends(get_db)
) -> User:
    # 模拟 token 验证
    user_id = token.split("-")[-1]
    if user_id not in db["users"]:
        raise HTTPException(status_code=401, detail="无效的认证信息")
    return db["users"][user_id]

# 依赖：可选认证
def get_current_user_optional(
    token: Optional[str] = Header(None, alias="Authorization"),
    db: dict = Depends(get_db)
) -> Optional[User]:
    if not token:
        return None
    user_id = token.split("-")[-1]
    return db["users"].get(user_id)

# 依赖：分页
class Pagination:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(10, ge=1, le=100)
    ):
        self.skip = (page - 1) * page_size
        self.limit = page_size

# 路由
@app.post("/users/", response_model=User)
async def create_user(
    username: str,
    email: str,
    db: dict = Depends(get_db)
):
    user_id = len(db["users"]) + 1
    user = User(id=user_id, username=username, email=email)
    db["users"][user_id] = user
    return user

@app.post("/posts/", response_model=Post)
async def create_post(
    title: str,
    content: str,
    current_user: User = Depends(get_current_user),
    db: dict = Depends(get_db)
):
    post_id = len(db["posts"]) + 1
    post = Post(
        id=post_id,
        title=title,
        content=content,
        author=current_user,
        created_at=datetime.now()
    )
    db["posts"][post_id] = post
    return post

@app.get("/posts/", response_model=List[Post])
async def list_posts(
    pagination: Pagination = Depends(),
    db: dict = Depends(get_db)
):
    posts = list(db["posts"].values())
    return posts[pagination.skip : pagination.skip + pagination.limit]

@app.get("/posts/{post_id}", response_model=Post)
async def read_post(
    post_id: int,
    current_user: Optional[User] = Depends(get_current_user_optional),
    db: dict = Depends(get_db)
):
    if post_id not in db["posts"]:
        raise HTTPException(status_code=404, detail="文章未找到")
    
    post = db["posts"][post_id]
    # 如果用户已登录，显示额外信息
    if current_user:
        print(f"User {current_user.username} is viewing post {post_id}")
    
    return post
```

## 6.10 练习

1. **基础练习**
   - 创建一个数据库依赖
   - 实现简单的认证依赖
   - 使用类作为依赖

2. **进阶练习**
   - 实现依赖链
   - 使用 yield 管理资源
   - 创建分页依赖

3. **挑战练习**
   - 实现权限系统依赖
   - 创建缓存依赖
   - 实现依赖覆盖用于测试

## 6.11 小结

- 依赖注入是 FastAPI 的核心特性
- 可以使用函数或类作为依赖
- 依赖可以嵌套形成依赖链
- 使用 yield 可以进行资源清理
- 可以在路由、路径操作和全局级别使用依赖
- 依赖提高了代码的复用性和可测试性

## 下一章

[第七章：数据库集成](./07_database.md) - 学习如何在 FastAPI 中使用数据库！
