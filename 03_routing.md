# 第三章：路径操作与路由

## 3.1 路由基础

### 路径操作装饰器

FastAPI 提供以下装饰器：
- `@app.get()`
- `@app.post()`
- `@app.put()`
- `@app.delete()`
- `@app.patch()`
- `@app.options()`
- `@app.head()`
- `@app.trace()`

### 完整示例

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def get_root():
    return {"method": "GET"}

@app.post("/")
def post_root():
    return {"method": "POST"}

@app.put("/")
def put_root():
    return {"method": "PUT"}

@app.delete("/")
def delete_root():
    return {"method": "DELETE"}
```

## 3.2 路径参数进阶

### 路径参数验证

```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/items/{item_id}")
async def read_items(
    item_id: int = Path(
        ...,
        title="Item ID",
        description="The ID of the item to get",
        ge=1,
        le=1000
    )
):
    return {"item_id": item_id}
```

### 验证参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `gt` | 大于 | `gt=0` |
| `ge` | 大于等于 | `ge=1` |
| `lt` | 小于 | `lt=100` |
| `le` | 小于等于 | `le=1000` |

### 浮点数验证

```python
@app.get("/items/{item_id}")
async def read_items(
    item_id: float = Path(
        ...,
        title="Item ID",
        gt=0,
        le=1000.00
    )
):
    return {"item_id": item_id}
```

### 字符串验证

```python
@app.get("/items/{item_name}")
async def read_items(
    item_name: str = Path(
        ...,
        min_length=3,
        max_length=50,
        regex="^[a-zA-Z0-9_-]+$"
    )
):
    return {"item_name": item_name}
```

## 3.3 查询参数进阶

### 查询参数列表

```python
from typing import List

@app.get("/items/")
async def read_items(q: List[str] = Query(None)):
    query_items = {"q": q}
    return query_items
```

访问：`/items/?q=foo&q=bar`

### 使用 Query 验证

```python
from fastapi import Query

