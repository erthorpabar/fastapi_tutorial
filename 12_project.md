# 第十二章：实战项目

## 12.1 项目概述

### 项目：电商 API 系统

我们将构建一个完整的电商 API 系统，包含以下功能：

- 用户认证与授权
- 商品管理
- 购物车
- 订单管理
- 支付集成
- 评价系统

### 技术栈

- **FastAPI**: Web 框架
- **SQLAlchemy**: ORM
- **PostgreSQL**: 数据库
- **Redis**: 缓存
- **JWT**: 认证
- **Docker**: 容器化

## 12.2 项目结构

```
ecommerce_api/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── dependencies.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   ├── cart.py
│   │   ├── order.py
│   │   └── review.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   ├── cart.py
│   │   ├── order.py
│   │   └── review.py
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── users.py
│   │   ├── products.py
│   │   ├── cart.py
│   │   ├── orders.py
│   │   └── reviews.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── product.py
│   │   ├── cart.py
│   │   └── order.py
│   └── utils/
│       ├── __init__.py
│       ├── security.py
│       ├── cache.py
│       └── exceptions.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_products.py
│   ├── test_cart.py
│   └── test_orders.py
├── alembic/
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env
└── README.md
```

## 12.3 数据库模型

### 用户模型

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, index=True)
    email = Column(String(100), unique=True, index=True)
    hashed_password = Column(String(255))
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # 关系
    cart_items = relationship("CartItem", back_populates="user", cascade="all, delete-orphan")
    orders = relationship("Order", back_populates="user", cascade="all, delete-orphan")
    reviews = relationship("Review", back_populates="user", cascade="all, delete-orphan")
```

### 商品模型

```python
# app/models/product.py
from sqlalchemy import Column, Integer, String, Float, Text, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class Category(Base):
    __tablename__ = "categories"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, index=True)
    description = Column(Text)
    is_active = Column(Boolean, default=True)
    
    products = relationship("Product", back_populates="category")

class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(200), index=True)
    description = Column(Text)
    price = Column(Float, nullable=False)
    stock = Column(Integer, default=0)
    category_id = Column(Integer, ForeignKey("categories.id"))
    image_url = Column(String(500))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    category = relationship("Category", back_populates="products")
    cart_items = relationship("CartItem", back_populates="product")
    order_items = relationship("OrderItem", back_populates="product")
    reviews = relationship("Review", back_populates="product")
```

### 购物车模型

```python
# app/models/cart.py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship
from app.database import Base

class CartItem(Base):
    __tablename__ = "cart_items"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity = Column(Integer, default=1)
    
    user = relationship("User", back_populates="cart_items")
    product = relationship("Product", back_populates="cart_items")
```

### 订单模型

```python
# app/models/order.py
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Enum
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base
import enum

class OrderStatus(str, enum.Enum):
    pending = "pending"
    paid = "paid"
    shipped = "shipped"
    delivered = "delivered"
    cancelled = "cancelled"

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    status = Column(Enum(OrderStatus), default=OrderStatus.pending)
    total_amount = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order", cascade="all, delete-orphan")

class OrderItem(Base):
    __tablename__ = "order_items"
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity = Column(Integer)
    price = Column(Float)  # 下单时的价格
    
    order = relationship("Order", back_populates="items")
    product = relationship("Product", back_populates="order_items")
```

### 评价模型

```python
# app/models/review.py
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from app.database import Base

class Review(Base):
    __tablename__ = "reviews"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    rating = Column(Integer)  # 1-5
    comment = Column(Text)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    user = relationship("User", back_populates="reviews")
    product = relationship("Product", back_populates="reviews")
```

## 12.4 Pydantic Schemas

### 用户 Schemas

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    username: str
    email: EmailStr

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[EmailStr] = None

class User(UserBase):
    id: int
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    user_id: Optional[int] = None
```

### 商品 Schemas

```python
# app/schemas/product.py
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime

class CategoryBase(BaseModel):
    name: str
    description: Optional[str] = None

class Category(CategoryBase):
    id: int
    
    class Config:
        from_attributes = True

class ProductBase(BaseModel):
    name: str = Field(..., min_length=2, max_length=200)
    description: Optional[str] = None
    price: float = Field(..., gt=0)
    stock: int = Field(default=0, ge=0)
    category_id: int
    image_url: Optional[str] = None

class ProductCreate(ProductBase):
    pass

class ProductUpdate(BaseModel):
    name: Optional[str] = Field(None, min_length=2, max_length=200)
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    stock: Optional[int] = Field(None, ge=0)
    category_id: Optional[int] = None
    image_url: Optional[str] = None
    is_active: Optional[bool] = None

class Product(ProductBase):
    id: int
    is_active: bool
    created_at: datetime
    category: Category
    
    class Config:
        from_attributes = True

class ProductList(BaseModel):
    products: List[Product]
    total: int
    page: int
    pages: int
```

