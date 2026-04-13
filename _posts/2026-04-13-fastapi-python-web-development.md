---
layout: post
title: "FastAPI vs Django vs Flask：Python Web框架性能对比与实战选择指南"
subtitle: "深入对比三大Python Web框架的架构设计、性能表现、适用场景，提供从Flask迁移到FastAPI的实战代码示例"
date: 2026-04-13
category: python
category_name: 🐍 Python
tags: [FastAPI, Django, Flask, Python Web框架, API开发, 性能对比, 异步编程]
excerpt: "本文系统性对比FastAPI、Django、Flask三大Python Web框架，从性能、扩展性、开发效率等维度分析，并提供FastAPI的完整RESTful API开发实战代码。"
keywords: FastAPI教程, Django vs Flask, Python Web框架, RESTful API, 异步编程, Pydantic, Uvicorn
---

# FastAPI vs Django vs Flask：Python Web框架性能对比与实战选择指南

Python 是 Web API 开发的主流语言之一，而 FastAPI、Django、Flask 是最受欢迎的三个框架。本文将从架构设计、性能、适用场景等维度进行深入对比，并提供 FastAPI 的完整实战代码。

## 三大框架对比

| 特性 | FastAPI | Django | Flask |
|------|---------|--------|-------|
| **诞生年份** | 2018 | 2005 | 2010 |
| **定位** | 现代高性能API | 全栈Web框架 | 轻量微框架 |
| **ASGI支持** | ✅ 原生 | ✅ (Django 3.1+) | ✅ (需扩展) |
| **异步** | ✅ 原生async/await | ⚠️ 部分支持 | ⚠️ 部分支持 |
| **自动文档** | ✅ OpenAPI/Swagger | ❌ 需DRF | ❌ 需扩展 |
| **ORM** | 无（可搭配SQLAlchemy） | ✅ 原生Django ORM | 无 |
| **管理后台** | ❌ 无 | ✅ 原生Admin | ❌ 需扩展 |
| **认证** | 无（可搭配OAuth2） | ✅ 原生Auth | 无 |
| **学习曲线** | 中等 | 陡峭 | 平缓 |
| **TPS（基准）** | ~50,000+ | ~10,000 | ~15,000 |

---

## FastAPI 完整实战

### 1. 项目结构

```bash
fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py              # 应用入口
│   ├── config.py            # 配置管理
│   ├── database.py          # 数据库连接
│   ├── models/              # SQLAlchemy模型
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── product.py
│   ├── schemas/             # Pydantic模型
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── product.py
│   ├── routers/             # 路由
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── products.py
│   ├── services/            # 业务逻辑
│   │   └── user_service.py
│   └── utils/               # 工具函数
│       └── security.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_users.py
│   └── test_products.py
├── alembic/                 # 数据库迁移
│   └── versions/
├── .env
├── .env.example
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 2. 核心代码

```python
# app/main.py
from fastapi import FastAPI, Request, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
import time

from app.config import settings
from app.database import engine, Base
from app.routers import users, products

# 应用生命周期管理
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动时执行
    print(f"🚀 启动 {settings.APP_NAME}...")
    
    # 创建数据库表（开发环境）
    if settings.DEBUG:
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
    
    yield  # 应用在这里运行
    
    # 关闭时执行
    print("👋 关闭应用...")
    await engine.dispose()

# 创建应用
app = FastAPI(
    title=settings.APP_NAME,
    description="高软科技 - FastAPI RESTful API 示例",
    version="1.0.0",
    docs_url="/docs",           # Swagger UI
    redoc_url="/redoc",         # ReDoc
    openapi_url="/openapi.json", # OpenAPI Schema
    lifespan=lifespan
)

# 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 请求日志中间件
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    process_time = (time.time() - start) * 1000
    
    print(f"{request.method} {request.url.path} → {response.status_code} ({process_time:.2f}ms)")
    
    response.headers["X-Process-Time"] = f"{process_time:.2f}ms"
    return response

# 全局异常处理
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "detail": "服务器内部错误",
            "error": str(exc) if settings.DEBUG else None
        }
    )

# 注册路由
app.include_router(users.router, prefix="/api/v1/users", tags=["用户管理"])
app.include_router(products.router, prefix="/api/v1/products", tags=["产品管理"])

# 健康检查
@app.get("/health", tags=["系统"])
async def health_check():
    return {
        "status": "healthy",
        "app": settings.APP_NAME,
        "version": "1.0.0"
    }
