# 第一章：简介与安装

## 1.1 FastAPI 简介

### 什么是 FastAPI？

FastAPI 是一个现代、高性能的 Python Web 框架，专门用于构建 API。它基于 Python 3.7+ 的类型提示（Type Hints），能够自动生成 API 文档并提供优秀的编辑器支持。

### 为什么选择 FastAPI？

#### 性能优势
- 与 NodeJS 和 Go 相当的性能
- 基于 Starlette 和 Uvicorn 的高性能异步框架
- 支持 async/await 异步编程

#### 开发效率
- 自动生成 API 文档（Swagger UI 和 ReDoc）
- 强大的类型检查和自动补全
- 减少约 40% 的人为错误
- 简洁直观的语法

#### 标准化
- 完全兼容 OpenAPI（原 Swagger）规范
- 自动生成 JSON Schema
- 支持 OAuth2 和 JWT 认证

## 1.2 核心特性

### 1. 类型提示
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

### 2. 自动 API 文档
- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`

### 3. 数据验证
```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = None
```

### 4. 异步支持
```python
@app.get("/async-endpoint")
async def async_read():
    await some_async_operation()
    return {"message": "Async operation completed"}
```

## 1.3 安装 FastAPI

### 系统要求
- Python 3.7 或更高版本
- pip 包管理器

### 安装步骤

#### 1. 创建虚拟环境（推荐）
```bash
# 使用 venv
python -m venv venv

# Windows 激活
venv\Scripts\activate

# Linux/Mac 激活
source venv/bin/activate
```

#### 2. 安装 FastAPI 和 Uvicorn
```bash
pip install fastapi uvicorn[standard]
```

#### 3. 安装开发依赖（可选）
```bash
pip install python-multipart  # 表单处理
pip install sqlalchemy        # 数据库 ORM
pip install databases         # 异步数据库支持
pip install pytest           # 测试框架
pip install httpx            # 异步 HTTP 客户端
```

### 验证安装

创建 `main.py` 文件：
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "FastAPI is working!"}
```

运行服务器：
```bash
uvicorn main:app --reload
```

访问 `http://localhost:8000` 查看结果。

## 1.4 项目结构推荐

```
my_fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI 应用入口
│   ├── dependencies.py      # 依赖注入
│   ├── routers/             # 路由模块
│   │   ├── __init__.py
│   │   ├── items.py
│   │   └── users.py
│   ├── models/              # 数据库模型
│   │   ├── __init__.py
│   │   └── models.py
│   ├── schemas/             # Pydantic 模型
│   │   ├── __init__.py
│   │   └── schemas.py
│   ├── services/            # 业务逻辑
│   │   ├── __init__.py
│   │   └── user_service.py
│   ├── core/                # 核心配置
│   │   ├── __init__.py
│   │   ├── config.py
│   │   └── security.py
│   └── utils/               # 工具函数
│       ├── __init__.py
│       └── helpers.py
├── tests/                   # 测试文件
│   ├── __init__.py
│   ├── test_main.py
│   └── test_users.py
├── .env                     # 环境变量
├── requirements.txt         # 依赖列表
├── Dockerfile              # Docker 配置
└── README.md               # 项目文档
```

## 1.5 开发工具推荐

### IDE 和编辑器
1. **VS Code** - 推荐
   - 扩展：Python, Pylance, FastAPI Snippets
2. **PyCharm Professional** - 强大的 Python IDE
3. **Sublime Text** - 轻量级编辑器

### VS Code 推荐配置
`.vscode/settings.json`:
```json
{
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.formatting.provider": "black",
    "editor.formatOnSave": true
}
```

### 推荐的 Python 包
```txt
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
python-multipart>=0.0.6
pydantic>=2.0.0
pydantic-settings>=2.0.0
python-dotenv>=1.0.0
```

## 1.6 第一个 FastAPI 应用

创建一个完整的示例应用：

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI(
    title="My First API",
    description="A simple example API",
    version="1.0.0"
)

# 数据模型
class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

# 模拟数据库
fake_db = {}

# 路由
@app.get("/")
def read_root():
    """首页路由"""
    return {"message": "Welcome to FastAPI!"}

@app.post("/items/{item_id}")
def create_item(item_id: int, item: Item):
    """创建商品"""
    fake_db[item_id] = item
    return {"item_id": item_id, "item": item}

@app.get("/items/{item_id}")
def read_item(item_id: int):
    """获取商品"""
    if item_id in fake_db:
        return fake_db[item_id]
    return {"error": "Item not found"}

# 运行: uvicorn main:app --reload
```

## 1.7 练习

1. **基础练习**
   - 安装 FastAPI 和 Uvicorn
   - 创建一个简单的 API，返回 "Hello World"
   - 访问自动生成的文档页面

2. **进阶练习**
   - 创建一个待办事项（TODO）API
   - 包含以下端点：
     - GET /todos - 获取所有待办事项
     - POST /todos - 创建新待办事项
     - GET /todos/{id} - 获取特定待办事项
     - PUT /todos/{id} - 更新待办事项
     - DELETE /todos/{id} - 删除待办事项

3. **挑战练习**
   - 添加数据验证
   - 实现错误处理
   - 添加查询参数（如过滤、排序）

## 1.8 小结

- FastAPI 是现代、高性能的 Python Web 框架
- 基于类型提示，提供自动文档和验证
- 支持异步编程，性能优异
- 安装简单：`pip install fastapi uvicorn[standard]`
- 项目结构应清晰分层，便于维护

## 下一章

[第二章：快速开始](./02_quickstart.md) - 创建你的第一个完整的 FastAPI 应用！