### 订单 Schemas

```python
# app/schemas/order.py
from pydantic import BaseModel
from typing import List
from datetime import datetime
from app.models.order import OrderStatus

class OrderItemBase(BaseModel):
    product_id: int
    quantity: int

class OrderItem(OrderItemBase):
    id: int
    price: float
    product_name: str
    
    class Config:
        from_attributes = True

class OrderBase(BaseModel):
    pass

class Order(OrderBase):
    id: int
    status: OrderStatus
    total_amount: float
    created_at: datetime
    items: List[OrderItem]
    
    class Config:
        from_attributes = True

class OrderList(BaseModel):
    orders: List[Order]
    total: int
```

## 12.5 核心路由

### 认证路由

```python
# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from app.schemas.user import Token, UserCreate, User
from app.services.auth import AuthService
from app.dependencies import get_db

router = APIRouter(prefix="/auth", tags=["认证"])

@router.post("/register", response_model=User, status_code=status.HTTP_201_CREATED)
def register(
    user_data: UserCreate,
    db: Session = Depends(get_db)
):
    """用户注册"""
    auth_service = AuthService(db)
    return auth_service.register(user_data)

@router.post("/login", response_model=Token)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """用户登录"""
    auth_service = AuthService(db)
    return auth_service.login(form_data.username, form_data.password)
```

### 商品路由

```python
# app/routers/products.py
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from typing import Optional
from app.schemas.product import (
    Product, ProductCreate, ProductUpdate, ProductList
)
from app.services.product import ProductService
from app.dependencies import get_db, get_current_user
from app.models.user import User

router = APIRouter(prefix="/products", tags=["商品"])

@router.get("/", response_model=ProductList)
def list_products(
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    category_id: Optional[int] = None,
    search: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    db: Session = Depends(get_db)
):
    """获取商品列表"""
    product_service = ProductService(db)
    return product_service.get_products(
        page=page,
        page_size=page_size,
        category_id=category_id,
        search=search,
        min_price=min_price,
        max_price=max_price
    )

@router.get("/{product_id}", response_model=Product)
def get_product(product_id: int, db: Session = Depends(get_db)):
    """获取商品详情"""
    product_service = ProductService(db)
    product = product_service.get_product(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="商品未找到")
    return product

@router.post("/", response_model=Product, status_code=201)
def create_product(
    product_data: ProductCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """创建商品（需要管理员权限）"""
    if not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="权限不足")
    
    product_service = ProductService(db)
    return product_service.create_product(product_data)

@router.put("/{product_id}", response_model=Product)
def update_product(
    product_id: int,
    product_data: ProductUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """更新商品（需要管理员权限）"""
    if not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="权限不足")
    
    product_service = ProductService(db)
    product = product_service.update_product(product_id, product_data)
    if not product:
        raise HTTPException(status_code=404, detail="商品未找到")
    return product

@router.delete("/{product_id}")
def delete_product(
    product_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """删除商品（需要管理员权限）"""
    if not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="权限不足")
    
    product_service = ProductService(db)
    success = product_service.delete_product(product_id)
    if not success:
        raise HTTPException(status_code=404, detail="商品未找到")
    return {"message": "商品已删除"}
```

### 购物车路由

```python
# app/routers/cart.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.schemas.cart import CartItemCreate, CartItemUpdate, CartItem
from app.services.cart import CartService
from app.dependencies import get_db, get_current_user
from app.models.user import User

router = APIRouter(prefix="/cart", tags=["购物车"])

@router.get("/", response_model=list[CartItem])
def get_cart(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """获取购物车"""
    cart_service = CartService(db)
    return cart_service.get_cart(current_user.id)

@router.post("/", response_model=CartItem, status_code=201)
def add_to_cart(
    item_data: CartItemCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """添加到购物车"""
    cart_service = CartService(db)
    return cart_service.add_to_cart(current_user.id, item_data)

@router.put("/{item_id}", response_model=CartItem)
def update_cart_item(
    item_id: int,
    item_data: CartItemUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """更新购物车商品数量"""
    cart_service = CartService(db)
    item = cart_service.update_cart_item(current_user.id, item_id, item_data)
    if not item:
        raise HTTPException(status_code=404, detail="购物车商品未找到")
    return item

@router.delete("/{item_id}")
def remove_from_cart(
    item_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """从购物车移除"""
    cart_service = CartService(db)
    success = cart_service.remove_from_cart(current_user.id, item_id)
    if not success:
        raise HTTPException(status_code=404, detail="购物车商品未找到")
    return {"message": "商品已从购物车移除"}

@router.delete("/")
def clear_cart(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """清空购物车"""
    cart_service = CartService(db)
    cart_service.clear_cart(current_user.id)
    return {"message": "购物车已清空"}
```

