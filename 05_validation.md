# 第五章：数据验证与模型

## 5.1 Pydantic 基础

### 什么是 Pydantic？

Pydantic 是一个使用 Python 类型提示进行数据验证和设置管理的库。

### 基本模型

```python
from pydantic import BaseModel
from typing import Optional

class User(BaseModel):
    id: int
    name: str
    email: str
    age: Optional[int] = None
```

### 使用模型

```python
# 创建实例
user = User(id=1, name="John", email="john@example.com", age=30)

# 访问属性
print(user.name)  # John

# 转换为字典
user_dict = user.dict()
# {'id': 1, 'name': 'John', 'email': 'john@example.com', 'age': 30}

# 转换为 JSON
user_json = user.json()
# '{"id": 1, "name": "John", "email": "john@example.com", "age": 30}'
```

### 从字典创建

```python
user_data = {
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30
}

user = User(**user_data)
# 或
user = User.parse_obj(user_data)
```

### 从 JSON 创建

```python
user = User.parse_raw('{"id": 1, "name": "John", "email": "john@example.com"}')
```

## 5.2 字段类型

### 基本类型

```python
from pydantic import BaseModel
from typing import Optional, List, Dict

class Item(BaseModel):
    name: str                    # 字符串
    age: int                     # 整数
    price: float                 # 浮点数
    is_active: bool              # 布尔值
    tags: List[str]              # 字符串列表
    metadata: Dict[str, str]     # 字典
    description: Optional[str]   # 可选字符串
```

### 特殊类型

```python
from datetime import datetime, date, time, timedelta
from pathlib import Path
from uuid import UUID

class AdvancedTypes(BaseModel):
    birthday: date
    meeting_time: time
    created_at: datetime
    duration: timedelta
    file_path: Path
    uuid: UUID
```

### 严格类型

```python
from pydantic import BaseModel, StrictInt, StrictStr

class StrictModel(BaseModel):
    age: StrictInt      # 只接受 int
    name: StrictStr     # 只接受 str
```

## 5.3 字段验证

### 使用 Field

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    price: float = Field(..., gt=0, description="价格必须大于0")
    quantity: int = Field(default=0, ge=0)
    tax: float = Field(default=0.1, le=1.0)
```

### Field 参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `default` | 默认值 | `default=0` |
| `default_factory` | 默认工厂函数 | `default_factory=list` |
| `alias` | 别名 | `alias="item-name"` |
| `title` | 标题 | `title="Item Name"` |
| `description` | 描述 | `description="Item description"` |
| `gt` | 大于 | `gt=0` |
| `ge` | 大于等于 | `ge=1` |
| `lt` | 小于 | `lt=100` |
| `le` | 小于等于 | `le=10` |
| `min_length` | 最小长度 | `min_length=3` |
| `max_length` | 最大长度 | `max_length=50` |
| `regex` | 正则表达式 | `regex="^[a-z]+$"` |
| `example` | 示例 | `example="Item 1"` |
| `deprecated` | 已弃用 | `deprecated=True` |

### 完整示例

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(
        ...,
        min_length=2,
        max_length=100,
        description="产品名称",
        example="iPhone 15"
    )
    price: float = Field(
        ...,
        gt=0,
        le=1000000,
        description="产品价格",
        example=999.99
    )
    quantity: int = Field(
        default=0,
        ge=0,
        description="库存数量",
        example=100
    )
    category: str = Field(
        default="general",
        max_length=50,
        description="产品分类"
    )

# FastAPI 中的使用
@app.post("/products/")
async def create_product(product: Product):
    return product
```

## 5.4 验证器

### 基本验证器

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    name: str
    email: str
    password: str
    confirm_password: str

    @validator('name')
    def name_must_contain_space(cls, v):
        if ' ' not in v:
            raise ValueError('必须包含全名')
        return v

    @validator('email')
    def email_must_contain_at(cls, v):
        if '@' not in v:
            raise ValueError('邮箱格式不正确')
        return v

    @validator('confirm_password')
    def passwords_match(cls, v, values):
        if 'password' in values and v != values['password']:
            raise ValueError('密码不匹配')
        return v
