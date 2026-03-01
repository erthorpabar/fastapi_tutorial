# 第七章：数据库集成

## 7.1 SQLAlchemy 简介

### 安装

```bash
pip install sqlalchemy
pip install databases  # 异步数据库支持
pip install asyncpg    # PostgreSQL 异步驱动
pip install aiosqlite  # SQLite 异步驱动
```

## 7.2 数据库配置

### 同步配置

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# 数据库 URL
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
# PostgreSQL: postgresql://user:password@localhost/dbname
# MySQL: mysql://user:password@localhost/dbname

# 创建引擎
engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}  # SQLite 需要
)

# 创建会话
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 创建基类
Base = declarative_base()
```

### 异步配置

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# 异步数据库 URL
SQLALCHEMY_DATABASE_URL = "sqlite+aiosqlite:///./test.db"
# PostgreSQL: postgresql+asyncpg://user:password@localhost/dbname

# 创建异步引擎
engine = create_async_engine(SQLALCHEMY_DATABASE_URL, echo=True)

# 创建异步会话
AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

Base = declarative_base()
```

## 7.3 数据库模型

### 定义模型

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, index=True)
    email = Column(String(100), unique=True, index=True)
    hashed_password = Column(String(255))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.now)
    
    # 关系
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(String(5000), nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.now)
    updated_at = Column(DateTime, default=datetime.now, onupdate=datetime.now)
    
    # 关系
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post")

class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True)
    content = Column(String(1000), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"))
    author_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.now)
    
    # 关系
    post = relationship("Post", back_populates="comments")
    author = relationship("User")
```

### 创建表

```python
# 同步
Base.metadata.create_all(bind=engine)

# 异步
async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

## 7.4 数据库会话依赖

### 同步会话

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

app = FastAPI()

# 依赖
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/")
def read_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
```

### 异步会话

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

# 依赖
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()

@app.get("/users/")
async def read_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()
    return users
```

## 7.5 CRUD 操作

### Create（创建）

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel

app = FastAPI()

# Pydantic 模型
class UserCreate(BaseModel):
    username: str
    email: str
    password: str

