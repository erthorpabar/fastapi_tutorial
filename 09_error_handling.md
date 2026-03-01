# 第九章：错误处理

## 9.1 HTTPException

### 基本用法

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

items = {"foo": "The Foo Wrestlers", "bar": "The Bar Wrestlers"}

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item": items[item_id]}
```

### 自定义响应头

```python
@app.get("/items-header/{item_id}")
async def read_item_header(item_id: str):
    if item_id not in items:
        raise HTTPException(
            status_code=404,
            detail="Item not found",
            headers={"X-Error": "There goes my error"},
        )
    return {"item": items[item_id]}
```

## 9.2 自定义异常处理

### 自定义异常类

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name

app = FastAPI()

@app.exception_handler(UnicornException)
async def unicorn_exception_handler(request: Request, exc: UnicornException):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} did something. There goes a rainbow..."},
    )

@app.get("/unicorns/{name}")
async def read_unicorn(name: str):
    if name == "yolo":
        raise UnicornException(name=name)
    return {"unicorn_name": name}
```

## 9.3 全局异常处理

### 处理所有异常

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "message": "Internal server error",
            "detail": str(exc)
        }
    )
```

### 处理 HTTP 异常

```python
from fastapi.exceptions import HTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.detail,
            "path": request.url.path
        }
    )
```

## 9.4 请求验证异常

### 自定义验证错误

```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from fastapi.encoders import jsonable_encoder

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content=jsonable_encoder({
            "detail": exc.errors(),
            "body": exc.body,
            "message": "Validation error"
        })
    )
```

### 格式化错误消息

```python
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })
    
    return JSONResponse(
        status_code=422,
        content={
            "success": False,
            "errors": errors
        }
    )
```

## 9.5 常见错误处理

### 404 错误

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    # 模拟数据库查询
    item = get_item_from_db(item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item {item_id} not found"
        )
    return item
```

### 401 未授权

```python
@app.get("/protected")
async def protected_route(token: str = Header(...)):
    if not is_valid_token(token):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return {"message": "Success"}
```

### 403 禁止访问

```python
@app.delete("/users/{user_id}")
async def delete_user(user_id: int, current_user: User = Depends(get_current_user)):
    if current_user.id != user_id and not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    # 删除用户逻辑
    return {"message": "User deleted"}
```

### 400 错误请求

```python
@app.post("/items/")
async def create_item(item: ItemCreate):
    if item.price < 0:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Price cannot be negative"
        )
    # 创建项目逻辑
    return item
```

## 9.6 错误处理最佳实践

### 1. 使用 HTTP 状态码常量

```python
from fastapi import status

# ✅ 推荐
raise HTTPException(status_code=status.HTTP_404_NOT_FOUND)

# ❌ 不推荐
raise HTTPException(status_code=404)
```

### 2. 提供详细的错误信息

```python
# ✅ 好的错误消息
raise HTTPException(
    status_code=400,
    detail={
        "error_code": "INVALID_EMAIL",
        "message": "Email format is invalid",
        "field": "email",
        "value": "invalid-email"
    }
)

# ❌ 不好的错误消息
raise HTTPException(status_code=400, detail="Bad request")
```

### 3. 记录错误日志

```python
import logging

logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Error processing request: {request.url}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"message": "Internal server error"}
    )
```

### 4. 结构化错误响应

```python
from pydantic import BaseModel
from typing import Any, Optional, List

class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    type: str

class ErrorResponse(BaseModel):
    success: bool = False
    error: ErrorDetail
    request_id: Optional[str] = None

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    error_detail = ErrorDetail(
        message=str(exc.detail),
        type="http_error"
    )
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(error=error_detail).dict()
    )
```

## 9.7 完整示例：错误处理系统