```

### 预验证器

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    name: str
    email: str

    @validator('name', 'email', pre=True)
    def convert_to_lowercase(cls, v):
        if isinstance(v, str):
            return v.lower()
        return v
```

### 根验证器

```python
from pydantic import BaseModel, root_validator

class User(BaseModel):
    password: str
    confirm_password: str

    @root_validator
    def passwords_match(cls, values):
        password = values.get('password')
        confirm_password = values.get('confirm_password')
        if password != confirm_password:
            raise ValueError('密码不匹配')
        return values
```

### 验证器示例

```python
from pydantic import BaseModel, validator
from datetime import date

class Event(BaseModel):
    name: str
    start_date: date
    end_date: date

    @validator('start_date')
    def start_date_must_be_future(cls, v):
        if v < date.today():
            raise ValueError('开始日期必须是未来日期')
        return v

    @validator('end_date')
    def end_date_must_after_start(cls, v, values):
        start_date = values.get('start_date')
        if start_date and v <= start_date:
            raise ValueError('结束日期必须在开始日期之后')
        return v
```

## 5.5 嵌套模型

### 嵌套结构

```python
from pydantic import BaseModel
from typing import List, Optional

class Image(BaseModel):
    url: str
    name: str
    alt_text: Optional[str] = None

class Category(BaseModel):
    name: str
    description: Optional[str] = None

class Product(BaseModel):
    name: str
    price: float
    categories: List[Category]
    images: List[Image]
```

### 使用示例

```python
product = Product(
    name="iPhone",
    price=999.99,
    categories=[
        Category(name="Electronics", description="电子设备"),
        Category(name="Phones")
    ],
    images=[
        Image(url="https://example.com/image1.jpg", name="Front"),
        Image(url="https://example.com/image2.jpg", name="Back")
    ]
)
```

### 自引用模型

```python
class Comment(BaseModel):
    text: str
    replies: List['Comment'] = []

# 更新前向引用
Comment.update_forward_refs()

comment = Comment(
    text="Great article!",
    replies=[
        Comment(text="Thanks!"),
        Comment(text="I agree!", replies=[
            Comment(text="Me too!")
        ])
    ]
)
```

## 5.6 模型配置

### Config 类

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    email: str

    class Config:
        # 允许字段别名
        allow_population_by_field_name = True
        
        # 验证赋值
        validate_assignment = True
        
        # 任意字段
        extra = "forbid"  # 'allow', 'forbid', 'ignore'
        
        # 使用枚举值
        use_enum_values = True
        
        # JSON 编码器
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }
        
        # Schema 示例
        schema_extra = {
            "example": {
                "name": "John Doe",
                "email": "john@example.com"
            }
        }
```

### 示例配置

```python
from pydantic import BaseModel, Field
from datetime import datetime

class Item(BaseModel):
    name: str
    created_at: datetime = Field(default_factory=datetime.now)
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.strftime("%Y-%m-%d %H:%M:%S")
        }
        schema_extra = {
            "example": {
                "name": "Example Item",
                "created_at": "2024-01-01 12:00:00"
            }
        }
```

## 5.7 模型方法

### 常用方法

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    email: str

# 创建实例
user = User(id=1, name="John", email="john@example.com")

# 转换为字典
user_dict = user.dict()
# {'id': 1, 'name': 'John', 'email': 'john@example.com'}

# 转换为 JSON
user_json = user.json()
# '{"id": 1, "name": "John", "email": "john@example.com"}'

# 仅包含某些字段
user_dict = user.dict(include={'id', 'name'})

# 排除某些字段
user_dict = user.dict(exclude={'email'})

# 复制并更新
user_copy = user.copy(update={'name': 'Jane'})

# 解析字典
user = User.parse_obj({'id': 1, 'name': 'John', 'email': 'john@example.com'})

# 解析 JSON
user = User.parse_raw('{"id": 1, "name": "John", "email": "john@example.com"}')
```

### 自定义方法

```python
from pydantic import BaseModel
from typing import Optional

class User(BaseModel):
    id: int
    name: str
    email: str
    full_name: Optional[str] = None

    def get_display_name(self) -> str:
        return self.full_name or self.name
    
    def get_email_domain(self) -> str:
        return self.email.split('@')[1]

# 使用
user = User(id=1, name="john", email="john@example.com", full_name="John Doe")
print(user.get_display_name())      # John Doe
print(user.get_email_domain())      # example.com
```