```

### 3. Pydantic模型

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, Field, ConfigDict
from typing import Optional, List
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    USER = "user"
    VIP = "vip"

class UserBase(BaseModel):
    email: EmailStr = Field(..., description="用户邮箱")
    username: str = Field(..., min_length=3, max_length=50, description="用户名")
    full_name: Optional[str] = Field(None, max_length=100)
    role: UserRole = Field(default=UserRole.USER)
    is_active: bool = Field(default=True)

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)
    
    model_config = {
        "json_schema_extra": {
            "example": {
                "email": "user@example.com",
                "username": "john_doe",
                "full_name": "John Doe",
                "password": "securepassword123"
            }
        }
    }

class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    full_name: Optional[str] = Field(None, max_length=100)
    password: Optional[str] = Field(None, min_length=8, max_length=100)
    is_active: Optional[bool] = None

class UserResponse(UserBase):
    id: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    
    model_config = ConfigDict(from_attributes=True)

class UserListResponse(BaseModel):
    total: int
    page: int
    page_size: int
    items: List[UserResponse]

class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int

class TokenData(BaseModel):
    user_id: Optional[int] = None
    username: Optional[str] = None
```

### 4. 数据库模型

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, Enum as SQLEnum
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.database import Base
import enum

class UserRoleDB(str, enum.Enum):
    ADMIN = "admin"
    USER = "user"
    VIP = "vip"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, index=True, nullable=False)
    username = Column(String(50), unique=True, index=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    full_name = Column(String(100))
    role = Column(SQLEnum(UserRoleDB), default=UserRoleDB.USER)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # 关系
    products = relationship("Product", back_populates="owner")
    
    def __repr__(self):
        return f"<User {self.username}>"
```

### 5. CRUD服务层

```python
# app/services/user_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload
from typing import List, Optional, Tuple

from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate
from app.utils.security import hash_password, verify_password

class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create(self, user_data: UserCreate) -> User:
        """创建用户"""
        # 检查邮箱唯一性
        result = await self.db.execute(
            select(User).where(User.email == user_data.email)
        )
        if result.scalar_one_or_none():
            raise ValueError("邮箱已被注册")
        
        # 检查用户名唯一性
        result = await self.db.execute(
            select(User).where(User.username == user_data.username)
        )
        if result.scalar_one_or_none():
            raise ValueError("用户名已被使用")
        
        # 创建用户
        db_user = User(
            email=user_data.email,
            username=user_data.username,
            hashed_password=hash_password(user_data.password),
            full_name=user_data.full_name,
            role=user_data.role.value,
        )
        
        self.db.add(db_user)
        await self.db.commit()
        await self.db.refresh(db_user)
        return db_user
    
    async def get_by_id(self, user_id: int) -> Optional[User]:
        """根据ID获取用户"""
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
    
    async def get_by_email(self, email: str) -> Optional[User]:
        """根据邮箱获取用户"""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()
    
    async def get_by_username(self, username: str) -> Optional[User]:
        """根据用户名获取用户"""
        result = await self.db.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()
    
    async def authenticate(self, email: str, password: str) -> Optional[User]:
        """验证用户登录"""
        user = await self.get_by_email(email)
        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None
        return user
    
    async def get_list(
        self,
        page: int = 1,
        page_size: int = 20,
        role: Optional[str] = None,
        search: Optional[str] = None
    ) -> Tuple[List[User], int]:
        """获取用户列表"""
        query = select(User)
        count_query = select(func.count()).select_from(User)
        
        # 过滤条件
        if role:
            query = query.where(User.role == role)
            count_query = count_query.where(User.role == role)
        
        if search:
            search_filter = User.username.ilike(f"%{search}%")
            query = query.where(search_filter)
            count_query = count_query.where(search_filter)
        
        # 总数
        total_result = await self.db.execute(count_query)
        total = total_result.scalar()
        
        # 分页
        offset = (page - 1) * page_size
        query = query.offset(offset).limit(page_size).order_by(User.created_at.desc())
        
        result = await self.db.execute(query)
        users = result.scalars().all()
        
        return users, total
    
    async def update(self, user_id: int, user_data: UserUpdate) -> Optional[User]:
        """更新用户"""
        user = await self.get_by_id(user_id)
        if not user:
            return None
        
        update_data = user_data.model_dump(exclude_unset=True)
        
        if "password" in update_data:
            update_data["hashed_password"] = hash_password(update_data.pop("password"))
        
        for field, value in update_data.items():
            setattr(user, field, value)
        
        await self.db.commit()
        await self.db.refresh(user)
        return user
    
    async def delete(self, user_id: int) -> bool:
        """删除用户"""
        user = await self.get_by_id(user_id)
        if not user:
            return False
        
        await self.db.delete(user)
        await self.db.commit()
        return True
```

### 6. 路由处理

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List

from app.database import get_db
from app.schemas.user import (
    UserCreate, UserUpdate, UserResponse, UserListResponse, Token
)
from app.services.user_service import UserService
from app.utils.security import create_access_token, get_current_user
from app.models.user import User

router = APIRouter()

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    """创建新用户"""
    service = UserService(db)
    try:
        user = await service.create(user_data)
        return user
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))

@router.get("/", response_model=UserListResponse)
async def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    role: str = Query(None),
    search: str = Query(None),
    db: AsyncSession = Depends(get_db)
):
    """获取用户列表"""
    service = UserService(db)
    users, total = await service.get_list(page, page_size, role, search)
    
    return UserListResponse(
        total=total,
        page=page,
        page_size=page_size,
        items=users
    )

@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_user)
):
    """获取当前用户信息"""
    return current_user

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db)
):
    """根据ID获取用户"""
    service = UserService(db)
    user = await service.get_by_id(user_id)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="用户不存在"
        )
    return user

@router.put("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    user_data: UserUpdate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """更新用户（需认证）"""
    # 权限检查：只能修改自己
    if current_user.id != user_id and current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="无权限修改该用户"
        )
    
    service = UserService(db)
    try:
        user = await service.update(user_id, user_data)
        if not user:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="用户不存在")
        return user
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """删除用户（仅管理员）"""
    if current_user.role != "admin":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="仅管理员可删除用户")
    
    service = UserService(db)
    deleted = await service.delete(user_id)
    
    if not deleted:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="用户不存在")
```

### 7. 安全工具

```python
# app/utils/security.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import get_db
from app.models.user import User
from app.services.user_service import UserService

# 密码哈希
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 Scheme
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(
        to_encode, 
        settings.SECRET_KEY, 
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def decode_token(token: str) -> Optional[dict]:
    try:
        payload = jwt.decode(
            token, 
            settings.SECRET_KEY, 
            algorithms=[settings.ALGORITHM]
        )
        return payload
    except JWTError:
        return None

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="无效的认证凭据",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    payload = decode_token(token)
    if payload is None:
        raise credentials_exception
    
    user_id: int = payload.get("sub")
    if user_id is None:
        raise credentials_exception
    
    service = UserService(db)
    user = await service.get_by_id(user_id)
    
    if user is None:
        raise credentials_exception
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="用户已被禁用"
        )
    
    return user
```

---

## 性能对比测试

```python
# benchmark.py
import uvicorn
from fastapi import FastAPI
from fastapi.testclient import TestClient

# 基准测试
def benchmark():
    app = FastAPI()
    
    @app.get("/")
    async def root():
        return {"message": "Hello"}
    
    @app.get("/sync")
    def sync_endpoint():
        return {"data": list(range(100))}
    
    @app.get("/async")
    async def async_endpoint():
        return {"data": list(range(100))}
    
    with TestClient(app) as client:
        import time
        
        # Sync endpoint
        start = time.time()
        for _ in range(1000):
            client.get("/sync")
        sync_time = time.time() - start
        
        # Async endpoint
        start = time.time()
        for _ in range(1000):
            client.get("/async")
        async_time = time.time() - start
        
        print(f"Sync: {sync_time:.3f}s ({1000/sync_time:.0f} req/s)")
        print(f"Async: {async_time:.3f}s ({1000/async_time:.0f} req/s)")

if __name__ == "__main__":
    benchmark()
```

---

## 总结

| 场景 | 推荐框架 |
|------|---------|
| 微服务/API网关 | **FastAPI** |
| 简单脚本/胶水代码 | **Flask** |
| 全栈Web应用（需后台）| **Django** |
| 需要ORM+Admin开箱即用 | **Django** |
| 高并发异步API | **FastAPI** |
| 快速原型验证 | **Flask** |

---

## 💼 需要API开发服务？

河北高软科技提供以下 **Python API开发服务**：

- 🚀 FastAPI/Flask RESTful API开发
- 🔐 JWT认证与OAuth2集成
- 📊 数据库设计与优化
- 🔄 微服务架构设计
- 📱 移动端API定制

**联系方式：13315868800（微信同号）**
