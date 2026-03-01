# 第四章：请求与响应

## 4.1 请求体（Request Body）

### Pydantic 模型

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
async def create_item(item: Item):
    return item
```

### 请求体属性访问

```python
@app.post("/items/")
async def create_item(item: Item):
    item_dict = item.dict()
    if item.tax:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})
    return item_dict
```

### 请求体 + 路径参数 + 查询参数

```python
@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, q: Optional[str] = None):
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
async def update_item(item_id: int, item: Item, user: User):
    results = {"item_id": item_id, "item": item, "user": user}
    return results
```

请求体示例：
```json
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2
    },
    "user": {
        "username": "dave",
        "full_name": "Dave Grohl"
    }
}
```

### 单个值请求体

```python
from fastapi import Body

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, importance: int = Body(...)):
    results = {"item_id": item_id, "item": item, "importance": importance}
    return results
```

### 嵌套模型

```python
from typing import List, Set

class Image(BaseModel):
    url: str
    name: str

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None
    tags: Set[str] = set()
    images: Optional[List[Image]] = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```

请求体示例：
```json
{
    "name": "Foo",
    "description": "The pretender",
    "price": 42.0,
    "tax": 3.2,
    "tags": ["rock", "metal", "bar"],
    "images": [
        {"url": "http://example.com/baz.jpg", "name": "Baz"},
        {"url": "http://example.com/dave.jpg", "name": "Dave"}
    ]
}
```

## 4.2 请求头

### 读取请求头

```python
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items/")
async def read_items(user_agent: Optional[str] = Header(None)):
    return {"User-Agent": user_agent}
```

### 自动转换

FastAPI 自动将下划线转为连字符：
```python
@app.get("/items/")
async def read_items(
    strange_header: Optional[str] = Header(None)
):
    return {"strange_header": strange_header}
```

访问：`strange-header`（HTTP 头）

### 重复的请求头

```python
from typing import List

@app.get("/items/")
async def read_items(x_token: List[str] = Header(None)):
    return {"X-Token values": x_token}
```

## 4.3 Cookie

### 读取 Cookie

```python
from fastapi import FastAPI, Cookie

app = FastAPI()

@app.get("/items/")
async def read_items(ads_id: Optional[str] = Cookie(None)):
    return {"ads_id": ads_id}
```

### 设置 Cookie

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.post("/cookie/")
def create_cookie(response: Response):
    response.set_cookie(
        key="fakesession",
        value="fake-cookie-session-value"
    )
    return {"message": "Cookie set"}
```

## 4.4 表单数据

### 接收表单

```python
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/login/")
async def login(username: str = Form(...), password: str = Form(...)):
    return {"username": username}
```

### 文件上传

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/files/")
async def create_file(file: bytes = File(...)):
    return {"file_size": len(file)}

@app.post("/uploadfile/")
async def create_upload_file(file: UploadFile):
    return {"filename": file.filename}
```

### 多文件上传

```python
from typing import List

@app.post("/files/")
async def create_files(files: List[bytes] = File(...)):
    return {"file_sizes": [len(file) for file in files]}

@app.post("/uploadfiles/")
async def create_upload_files(files: List[UploadFile]):
    return {"filenames": [file.filename for file in files]}
```

### 文件 + 表单

```python
@app.post("/files/")
async def create_file(
    file: bytes = File(...),
    fileb: UploadFile = File(...),
    token: str = Form(...)
):
    return {
        "file_size": len(file),
        "token": token,
        "fileb_content_type": fileb.content_type,
    }
```

## 4.5 响应模型

### 基本响应模型

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class User(BaseModel):
    username: str
    email: str
    full_name: Optional[str] = None

class UserInDB(User):
    hashed_password: str

@app.post("/user/", response_model=User)
async def create_user(user: UserInDB):
    return user
```

### 响应模型过滤

```python
class User(BaseModel):
    username: str
    email: str
    full_name: Optional[str] = None

class UserInDB(User):
    password: str

@app.post("/user/", response_model=User)
async def create_user(user: UserInDB):
    # password 会被自动过滤
    return user
```

### 响应模型配置

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

    class Config:
        schema_extra = {
            "example": {
                "name": "Foo",
                "description": "A very nice Item",
                "price": 35.4,
                "tax": 3.2,
            }
        }

@app.post("/items/", response_model=Item)
async def create_item(item: Item):
    return item
```

### List 响应

```python
from typing import List

@app.get("/items/", response_model=List[Item])
async def read_items():
    return [
        {"name": "Foo", "price": 10.5},
        {"name": "Bar", "price": 20.5},
    ]
```

### 字典响应

```python
from typing import Dict

@app.get("/items/", response_model=Dict[str, float])
async def read_items():
    return {"item1": 10.5, "item2": 20.5}
```

## 4.6 响应状态码

### 设置状态码

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items/", status_code=201)
async def create_item(name: str):
    return {"name": name}

# 使用 status 模块
@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(name: str):
    return {"name": name}
```

### 常用状态码