## 5.8 特殊数据类型

### EmailStr

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    email: EmailStr

user = User(email="john@example.com")
```

### HttpUrl

```python
from pydantic import BaseModel, HttpUrl

class Website(BaseModel):
    url: HttpUrl

site = Website(url="https://example.com")
```

### FilePath 和 DirectoryPath

```python
from pydantic import BaseModel, FilePath, DirectoryPath

class Config(BaseModel):
    config_file: FilePath
    data_dir: DirectoryPath
```

### 颜色类型

```python
from pydantic import BaseModel, color

class Theme(BaseModel):
    primary_color: color.Color
    secondary_color: color.Color

theme = Theme(
    primary_color="#FF5733",
    secondary_color="rgb(100, 200, 150)"
)
```

## 5.9 实际应用示例

### 用户注册

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Optional

app = FastAPI()

class UserRegister(BaseModel):
    username: str = Field(
        ...,
        min_length=3,
        max_length=20,
        regex="^[a-zA-Z0-9_]+$",
        description="用户名（3-20字符，仅字母数字下划线）"
    )
    email: EmailStr
    password: str = Field(
        ...,
        min_length=8,
        max_length=50,
        description="密码（8-50字符）"
    )
    confirm_password: str
    full_name: Optional[str] = Field(None, max_length=100)
    age: Optional[int] = Field(None, ge=13, le=120)

    @validator('username')
    def username_alphanumeric(cls, v):
        if not v.isalnum() and '_' not in v:
            raise ValueError('用户名只能包含字母、数字和下划线')
        return v

    @validator('confirm_password')
    def passwords_match(cls, v, values):
        if 'password' in values and v != values['password']:
            raise ValueError('两次密码不匹配')
        return v

class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
    full_name: Optional[str] = None

@app.post("/register", response_model=UserResponse)
async def register(user: UserRegister):
    # 检查用户是否存在
    # 哈希密码
    # 保存用户
    
    return UserResponse(
        id=1,
        username=user.username,
        email=user.email,
        full_name=user.full_name
    )
```

### 产品管理

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, HttpUrl
from typing import List, Optional
from decimal import Decimal
from datetime import datetime

app = FastAPI()

class ProductBase(BaseModel):
    name: str = Field(..., min_length=2, max_length=100)
    description: Optional[str] = Field(None, max_length=1000)
    price: Decimal = Field(..., gt=0, decimal_places=2)
    quantity: int = Field(default=0, ge=0)
    sku: str = Field(..., regex="^[A-Z]{3}-[0-9]{4}$")
    images: List[HttpUrl] = Field(default_factory=list)
    tags: List[str] = Field(default_factory=list)

class ProductCreate(ProductBase):
    category_id: int

class Product(ProductBase):
    id: int
    created_at: datetime
    updated_at: datetime
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }

@app.post("/products/", response_model=Product)
async def create_product(product: ProductCreate):
    # 创建产品逻辑
    return Product(
        id=1,
        name=product.name,
        description=product.description,
        price=product.price,
        quantity=product.quantity,
        sku=product.sku,
        images=product.images,
        tags=product.tags,
        created_at=datetime.now(),
        updated_at=datetime.now()
    )
```

## 5.10 练习

1. **基础练习**
   - 创建一个包含验证器的用户模型
   - 创建一个嵌套的产品模型
   - 实现自定义验证逻辑

2. **进阶练习**
   - 创建一个订单系统模型（包含多个嵌套模型）
   - 实现复杂的验证逻辑
   - 使用配置优化模型

3. **挑战练习**
   - 实现动态模型验证
   - 创建自定义数据类型
   - 实现模型继承和复用

## 5.11 小结

- Pydantic 提供强大的数据验证功能
- 使用 Field 进行字段级别的验证
- 使用 @validator 创建自定义验证器
- 支持嵌套模型和复杂的数据结构
- Config 类可以配置模型行为
- 提供丰富的内置数据类型

## 下一章

[第六章：依赖注入](./06_dependencies.md) - 学习 FastAPI 的依赖注入系统！
