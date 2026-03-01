# 第十章：测试

## 10.1 测试基础

### 安装测试依赖

```bash
pip install pytest
pip install httpx
pip install pytest-asyncio
```

### 第一个测试

创建 `test_main.py`：

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}
```

运行测试：
```bash
pytest
```

## 10.2 测试客户端

### 同步测试

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_item():
    response = client.get("/items/1")
    assert response.status_code == 200
    assert response.json() == {"item_id": 1}

def test_create_item():
    response = client.post(
        "/items/",
        json={"name": "Item 1", "price": 10.5}
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Item 1"
```

### 异步测试

```python
import pytest
from httpx import AsyncClient
from main import app

@pytest.mark.asyncio
async def test_async_read_item():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/items/1")
    assert response.status_code == 200
```

## 10.3 测试 CRUD 操作

### 应用代码

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI()

class Item(BaseModel):
    id: Optional[int] = None
    name: str
    price: float
    description: Optional[str] = None

# 模拟数据库
items_db = {}

@app.post("/items/", response_model=Item, status_code=201)
def create_item(item: Item):
    item_id = max(items_db.keys(), default=0) + 1
    item.id = item_id
    items_db[item_id] = item
    return item

@app.get("/items/", response_model=List[Item])
def read_items():
    return list(items_db.values())

@app.get("/items/{item_id}", response_model=Item)
def read_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return items_db[item_id]

@app.put("/items/{item_id}", response_model=Item)
def update_item(item_id: int, item: Item):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    item.id = item_id
    items_db[item_id] = item
    return item

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    del items_db[item_id]
    return {"message": "Item deleted"}
```

### 测试代码

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

class TestItems:
    def test_create_item(self):
        response = client.post(
            "/items/",
            json={"name": "Item 1", "price": 10.5}
        )
        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Item 1"
        assert data["price"] == 10.5
        assert "id" in data
    
    def test_read_items(self):
        # 创建测试数据
        client.post("/items/", json={"name": "Item 1", "price": 10.5})
        client.post("/items/", json={"name": "Item 2", "price": 20.5})
        
        response = client.get("/items/")
        assert response.status_code == 200
        data = response.json()
        assert len(data) >= 2
    
    def test_read_item(self):
        # 创建测试数据
        create_response = client.post(
            "/items/",
            json={"name": "Test Item", "price": 15.5}
        )
        item_id = create_response.json()["id"]
        
        # 读取
        response = client.get(f"/items/{item_id}")
        assert response.status_code == 200
        data = response.json()
        assert data["name"] == "Test Item"
        assert data["price"] == 15.5
    
    def test_read_item_not_found(self):
        response = client.get("/items/99999")
        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()
    
    def test_update_item(self):
        # 创建测试数据
        create_response = client.post(
            "/items/",
            json={"name": "Original", "price": 10.0}
        )
        item_id = create_response.json()["id"]
        
        # 更新
        response = client.put(
            f"/items/{item_id}",
            json={"name": "Updated", "price": 20.0}
        )
        assert response.status_code == 200
        data = response.json()
        assert data["name"] == "Updated"
        assert data["price"] == 20.0
    
    def test_delete_item(self):
        # 创建测试数据
        create_response = client.post(
            "/items/",
            json={"name": "To Delete", "price": 10.0}
        )
        item_id = create_response.json()["id"]
        
        # 删除
        response = client.delete(f"/items/{item_id}")
        assert response.status_code == 200
        
        # 验证已删除
        get_response = client.get(f"/items/{item_id}")
        assert get_response.status_code == 404
```

## 10.4 测试认证

### 应用代码

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def get_current_user(token: str = Depends(oauth2_scheme)):
    if token == "invalid":
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"username": "testuser", "token": token}

@app.get("/protected")
def protected_route(user: dict = Depends(get_current_user)):
    return {"message": f"Hello {user['username']}"}
```

### 测试代码

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_protected_route_without_token():
    response = client.get("/protected")
    assert response.status_code == 401

def test_protected_route_with_invalid_token():
    response = client.get(
        "/protected",
        headers={"Authorization": "Bearer invalid"}
    )
    assert response.status_code == 401

def test_protected_route_with_valid_token():
    response = client.get(
        "/protected",
        headers={"Authorization": "Bearer valid-token"}
    )
    assert response.status_code == 200
    assert response.json()["message"] == "Hello testuser"
```

## 10.5 测试数据库

### 使用测试数据库

```python
# conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from fastapi.testclient import TestClient
from main import app, get_db, Base

# 测试数据库
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 创建表
Base.metadata.create_all(bind=engine)

@pytest.fixture
def db_session():
    connection = engine.connect()
    transaction = connection.begin()
    session = TestingSessionLocal(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture
def client(db_session):
    def override_get_db():
        try:
            yield db_session
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

### 测试用例

```python
# test_main.py
def test_create_user(client):
    response = client.post(
        "/users/",
        json={"username": "testuser", "email": "test@example.com"}
    )
    assert response.status_code == 201
    assert response.json()["username"] == "testuser"