| 状态码 | 说明 | 常量 |
|--------|------|------|
| 200 | OK | `HTTP_200_OK` |
| 201 | Created | `HTTP_201_CREATED` |
| 204 | No Content | `HTTP_204_NO_CONTENT` |
| 400 | Bad Request | `HTTP_400_BAD_REQUEST` |
| 401 | Unauthorized | `HTTP_401_UNAUTHORIZED` |
| 403 | Forbidden | `HTTP_403_FORBIDDEN` |
| 404 | Not Found | `HTTP_404_NOT_FOUND` |
| 500 | Internal Server Error | `HTTP_500_INTERNAL_SERVER_ERROR` |

## 4.7 自定义响应

### Response 类

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.get("/items/")
async def read_items():
    return Response(
        content='{"message": "Hello World"}',
        media_type="application/json",
        headers={"X-Custom-Header": "value"}
    )
```

### JSONResponse

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/items/")
async def read_items():
    return JSONResponse(
        content={"item_id": "foo", "message": "Hello World"},
        status_code=200,
        headers={"X-Custom-Header": "value"}
    )
```

### HTMLResponse

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.get("/items/", response_class=HTMLResponse)
async def read_items():
    return """
    <html>
        <head>
            <title>Some HTML in here</title>
        </head>
        <body>
            <h1>Look ma! HTML!</h1>
        </body>
    </html>
    """
```

### PlainTextResponse

```python
from fastapi import FastAPI
from fastapi.responses import PlainTextResponse

app = FastAPI()

@app.get("/", response_class=PlainTextResponse)
async def main():
    return "Hello World"
```

### RedirectResponse

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/typer")
async def redirect_typer():
    return RedirectResponse("https://typer.tiangolo.com")
```

### StreamingResponse

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def fake_video_streamer():
    for i in range(10):
        yield b"some fake video bytes"

@app.get("/")
async def main():
    return StreamingResponse(fake_video_streamer())
```

### FileResponse

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse

app = FastAPI()

@app.get("/files/{file_path}")
async def get_file(file_path: str):
    return FileResponse(file_path)
```

## 4.8 Request 对象

### 直接访问 Request

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/info")
async def get_info(request: Request):
    return {
        "method": request.method,
        "url": str(request.url),
        "headers": dict(request.headers),
        "client": request.client.host,
        "cookies": request.cookies,
    }
```

### 读取请求体（原始）

```python
@app.post("/items/")
async def create_item(request: Request):
    body = await request.body()
    return {"body": body.decode()}
```

## 4.9 完整示例：用户管理 API

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List

app = FastAPI(title="用户管理 API")

# 模型
class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    full_name: Optional[str] = None

class UserCreate(UserBase):
    password: str = Field(..., min_length=6)

class User(UserBase):
    id: int
    is_active: bool = True

class UserInDB(User):
    hashed_password: str

# 模拟数据库
users_db = {}

# 密码哈希（简化示例）
def hash_password(password: str) -> str:
    return f"hashed_{password}"

# 路由
@app.post(
    "/users/",
    response_model=User,
    status_code=status.HTTP_201_CREATED,
    summary="创建用户",
    description="创建一个新用户账户",
)
async def create_user(user: UserCreate):
    """创建新用户
    
    - **username**: 用户名（3-50字符）
    - **email**: 有效邮箱地址
    - **password**: 密码（至少6字符）
    - **full_name**: 全名（可选）
    """
    # 检查用户是否存在
    for u in users_db.values():
        if u.username == user.username:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="用户名已存在"
            )
    
    # 创建用户
    user_id = max(users_db.keys(), default=0) + 1
    user_in_db = UserInDB(
        id=user_id,
        username=user.username,
        email=user.email,
        full_name=user.full_name,
        hashed_password=hash_password(user.password),
    )
    users_db[user_id] = user_in_db
    
    # 返回用户（不包含密码）
    return User(**user_in_db.dict())

@app.get("/users/", response_model=List[User])
async def list_users(skip: int = 0, limit: int = 10):
    """获取用户列表"""
    users = [
        User(**user.dict())
        for user in users_db.values()
    ]
    return users[skip : skip + limit]

@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    """获取单个用户"""
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="用户未找到"
        )
    return User(**users_db[user_id].dict())

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    """删除用户"""
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="用户未找到"
        )
    del users_db[user_id]
    return None
```

## 4.10 练习

1. **基础练习**
   - 创建一个产品 API，包含完整的请求和响应
   - 实现图片上传功能
   - 返回不同的状态码

2. **进阶练习**
   - 实现嵌套的请求体模型
   - 添加请求头验证
   - 实现文件下载功能

3. **挑战练习**
   - 实现自定义响应类
   - 创建流式响应 API
   - 实现请求日志记录

## 4.11 小结

- 请求体使用 Pydantic 模型定义
- 可以混合使用路径参数、查询参数和请求体
- 请求头和 Cookie 通过 Header 和 Cookie 获取
- 表单数据使用 Form，文件使用 File
- response_model 自动过滤和验证响应
- 可以自定义响应类型和状态码

## 下一章

[第五章：数据验证与模型](./05_validation.md) - 深入学习 Pydantic 模型和数据验证！