```python
from fastapi import FastAPI, Request, HTTPException, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional, List, Any
import logging
import uuid

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="错误处理示例")

# 错误模型
class ErrorDetail(BaseModel):
    code: str
    message: str
    field: Optional[str] = None
    value: Optional[Any] = None

class ErrorResponse(BaseModel):
    success: bool = False
    error: ErrorDetail
    request_id: str
    path: str

# 自定义异常
class AppException(Exception):
    def __init__(
        self,
        status_code: int,
        code: str,
        message: str,
        field: Optional[str] = None,
        value: Optional[Any] = None
    ):
        self.status_code = status_code
        self.code = code
        self.message = message
        self.field = field
        self.value = value

class NotFoundException(AppException):
    def __init__(self, resource: str, resource_id: Any):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            code="NOT_FOUND",
            message=f"{resource} not found",
            field="id",
            value=resource_id
        )

class ValidationException(AppException):
    def __init__(self, field: str, message: str, value: Any = None):
        super().__init__(
            status_code=status.HTTP_400_BAD_REQUEST,
            code="VALIDATION_ERROR",
            message=message,
            field=field,
            value=value
        )

class UnauthorizedException(AppException):
    def __init__(self, message: str = "Unauthorized"):
        super().__init__(
            status_code=status.HTTP_401_UNAUTHORIZED,
            code="UNAUTHORIZED",
            message=message
        )

class ForbiddenException(AppException):
    def __init__(self, message: str = "Forbidden"):
        super().__init__(
            status_code=status.HTTP_403_FORBIDDEN,
            code="FORBIDDEN",
            message=message
        )

# 生成请求 ID
def generate_request_id():
    return str(uuid.uuid4())

# 异常处理器
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    request_id = generate_request_id()
    
    logger.error(
        f"Request {request_id} failed: {exc.code} - {exc.message}",
        extra={
            "request_id": request_id,
            "path": request.url.path,
            "error_code": exc.code,
            "field": exc.field,
            "value": exc.value
        }
    )
    
    error_detail = ErrorDetail(
        code=exc.code,
        message=exc.message,
        field=exc.field,
        value=exc.value
    )
    
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error=error_detail,
            request_id=request_id,
            path=request.url.path
        ).dict()
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    request_id = generate_request_id()
    errors = exc.errors()
    
    logger.warning(
        f"Request {request_id} validation failed",
        extra={
            "request_id": request_id,
            "errors": errors
        }
    )
    
    formatted_errors = []
    for error in errors:
        formatted_errors.append({
            "field": ".".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })
    
    error_detail = ErrorDetail(
        code="VALIDATION_ERROR",
        message="Request validation failed",
        value=formatted_errors
    )
    
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content=ErrorResponse(
            error=error_detail,
            request_id=request_id,
            path=request.url.path
        ).dict()
    )

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    request_id = generate_request_id()
    
    logger.error(
        f"Request {request_id} HTTP error: {exc.status_code}",
        extra={
            "request_id": request_id,
            "status_code": exc.status_code,
            "detail": str(exc.detail)
        }
    )
    
    error_detail = ErrorDetail(
        code="HTTP_ERROR",
        message=str(exc.detail)
    )
    
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error=error_detail,
            request_id=request_id,
            path=request.url.path
        ).dict()
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    request_id = generate_request_id()
    
    logger.exception(
        f"Request {request_id} unexpected error",
        extra={
            "request_id": request_id,
            "path": request.url.path
        }
    )
    
    error_detail = ErrorDetail(
        code="INTERNAL_ERROR",
        message="An unexpected error occurred"
    )
    
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content=ErrorResponse(
            error=error_detail,
            request_id=request_id,
            path=request.url.path
        ).dict()
    )

# 示例路由
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    # 模拟数据库查询
    if item_id == 0:
        raise NotFoundException("Item", item_id)
    
    if item_id < 0:
        raise ValidationException("item_id", "Item ID must be positive", item_id)
    
    return {"item_id": item_id, "name": f"Item {item_id}"}

@app.get("/protected")
async def protected_route(token: str = None):
    if not token:
        raise UnauthorizedException("Token is required")
    
    if token != "valid-token":
        raise ForbiddenException("Invalid token")
    
    return {"message": "Success"}

@app.get("/error")
async def trigger_error():
    # 触发未处理的异常
    raise ValueError("This is an unexpected error")

# 健康检查
@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## 9.8 错误响应示例

### 404 错误

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Item not found",
    "field": "id",
    "value": 0
  },
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "path": "/items/0"
}
```

### 验证错误

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "value": [
      {
        "field": "body.email",
        "message": "invalid email format",
        "type": "value_error.email"
      }
    ]
  },
  "request_id": "550e8400-e29b-41d4-a716-446655440001",
  "path": "/users/"
}
```

## 9.9 练习

1. **基础练习**
   - 创建自定义异常类
   - 实现异常处理器
   - 返回格式化的错误响应

2. **进阶练习**
   - 实现错误日志记录
   - 添加请求追踪
   - 创建错误监控

3. **挑战练习**
   - 实现错误重试机制
   - 创建错误仪表板
   - 集成错误追踪服务（如 Sentry）

## 9.10 小结

- 使用 HTTPException 返回 HTTP 错误
- 自定义异常类实现业务错误
- 全局异常处理器统一错误格式
- 提供详细的错误信息帮助调试
- 记录错误日志便于问题追踪
- 使用合适的状态码表示错误类型

## 下一章

[第十章：测试](./10_testing.md) - 学习如何在 FastAPI 中编写测试！