# 创建用户
@app.post("/users/")
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # 检查用户是否存在
    db_user = db.query(User).filter(User.email == user.email).first()
    if db_user:
        raise HTTPException(status_code=400, detail="邮箱已注册")
    
    # 创建用户
    hashed_password = hash_password(user.password)
    db_user = User(
        username=user.username,
        email=user.email,
        hashed_password=hashed_password
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

### Read（读取）

```python
from typing import List

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool
    
    class Config:
        from_attributes = True  # Pydantic v2

# 获取所有用户
@app.get("/users/", response_model=List[UserResponse])
def read_users(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    users = db.query(User).offset(skip).limit(limit).all()
    return users

# 获取单个用户
@app.get("/users/{user_id}", response_model=UserResponse)
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="用户未找到")
    return user
```

### Update（更新）

```python
class UserUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[str] = None

@app.put("/users/{user_id}", response_model=UserResponse)
def update_user(
    user_id: int,
    user_update: UserUpdate,
    db: Session = Depends(get_db)
):
    db_user = db.query(User).filter(User.id == user_id).first()
    if db_user is None:
        raise HTTPException(status_code=404, detail="用户未找到")
    
    # 更新字段
    update_data = user_update.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(db_user, field, value)
    
    db.commit()
    db.refresh(db_user)
    return db_user
```

### Delete（删除）

```python
@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    if db_user is None:
        raise HTTPException(status_code=404, detail="用户未找到")
    
    db.delete(db_user)
    db.commit()
    return {"message": "用户已删除"}
```

## 7.6 查询操作

### 基本查询

```python
# 查询所有
users = db.query(User).all()

# 条件查询
user = db.query(User).filter(User.username == "john").first()
users = db.query(User).filter(User.is_active == True).all()

# 多条件
users = db.query(User).filter(
    User.is_active == True,
    User.username.like("%john%")
).all()

# 排序
users = db.query(User).order_by(User.created_at.desc()).all()

# 限制数量
users = db.query(User).limit(10).all()

# 分页
users = db.query(User).offset(0).limit(10).all()
```

### 高级查询

```python
from sqlalchemy import or_, and_, func

# OR 条件
users = db.query(User).filter(
    or_(User.username == "john", User.email == "john@example.com")
).all()

# AND 条件
users = db.query(User).filter(
    and_(User.is_active == True, User.username.like("%j%"))
).all()

# 聚合函数
count = db.query(func.count(User.id)).scalar()

# 分组
from sqlalchemy import func
result = db.query(
    User.is_active,
    func.count(User.id)
).group_by(User.is_active).all()

# 连接查询
posts = db.query(Post).join(User).filter(User.username == "john").all()

# 外连接
posts = db.query(Post).outerjoin(Comment).all()
```

### 异步查询

```python
from sqlalchemy import select

# 基本查询
async def get_users(db: AsyncSession):
    result = await db.execute(select(User))
    return result.scalars().all()

# 条件查询
async def get_user_by_username(db: AsyncSession, username: str):
    result = await db.execute(
        select(User).where(User.username == username)
    )
    return result.scalar_one_or_none()

# 连接查询
async def get_user_posts(db: AsyncSession, user_id: int):
    result = await db.execute(
        select(Post)
        .where(Post.author_id == user_id)
        .options(selectinload(Post.author))
    )
    return result.scalars().all()
```

## 7.7 关系操作

### 一对多关系

```python
# 创建用户和文章
user = User(username="john", email="john@example.com")
db.add(user)
db.commit()

post = Post(title="First Post", content="Hello World", author=user)
db.add(post)
db.commit()

# 查询用户的文章
user = db.query(User).filter(User.id == 1).first()
posts = user.posts  # 自动加载

# 查询文章的作者
post = db.query(Post).first()
author = post.author
```

### 多对多关系

```python
# 关联表
post_tags = Table(
    'post_tags',
    Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id')),
    Column('tag_id', Integer, ForeignKey('tags.id'))
)

class Tag(Base):
    __tablename__ = "tags"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    
    posts = relationship("Post", secondary=post_tags, back_populates="tags")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    
    tags = relationship("Tag", secondary=post_tags, back_populates="posts")

# 使用
post = Post(title="My Post")
tag1 = Tag(name="Python")
tag2 = Tag(name="FastAPI")

post.tags.append(tag1)
post.tags.append(tag2)

db.add(post)
db.commit()
```

## 7.8 完整示例：博客系统

```python
# models.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, index=True)
    email = Column(String(100), unique=True, index=True)
    hashed_password = Column(String(255))
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.now)
    
    posts = relationship("Post", back_populates="author")
    comments = relationship("Comment", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.now)
    updated_at = Column(DateTime, default=datetime.now, onupdate=datetime.now)
    
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post")

class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True)
    content = Column(String(1000), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"))
    author_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.now)
    
    post = relationship("Post", back_populates="comments")
    author = relationship("User", back_populates="comments")
```

```python
# schemas.py
from pydantic import BaseModel, EmailStr
from typing import Optional, List
from datetime import datetime

# User Schemas
class UserBase(BaseModel):
    username: str
    email: EmailStr

class UserCreate(UserBase):
    password: str

class User(UserBase):
    id: int
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True

# Post Schemas
class PostBase(BaseModel):
    title: str
    content: str

class PostCreate(PostBase):
    pass

class Post(PostBase):
    id: int
    author_id: int
    published: bool
    created_at: datetime
    updated_at: datetime
    
    class Config:
        from_attributes = True

class PostWithAuthor(Post):
    author: User

# Comment Schemas
class CommentBase(BaseModel):
    content: str

class CommentCreate(CommentBase):
    pass

class Comment(CommentBase):
    id: int
    post_id: int
    author_id: int
    created_at: datetime
    
    class Config:
        from_attributes = True
```

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from typing import List
import models, schemas
from passlib.context import CryptContext

# 数据库配置
SQLALCHEMY_DATABASE_URL = "sqlite:///./blog.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 创建表
models.Base.metadata.create_all(bind=engine)

app = FastAPI(title="博客系统 API")

# 密码哈希
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def get_password_hash(password):
    return pwd_context.hash(password)

# 依赖
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 用户路由
@app.post("/users/", response_model=schemas.User)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    # 检查邮箱
    db_user = db.query(models.User).filter(models.User.email == user.email).first()
    if db_user:
        raise HTTPException(status_code=400, detail="邮箱已注册")
    
    # 检查用户名
    db_user = db.query(models.User).filter(models.User.username == user.username).first()
    if db_user:
        raise HTTPException(status_code=400, detail="用户名已存在")
    
    # 创建用户
    hashed_password = get_password_hash(user.password)
    db_user = models.User(
        username=user.username,
        email=user.email,
        hashed_password=hashed_password
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.get("/users/", response_model=List[schemas.User])
def read_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    users = db.query(models.User).offset(skip).limit(limit).all()
    return users

@app.get("/users/{user_id}", response_model=schemas.User)
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(models.User).filter(models.User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="用户未找到")
    return user

# 文章路由
@app.post("/posts/", response_model=schemas.Post)
def create_post(post: schemas.PostCreate, author_id: int, db: Session = Depends(get_db)):
    # 检查作者
    author = db.query(models.User).filter(models.User.id == author_id).first()
    if not author:
        raise HTTPException(status_code=404, detail="作者未找到")
    
    db_post = models.Post(**post.dict(), author_id=author_id)
    db.add(db_post)
    db.commit()
    db.refresh(db_post)
    return db_post

@app.get("/posts/", response_model=List[schemas.PostWithAuthor])
def read_posts(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    posts = db.query(models.Post).offset(skip).limit(limit).all()
    return posts

@app.get("/posts/{post_id}", response_model=schemas.PostWithAuthor)
def read_post(post_id: int, db: Session = Depends(get_db)):
    post = db.query(models.Post).filter(models.Post.id == post_id).first()
    if post is None:
        raise HTTPException(status_code=404, detail="文章未找到")
    return post

@app.put("/posts/{post_id}", response_model=schemas.Post)
def update_post(
    post_id: int,
    post_update: schemas.PostCreate,
    author_id: int,
    db: Session = Depends(get_db)
):
    post = db.query(models.Post).filter(models.Post.id == post_id).first()
    if post is None:
        raise HTTPException(status_code=404, detail="文章未找到")
    
    if post.author_id != author_id:
        raise HTTPException(status_code=403, detail="无权修改此文章")
    
    for field, value in post_update.dict().items():
        setattr(post, field, value)
    
    db.commit()
    db.refresh(post)
    return post

@app.delete("/posts/{post_id}")
def delete_post(post_id: int, author_id: int, db: Session = Depends(get_db)):
    post = db.query(models.Post).filter(models.Post.id == post_id).first()
    if post is None:
        raise HTTPException(status_code=404, detail="文章未找到")
    
    if post.author_id != author_id:
        raise HTTPException(status_code=403, detail="无权删除此文章")
    
    db.delete(post)
    db.commit()
    return {"message": "文章已删除"}

# 评论路由
@app.post("/posts/{post_id}/comments/", response_model=schemas.Comment)
def create_comment(
    post_id: int,
    comment: schemas.CommentCreate,
    author_id: int,
    db: Session = Depends(get_db)
):
    # 检查文章
    post = db.query(models.Post).filter(models.Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="文章未找到")
    
    db_comment = models.Comment(
        **comment.dict(),
        post_id=post_id,
        author_id=author_id
    )
    db.add(db_comment)
    db.commit()
    db.refresh(db_comment)
    return db_comment
```

## 7.9 数据库迁移（Alembic）

### 安装和初始化

```bash
pip install alembic
alembic init alembic
```

### 配置

编辑 `alembic/env.py`:

```python
from models import Base
from sqlalchemy import create_engine

# 数据库 URL
config.set_main_option("sqlalchemy.url", "sqlite:///./blog.db")

# 添加模型元数据
target_metadata = Base.metadata
```

### 生成和运行迁移

```bash
# 生成迁移
alembic revision --autogenerate -m "Initial migration"

# 运行迁移
alembic upgrade head

# 回滚
alembic downgrade -1
```

## 7.10 练习

1. **基础练习**
   - 创建一个简单的用户模型
   - 实现 CRUD 操作
   - 使用查询过滤

2. **进阶练习**
   - 创建多表关系
   - 实现分页功能
   - 使用 Alembic 迁移

3. **挑战练习**
   - 实现异步数据库操作
   - 创建复杂的查询
   - 实现数据库事务

## 7.11 小结

- SQLAlchemy 提供强大的 ORM 功能
- 可以使用同步或异步方式操作数据库
- 使用 Pydantic 模型进行数据验证
- 依赖注入管理数据库会话
- 关系映射支持一对多和多对多
- Alembic 用于数据库迁移

## 下一章

[第八章：认证与授权](./08_authentication.md) - 学习如何在 FastAPI 中实现用户认证！