def test_read_users(client, db_session):
    # 创建测试数据
    user = User(username="testuser", email="test@example.com")
    db_session.add(user)
    db_session.commit()
    
    response = client.get("/users/")
    assert response.status_code == 200
    assert len(response.json()) == 1
```

## 10.6 Pytest Fixtures

### 常用 Fixtures

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def test_user():
    return {
        "username": "testuser",
        "email": "test@example.com",
        "password": "testpass123"
    }

@pytest.fixture
def auth_headers(client, test_user):
    # 注册用户
    client.post("/register", json=test_user)
    
    # 登录获取 token
    response = client.post(
        "/token",
        data={
            "username": test_user["username"],
            "password": test_user["password"]
        }
    )
    token = response.json()["access_token"]
    
    return {"Authorization": f"Bearer {token}"}
```

### 使用 Fixtures

```python
# test_main.py
def test_protected_route(client, auth_headers):
    response = client.get("/protected", headers=auth_headers)
    assert response.status_code == 200

def test_create_post(client, auth_headers):
    response = client.post(
        "/posts/",
        json={"title": "Test Post", "content": "Test Content"},
        headers=auth_headers
    )
    assert response.status_code == 201
```

## 10.7 参数化测试

```python
import pytest

@pytest.mark.parametrize("item_id,expected_status", [
    (1, 200),
    (2, 200),
    (999, 404),
    (-1, 404),
])
def test_read_item_parametrized(client, item_id, expected_status):
    response = client.get(f"/items/{item_id}")
    assert response.status_code == expected_status

@pytest.mark.parametrize("price,should_pass", [
    (10.5, True),
    (0, True),
    (-10, False),
    (1000000, True),
])
def test_create_item_validation(client, price, should_pass):
    response = client.post(
        "/items/",
        json={"name": "Test", "price": price}
    )
    if should_pass:
        assert response.status_code == 201
    else:
        assert response.status_code == 422
```

## 10.8 测试覆盖多个场景

```python
class TestUserRegistration:
    def test_register_success(self, client):
        response = client.post(
            "/register",
            json={
                "username": "newuser",
                "email": "new@example.com",
                "password": "password123"
            }
        )
        assert response.status_code == 201
    
    def test_register_duplicate_username(self, client, test_user):
        # 第一次注册
        client.post("/register", json=test_user)
        
        # 第二次注册（相同用户名）
        response = client.post(
            "/register",
            json={
                "username": test_user["username"],
                "email": "another@example.com",
                "password": "password123"
            }
        )
        assert response.status_code == 400
        assert "already exists" in response.json()["detail"].lower()
    
    def test_register_duplicate_email(self, client, test_user):
        client.post("/register", json=test_user)
        
        response = client.post(
            "/register",
            json={
                "username": "anotheruser",
                "email": test_user["email"],
                "password": "password123"
            }
        )
        assert response.status_code == 400
    
    def test_register_invalid_email(self, client):
        response = client.post(
            "/register",
            json={
                "username": "newuser",
                "email": "invalid-email",
                "password": "password123"
            }
        )
        assert response.status_code == 422
    
    def test_register_weak_password(self, client):
        response = client.post(
            "/register",
            json={
                "username": "newuser",
                "email": "new@example.com",
                "password": "123"  # 太短
            }
        )
        assert response.status_code == 422
```