@app.get("/items/")
async def read_items(
    q: str = Query(
        None,
        min_length=3,
        max_length=50,
        regex="^fixedquery$"
    )
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

### 必需参数

```python
@app.get("/items/")
async def read_items(q: str = Query(..., min_length=3)):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

### 查询参数元数据

```python
@app.get("/items/")
async def read_items(
    q: str = Query(
        None,
        alias="item-query",
        title="Query string",
        description="Query string for the items",
        deprecated=True,
    )
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

## 3.4 路径操作配置

### 响应模型

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

@app.post("/items/", response_model=Item)
async def create_item(item: Item):
    return item
```

### 响应状态码

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(name: str):
    return {"name": name}
```

### 标签（Tags）

```python
@app.post("/items/", tags=["items"])
async def create_item(item: Item):
    return item

@app.get("/items/", tags=["items"])
async def read_items():
    return [{"name": "Foo"}, {"name": "Bar"}]

@app.get("/users/", tags=["users"])
async def read_users():
    return [{"username": "johndoe"}]
```

### 描述和文档

```python
@app.post(
    "/items/",
    response_model=Item,
    status_code=status.HTTP_201_CREATED,
    tags=["items"],
    summary="Create an item",
    description="Create an item with all the information",
    response_description="The created item",
)
async def create_item(item: Item):
    """
    Create an item with all the information:
    
    - **name**: each item must have a name
    - **description**: a long description
    - **price**: required
    - **tax**: if the item doesn't have tax, you can omit this
    """
    return item
```

### 弃用路由

```python
@app.get("/items/", deprecated=True)
async def read_items():
    return [{"name": "Foo"}]
```

## 3.5 模块化路由

### 创建路由模块

`app/routers/items.py`:

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def read_items():
    return [{"name": "Item 1"}, {"name": "Item 2"}]

@router.get("/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id, "name": f"Item {item_id}"}
```

`app/routers/users.py`:

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def read_users():
    return [{"username": "john"}, {"username": "jane"}]

@router.get("/{user_id}")
async def read_user(user_id: int):
    return {"user_id": user_id, "username": f"user{user_id}"}
```

### 主应用导入路由

`app/main.py`:

```python
from fastapi import FastAPI
from app.routers import items, users

app = FastAPI()

app.include_router(items.router, prefix="/items", tags=["items"])
app.include_router(users.router, prefix="/users", tags=["users"])

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### 路由器配置

```python
router = APIRouter(
    prefix="/items",
    tags=["items"],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
async def read_items():
    return [{"name": "Item 1"}]

@router.get("/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

### 带依赖的路由器

```python
from fastapi import APIRouter, Depends

def get_current_user():
    return {"username": "john"}

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(get_current_user)],
)

@router.get("/")
async def read_items():
    return [{"name": "Item 1"}]
```

## 3.6 路径操作顺序

### 重要规则

路径操作的顺序很重要，FastAPI 会按顺序匹配。

```python
# ❌ 错误示例
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "current user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

访问 `/users/me` 会返回 `{"user_id": "current user"}`。

```python
# ✅ 正确示例
@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}

@app.get("/users/me")
async def read_user_me():
    return {"user_id": "current user"}
```

访问 `/users/me` 会返回 `{"user_id": "me"}`（错误！）

**正确做法：**

```python
# ✅ 最正确的顺序
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "current user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

## 3.7 路由元数据

### 标题、描述、版本

```python
app = FastAPI(
    title="My API",
    description="A very cool API",
    version="1.0.0",
    terms_of_service="http://example.com/terms/",
    contact={
        "name": "Support Team",
        "url": "http://example.com/contact",
        "email": "support@example.com",
    },
    license_info={
        "name": "Apache 2.0",
        "url": "https://www.apache.org/licenses/LICENSE-2.0.html",
    },
)
```

### 响应模型

```python
from typing import List

@app.get(
    "/items/",
    response_model=List[Item],
    response_description="List of items",
    responses={
        200: {
            "description": "Return list of items",
            "content": {
                "application/json": {
                    "example": [
                        {"name": "Item 1", "price": 10.5},
                        {"name": "Item 2", "price": 20.5}
                    ]
                }
            },
        },
        404: {"description": "Items not found"},
    },
)
async def read_items():
    return [{"name": "Item 1", "price": 10.5}]
```

## 3.8 静态文件

### 挂载静态文件

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")
```

目录结构：
```
.
├── app
│   └── main.py
└── static
    ├── images
    ├── css
    └── js
```

访问：`http://localhost:8000/static/images/logo.png`

## 3.9 完整示例：博客 API

```python
from fastapi import FastAPI, APIRouter, HTTPException, Query, Path
from pydantic import BaseModel
from typing import Optional, List
from enum import Enum

app = FastAPI(title="博客 API")

# 枚举
class PostStatus(str, Enum):
    draft = "draft"
    published = "published"

# 模型
class Post(BaseModel):
    id: Optional[int] = None
    title: str
    content: str
    author: str
    status: PostStatus = PostStatus.draft

# 路由器
router = APIRouter(prefix="/posts", tags=["posts"])

# 模拟数据库
posts_db = {}

@router.post("/", status_code=201, summary="创建文章")
async def create_post(post: Post):
    """创建一篇新文章"""
    post_id = max(posts_db.keys(), default=0) + 1
    post.id = post_id
    posts_db[post_id] = post
    return post

@router.get("/", response_model=List[Post])
async def list_posts(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    status: Optional[PostStatus] = None,
):
    """获取文章列表"""
    posts = list(posts_db.values())
    if status:
        posts = [p for p in posts if p.status == status]
    return posts[skip : skip + limit]

@router.get("/{post_id}", response_model=Post)
async def get_post(
    post_id: int = Path(..., ge=1, description="文章 ID")
):
    """获取单篇文章"""
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="文章未找到")
    return posts_db[post_id]

@router.put("/{post_id}", response_model=Post)
async def update_post(post_id: int, post: Post):
    """更新文章"""
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="文章未找到")
    post.id = post_id
    posts_db[post_id] = post
    return post

@router.delete("/{post_id}")
async def delete_post(post_id: int):
    """删除文章"""
    if post_id not in posts_db:
        raise HTTPException(status_code=404, detail="文章未找到")
    del posts_db[post_id]
    return {"message": "文章已删除"}

# 注册路由
app.include_router(router)

@app.get("/")
async def root():
    return {"message": "欢迎访问博客 API"}
```

## 3.10 练习

1. **基础练习**
   - 创建一个用户路由模块
   - 实现基本的 CRUD 操作
   - 使用标签组织 API

2. **进阶练习**
   - 实现分页功能
   - 添加查询参数验证
   - 使用枚举类型

3. **挑战练习**
   - 实现嵌套路由（如 /users/{id}/posts）
   - 添加路由中间件
   - 实现版本化 API

## 3.11 小结

- 路径操作使用装饰器定义
- 路径参数可以使用 Path 进行验证
- 查询参数可以使用 Query 进行验证
- 使用 APIRouter 实现模块化路由
- 路由顺序很重要，更具体的路由应放在前面
- 使用 tags、summary、description 增强 API 文档

## 下一章

[第四章：请求与响应](./04_request_response.md) - 深入了解请求处理和响应格式！
