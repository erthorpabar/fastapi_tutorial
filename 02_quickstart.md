# 第二章：快速开始

## 2.1 创建第一个应用

### 最简单的 FastAPI 应用

创建 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
```

运行：
```bash
uvicorn main:app --reload
```

## 2.2 运行 FastAPI 应用

### 使用 Uvicorn

```bash
# 基本运行
uvicorn main:app

# 开发模式（自动重载）
uvicorn main:app --reload

# 指定端口
uvicorn main:app --port 8080

# 指定主机
uvicorn main:app --host 0.0.0.0

# 完整开发配置
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### 使用 Python 运行

创建 `run.py`：

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        log_level="info"
    )
```

运行：
```bash
python run.py
```

### 使用环境变量

```python
import os
from fastapi import FastAPI

app = FastAPI()

@app.get("/info")
def get_info():
    return {
        "environment": os.getenv("ENVIRONMENT", "development"),
        "version": os.getenv("APP_VERSION", "1.0.0")
    }
```

## 2.3 基础路由

### GET 请求

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {"message": "Home page"}

@app.get("/about")
def about():
    return {"message": "About page"}

@app.get("/contact")
def contact():
    return {"email": "contact@example.com", "phone": "123-456-7890"}
```

### POST 请求

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    name: str
    email: str
    age: int

@app.post("/users")
def create_user(user: User):
    return {
        "message": "User created successfully",
        "user": user
    }
```

### PUT 请求

```python
@app.put("/users/{user_id}")
def update_user(user_id: int, user: User):
    return {
        "user_id": user_id,
        "message": "User updated",
        "user": user
    }
```

### DELETE 请求

```python
@app.delete("/users/{user_id}")
def delete_user(user_id: int):
    return {
        "user_id": user_id,
        "message": "User deleted successfully"
    }
```

## 2.4 路径参数

### 基本路径参数

```python
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}
```

### 多个路径参数

```python
@app.get("/users/{user_id}/items/{item_id}")
def read_user_item(user_id: int, item_id: int):
    return {
        "user_id": user_id,
        "item_id": item_id
    }
```

### 路径参数类型

```python
from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    if model_name == ModelName.alexnet:
        return {"model": model_name, "message": "Deep Learning FTW!"}
    if model_name.value == "lenet":
        return {"model": model_name, "message": "LeCNN all the images"}
    return {"model": model_name, "message": "Have some residuals"}
```

### 路径包含路径

```python
from fastapi import Path

@app.get("/files/{file_path:path}")
def read_file(file_path: str):
    return {"file_path": file_path}
```

访问：`/files/home/johndoe/myfile.txt`

## 2.5 查询参数

### 基本查询参数

```python
fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
def read_items(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

访问：
- `/items/` - 使用默认值
- `/items/?skip=0&limit=2` - 自定义值

### 可选查询参数

```python
from typing import Optional

@app.get("/items/{item_id}")
def read_item(item_id: str, q: Optional[str] = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```

### 必需查询参数

```python
@app.get("/items/{item_id}")
def read_item(item_id: str, needy: str):
    return {"item_id": item_id, "needy": needy}
```

### 查询参数类型转换

```python
@app.get("/items/{item_id}")
def read_item(
    item_id: str,
    q: Optional[str] = None,
    short: bool = False
):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update(
            {"description": "This is an amazing item that has a long description"}
        )
    return item
```

访问：
- `/items/foo?short=true`
- `/items/foo?short=on`
- `/items/foo?short=yes`
- `/items/foo?short=1`

### 多个查询参数

```python
from fastapi import Query

@app.get("/items/")
async def read_items(
    q: Optional[str] = Query(
        None,
        alias="item-query",
        title="Query string",
        description="Query string for the items to search in the database",
        min_length=3,
        max_length=50,
        regex="^fixedquery$",
        deprecated=True,
    )
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

## 2.6 请求体

### 使用 Pydantic 模型

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

@app.post("/items/")
def create_item(item: Item):
    return item
```

### 使用请求体 + 路径参数 + 查询参数

```python
@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item, q: Optional[str] = None):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    return result
```

### 多个请求体参数

```python
class User(BaseModel):
    username: str
    full_name: Optional[str] = None

@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item, user: User):
    results = {"item_id": item_id, "item": item, "user": user}
    return results
```

## 2.7 完整示例：图书管理 API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI(title="图书管理 API")

# 数据模型
class Book(BaseModel):
    id: Optional[int] = None
    title: str
    author: str
    year: int
    isbn: Optional[str] = None

# 模拟数据库
books_db = {}

@app.get("/")
def read_root():
    return {"message": "欢迎使用图书管理 API"}

# 获取所有图书
@app.get("/books", response_model=List[Book])
def get_books():
    return list(books_db.values())

# 获取单本图书
@app.get("/books/{book_id}", response_model=Book)
def get_book(book_id: int):
    if book_id not in books_db:
        raise HTTPException(status_code=404, detail="图书未找到")
    return books_db[book_id]

# 创建图书
@app.post("/books", response_model=Book, status_code=201)
def create_book(book: Book):
    book_id = max(books_db.keys(), default=0) + 1
    book.id = book_id
    books_db[book_id] = book
    return book

# 更新图书
@app.put("/books/{book_id}", response_model=Book)
def update_book(book_id: int, book: Book):
    if book_id not in books_db:
        raise HTTPException(status_code=404, detail="图书未找到")
    book.id = book_id
    books_db[book_id] = book
    return book

# 删除图书
@app.delete("/books/{book_id}")
def delete_book(book_id: int):
    if book_id not in books_db:
        raise HTTPException(status_code=404, detail="图书未找到")
    del books_db[book_id]
    return {"message": "图书已删除"}

# 搜索图书
@app.get("/books/search/", response_model=List[Book])
def search_books(title: Optional[str] = None, author: Optional[str] = None):
    results = list(books_db.values())
    if title:
        results = [b for b in results if title.lower() in b.title.lower()]
    if author:
        results = [b for b in results if author.lower() in b.author.lower()]
    return results
```

## 2.8 测试 API

### 使用浏览器
访问 `http://localhost:8000/docs` 使用 Swagger UI 测试。

### 使用 curl

```bash
# GET 请求
curl http://localhost:8000/books

# POST 请求
curl -X POST http://localhost:8000/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Python编程", "author": "张三", "year": 2023}'

# PUT 请求
curl -X PUT http://localhost:8000/books/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Python高级编程", "author": "张三", "year": 2024}'

# DELETE 请求
curl -X DELETE http://localhost:8000/books/1
```

### 使用 Python requests

```python
import requests

# GET
response = requests.get("http://localhost:8000/books")
print(response.json())

# POST
data = {"title": "新书籍", "author": "作者", "year": 2024}
response = requests.post("http://localhost:8000/books", json=data)
print(response.json())
```

## 2.9 练习

1. **基础练习**
   - 创建一个产品管理 API
   - 实现增删改查（CRUD）操作
   - 测试所有端点

2. **进阶练习**
   - 添加查询参数（价格范围、分类）
   - 实现搜索功能
   - 添加数据验证

3. **挑战练习**
   - 添加分页功能
   - 实现排序功能
   - 添加缓存机制

## 2.10 小结

- FastAPI 应用通过装饰器定义路由
- 支持多种 HTTP 方法（GET、POST、PUT、DELETE）
- 路径参数在 URL 中定义
- 查询参数在 URL 后用 `?` 添加
- 请求体使用 Pydantic 模型定义
- 使用 Uvicorn 运行应用

## 下一章

[第三章：路径操作与路由](./03_routing.md) - 深入了解 FastAPI 的路由系统！