### 订单路由

```python
# app/routers/orders.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.schemas.order import Order, OrderList
from app.services.order import OrderService
from app.dependencies import get_db, get_current_user
from app.models.user import User

router = APIRouter(prefix="/orders", tags=["订单"])

@router.post("/", response_model=Order, status_code=201)
def create_order(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """从购物车创建订单"""
    order_service = OrderService(db)
    order = order_service.create_order_from_cart(current_user.id)
    if not order:
        raise HTTPException(status_code=400, detail="购物车为空")
    return order

@router.get("/", response_model=OrderList)
def list_orders(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """获取用户订单列表"""
    order_service = OrderService(db)
    return order_service.get_user_orders(current_user.id)

@router.get("/{order_id}", response_model=Order)
def get_order(
    order_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """获取订单详情"""
    order_service = OrderService(db)
    order = order_service.get_order(order_id, current_user.id)
    if not order:
        raise HTTPException(status_code=404, detail="订单未找到")
    return order

@router.post("/{order_id}/cancel")
def cancel_order(
    order_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """取消订单"""
    order_service = OrderService(db)
    success = order_service.cancel_order(order_id, current_user.id)
    if not success:
        raise HTTPException(status_code=400, detail="无法取消订单")
    return {"message": "订单已取消"}
```

## 12.6 业务逻辑服务

### 认证服务

```python
# app/services/auth.py
from sqlalchemy.orm import Session
from fastapi import HTTPException, status
from app.models.user import User
from app.schemas.user import UserCreate, Token
from app.utils.security import get_password_hash, verify_password, create_access_token

class AuthService:
    def __init__(self, db: Session):
        self.db = db
    
    def register(self, user_data: UserCreate) -> User:
        # 检查用户名
        if self.db.query(User).filter(User.username == user_data.username).first():
            raise HTTPException(status_code=400, detail="用户名已存在")
        
        # 检查邮箱
        if self.db.query(User).filter(User.email == user_data.email).first():
            raise HTTPException(status_code=400, detail="邮箱已注册")
        
        # 创建用户
        user = User(
            username=user_data.username,
            email=user_data.email,
            hashed_password=get_password_hash(user_data.password)
        )
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user
    
    def login(self, username: str, password: str) -> Token:
        # 查找用户
        user = self.db.query(User).filter(
            (User.username == username) | (User.email == username)
        ).first()
        
        if not user or not verify_password(password, user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="用户名或密码错误"
            )
        
        if not user.is_active:
            raise HTTPException(status_code=400, detail="用户已被禁用")
        
        # 创建 token
        access_token = create_access_token(data={"sub": str(user.id)})
        return Token(access_token=access_token, token_type="bearer")
```

### 商品服务

```python
# app/services/product.py
from sqlalchemy.orm import Session
from typing import Optional
from app.models.product import Product, Category
from app.schemas.product import ProductCreate, ProductUpdate, ProductList

class ProductService:
    def __init__(self, db: Session):
        self.db = db
    
    def get_products(
        self,
        page: int = 1,
        page_size: int = 10,
        category_id: Optional[int] = None,
        search: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None
    ) -> ProductList:
        query = self.db.query(Product).filter(Product.is_active == True)
        
        # 过滤条件
        if category_id:
            query = query.filter(Product.category_id == category_id)
        
        if search:
            query = query.filter(Product.name.ilike(f"%{search}%"))
        
        if min_price:
            query = query.filter(Product.price >= min_price)
        
        if max_price:
            query = query.filter(Product.price <= max_price)
        
        # 分页
        total = query.count()
        pages = (total + page_size - 1) // page_size
        products = query.offset((page - 1) * page_size).limit(page_size).all()
        
        return ProductList(
            products=products,
            total=total,
            page=page,
            pages=pages
        )
    
    def get_product(self, product_id: int) -> Optional[Product]:
        return self.db.query(Product).filter(Product.id == product_id).first()
    
    def create_product(self, product_data: ProductCreate) -> Product:
        product = Product(**product_data.dict())
        self.db.add(product)
        self.db.commit()
        self.db.refresh(product)
        return product
    
    def update_product(self, product_id: int, product_data: ProductUpdate) -> Optional[Product]:
        product = self.get_product(product_id)
        if not product:
            return None
        
        update_data = product_data.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(product, field, value)
        
        self.db.commit()
        self.db.refresh(product)
        return product
    
    def delete_product(self, product_id: int) -> bool:
        product = self.get_product(product_id)
        if not product:
            return False
        
        self.db.delete(product)
        self.db.commit()
        return True
```