## 10.9 完整测试示例

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# 模型
class User(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool = True

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class Token(BaseModel):
    access_token: str
    token_type: str

# 模拟数据库
users_db = {}
tokens_db = {}

# 辅助函数
def get_user_by_username(username: str):
    for user in users_db.values():
        if user["username"] == username:
            return user
    return None

def get_user_by_email(email: str):
    for user in users_db.values():
        if user["email"] == email:
            return user
    return None

# 路由
@app.post("/register", response_model=User, status_code=status.HTTP_201_CREATED)
def register(user: UserCreate):
    if get_user_by_username(user.username):
        raise HTTPException(status_code=400, detail="Username already exists")
    if get_user_by_email(user.email):
        raise HTTPException(status_code=400, detail="Email already registered")
    
    user_id = len(users_db) + 1
    users_db[user_id] = {
        "id": user_id,
        "username": user.username,
        "email": user.email,
        "password": user.password,  # 实际应用应该哈希
        "is_active": True
    }
    return User(**users_db[user_id])

@app.post("/token", response_model=Token)
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = get_user_by_username(form_data.username)
    if not user or user["password"] != form_data.password:
        raise HTTPException(
            status_code=401,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    token = f"token-{user['id']}"
    tokens_db[token] = user["id"]
    return {"access_token": token, "token_type": "bearer"}

def get_current_user(token: str = Depends(oauth2_scheme)):
    if token not in tokens_db:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    user_id = tokens_db[token]
    user = users_db.get(user_id)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return User(**user)

@app.get("/users/me", response_model=User)
def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

```python
# test_main.py
import pytest
from fastapi.testclient import TestClient
from main import app, users_db, tokens_db

client = TestClient(app)

@pytest.fixture(autouse=True)
def cleanup():
    # 每个测试前清空数据库
    users_db.clear()
    tokens_db.clear()
    yield
    users_db.clear()
    tokens_db.clear()

class TestHealthCheck:
    def test_health_check(self):
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json() == {"status": "healthy"}

class TestRegistration:
    def test_register_success(self):
        response = client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "test@example.com",
                "password": "password123"
            }
        )
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "testuser"
        assert data["email"] == "test@example.com"
        assert "id" in data
    
    def test_register_duplicate_username(self):
        # 第一次注册
        client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "test1@example.com",
                "password": "password123"
            }
        )
        
        # 第二次注册（相同用户名）
        response = client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "test2@example.com",
                "password": "password123"
            }
        )
        assert response.status_code == 400
        assert "username already exists" in response.json()["detail"].lower()
    
    def test_register_duplicate_email(self):
        client.post(
            "/register",
            json={
                "username": "user1",
                "email": "test@example.com",
                "password": "password123"
            }
        )
        
        response = client.post(
            "/register",
            json={
                "username": "user2",
                "email": "test@example.com",
                "password": "password123"
            }
        )
        assert response.status_code == 400
    
    def test_register_invalid_email(self):
        response = client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "invalid-email",
                "password": "password123"
            }
        )
        assert response.status_code == 422

class TestAuthentication:
    @pytest.fixture
    def registered_user(self):
        response = client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "test@example.com",
                "password": "password123"
            }
        )
        return response.json()
    
    def test_login_success(self, registered_user):
        response = client.post(
            "/token",
            data={
                "username": "testuser",
                "password": "password123"
            }
        )
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"
    
    def test_login_wrong_password(self, registered_user):
        response = client.post(
            "/token",
            data={
                "username": "testuser",
                "password": "wrongpassword"
            }
        )
        assert response.status_code == 401
    
    def test_login_nonexistent_user(self):
        response = client.post(
            "/token",
            data={
                "username": "nonexistent",
                "password": "password123"
            }
        )
        assert response.status_code == 401

class TestProtectedRoutes:
    @pytest.fixture
    def auth_token(self, registered_user):
        response = client.post(
            "/token",
            data={
                "username": "testuser",
                "password": "password123"
            }
        )
        return response.json()["access_token"]
    
    @pytest.fixture
    def registered_user(self):
        response = client.post(
            "/register",
            json={
                "username": "testuser",
                "email": "test@example.com",
                "password": "password123"
            }
        )
        return response.json()
    
    def test_read_users_me_success(self, auth_token):
        response = client.get(
            "/users/me",
            headers={"Authorization": f"Bearer {auth_token}"}
        )
        assert response.status_code == 200
        data = response.json()
        assert data["username"] == "testuser"
    
    def test_read_users_me_no_token(self):
        response = client.get("/users/me")
        assert response.status_code == 401
    
    def test_read_users_me_invalid_token(self):
        response = client.get(
            "/users/me",
            headers={"Authorization": "Bearer invalid-token"}
        )
        assert response.status_code == 401
```

## 10.10 运行测试

```bash
# 运行所有测试
pytest

# 详细输出
pytest -v

# 运行特定文件
pytest test_main.py

# 运行特定测试类
pytest test_main.py::TestRegistration

# 运行特定测试方法
pytest test_main.py::TestRegistration::test_register_success

# 显示打印输出
pytest -s

# 生成覆盖率报告
pytest --cov=main

# 生成 HTML 覆盖率报告
pytest --cov=main --cov-report=html
```

## 10.11 练习

1. **基础练习**
   - 为 API 端点编写基本测试
   - 测试 CRUD 操作
   - 测试错误场景

2. **进阶练习**
   - 使用 fixtures 管理测试数据
   - 编写参数化测试
   - 测试认证流程

3. **挑战练习**
   - 实现测试数据库隔离
   - 编写端到端测试
   - 集成测试覆盖率报告

## 10.12 小结

- 使用 TestClient 进行 API 测试
- pytest 提供强大的测试功能
- fixtures 用于管理测试依赖
- 参数化测试可以覆盖多种场景
- 测试应该覆盖成功和失败情况
- 使用覆盖率工具确保测试完整性

## 下一章

[第十一章：部署](./11_deployment.md) - 学习如何部署 FastAPI 应用！