### 订单服务

```python
# app/services/order.py
from sqlalchemy.orm import Session
from app.models.order import Order, OrderItem, OrderStatus
from app.models.cart import CartItem
from app.models.product import Product
from app.schemas.order import OrderList

class OrderService:
    def __init__(self, db: Session):
        self.db = db
    
    def create_order_from_cart(self, user_id: int) -> Order:
        # 获取购物车
        cart_items = self.db.query(CartItem).filter(CartItem.user_id == user_id).all()
        if not cart_items:
            return None
        
        # 计算总价
        total_amount = 0
        order_items = []
        
        for cart_item in cart_items:
            product = self.db.query(Product).filter(
                Product.id == cart_item.product_id
            ).first()
            
            if not product or product.stock < cart_item.quantity:
                raise ValueError(f"商品 {product.name} 库存不足")
            
            order_item = OrderItem(
                product_id=product.id,
                quantity=cart_item.quantity,
                price=product.price
            )
            order_items.append(order_item)
            total_amount += product.price * cart_item.quantity
            
            # 更新库存
            product.stock -= cart_item.quantity
        
        # 创建订单
        order = Order(
            user_id=user_id,
            total_amount=total_amount,
            status=OrderStatus.pending
        )
        order.items = order_items
        
        self.db.add(order)
        
        # 清空购物车
        self.db.query(CartItem).filter(CartItem.user_id == user_id).delete()
        
        self.db.commit()
        self.db.refresh(order)
        return order
    
    def get_user_orders(self, user_id: int) -> OrderList:
        orders = self.db.query(Order).filter(Order.user_id == user_id).all()
        return OrderList(orders=orders, total=len(orders))
    
    def get_order(self, order_id: int, user_id: int) -> Order:
        return self.db.query(Order).filter(
            Order.id == order_id,
            Order.user_id == user_id
        ).first()
    
    def cancel_order(self, order_id: int, user_id: int) -> bool:
        order = self.get_order(order_id, user_id)
        if not order or order.status != OrderStatus.pending:
            return False
        
        # 恢复库存
        for item in order.items:
            product = self.db.query(Product).filter(Product.id == item.product_id).first()
            if product:
                product.stock += item.quantity
        
        order.status = OrderStatus.cancelled
        self.db.commit()
        return True
```

## 12.7 主应用

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.routers import auth, users, products, cart, orders, reviews

app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    description="电商 API 系统"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 注册路由
app.include_router(auth.router)
app.include_router(users.router)
app.include_router(products.router)
app.include_router(cart.router)
app.include_router(orders.router)
app.include_router(reviews.router)

@app.get("/")
def root():
    return {
        "message": "Welcome to E-Commerce API",
        "docs": "/docs",
        "version": settings.APP_VERSION
    }

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

## 12.8 完整代码仓库

完整的项目代码已上传到 GitHub：

```
https://github.com/yourusername/ecommerce-fastapi
```

### 运行项目

```bash
# 克隆项目
git clone https://github.com/yourusername/ecommerce-fastapi
cd ecommerce-fastapi

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 运行数据库迁移
alembic upgrade head

# 启动服务
uvicorn app.main:app --reload
```

### Docker 部署

```bash
# 构建镜像
docker-compose build

# 启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止服务
docker-compose down
```

## 12.9 功能测试

### 用户注册和登录

```bash
# 注册
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "email": "test@example.com", "password": "password123"}'

# 登录
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=testuser&password=password123"
```

### 商品管理

```bash
# 获取商品列表
curl http://localhost:8000/products/

# 创建商品（需要管理员权限）
curl -X POST http://localhost:8000/products/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Product 1", "price": 99.99, "stock": 100, "category_id": 1}'
```

### 购物车和订单

```bash
# 添加到购物车
curl -X POST http://localhost:8000/cart/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"product_id": 1, "quantity": 2}'

# 创建订单
curl -X POST http://localhost:8000/orders/ \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## 12.10 练习

1. **基础练习**
   - 完成项目的安装和运行
   - 测试所有 API 端点
   - 创建测试数据

2. **进阶练习**
   - 添加商品图片上传功能
   - 实现订单支付集成
   - 添加邮件通知

3. **挑战练习**
   - 实现商品推荐系统
   - 添加实时库存管理
   - 实现分布式订单处理

## 12.11 小结

通过这个实战项目，我们综合运用了：

- 数据库模型设计
- RESTful API 设计
- 用户认证和授权
- 业务逻辑分层
- 错误处理
- 测试编写
- Docker 部署

这是一个完整的、可用于生产环境的项目结构！

## 下一章

[第十三章：最佳实践](./13_best_practices.md) - 学习 FastAPI 的最佳实践！
