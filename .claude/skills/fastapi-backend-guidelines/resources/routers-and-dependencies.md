# Routers and Dependencies - FastAPI Best Practices

Complete guide to FastAPI router organization and dependency injection patterns.

## Table of Contents

- [Router Organization](#router-organization)
- [Dependency Injection Patterns](#dependency-injection-patterns)
- [Path Operations](#path-operations)
- [Request/Response Models](#requestresponse-models)
- [Error Handling](#error-handling)

---

## Router Organization

### Clean Router Pattern

**Routers should ONLY:**
- ✅ Define HTTP endpoints (path, method)
- ✅ Declare dependencies with `Depends()`
- ✅ Specify request/response models
- ✅ Set status codes and tags
- ✅ Delegate to services

**Routers should NEVER:**
- ❌ Contain business logic
- ❌ Access database directly
- ❌ Implement complex validation
- ❌ Format complex responses

### Example Router

```python
# app/routers/user_router.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.dependencies.auth import get_current_user, get_current_admin
from app.dependencies.services import get_user_service
from app.schemas.user import UserCreate, UserUpdate, UserResponse
from app.models.user import User
from app.services.user_service import UserService

router = APIRouter(
    prefix="/users",
    tags=["users"]
)

@router.post(
    "/",
    status_code=status.HTTP_201_CREATED,
    response_model=UserResponse
)
async def create_user(
    user_data: UserCreate,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_admin)
) -> UserResponse:
    """
    Create a new user (admin only).

    Requires admin authentication.
    """
    return await service.create_user(user_data, current_user)


@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
) -> UserResponse:
    """Get user by ID."""
    return await service.get_user(user_id)


@router.get("/", response_model=list[UserResponse])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
) -> list[UserResponse]:
    """List all users with pagination."""
    return await service.list_users(skip, limit)


@router.put("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    user_data: UserUpdate,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
) -> UserResponse:
    """Update user details."""
    return await service.update_user(user_id, user_data, current_user)


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_admin)
) -> None:
    """Delete user (admin only)."""
    await service.delete_user(user_id)
```

**Key Points:**
- Each endpoint: method, path, dependencies, schemas
- Clear docstrings for API documentation
- Response models for automatic validation
- Consistent parameter ordering (path params, body, dependencies)

---

## Dependency Injection Patterns

### Database Session Dependency

```python
# app/dependencies/database.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import AsyncSessionLocal

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """
    Provide database session.

    Automatically commits on success, rolls back on error.
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### Authentication Dependencies

```python
# app/dependencies/auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import settings
from app.dependencies.database import get_db
from app.repositories.user_repository import UserRepository
from app.models.user import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Get current authenticated user from JWT token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user_repo = UserRepository(db)
    user = await user_repo.get_by_id(int(user_id))
    if user is None:
        raise credentials_exception

    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """Ensure user is active (not disabled)."""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )
    return current_user


async def get_current_admin(
    current_user: User = Depends(get_current_active_user)
) -> User:
    """Ensure user has admin privileges."""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user
```

### Service Dependencies

```python
# app/dependencies/services.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.services.user_service import UserService
from app.services.post_service import PostService

def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    """Provide UserService instance."""
    return UserService(db)

def get_post_service(db: AsyncSession = Depends(get_db)) -> PostService:
    """Provide PostService instance."""
    return PostService(db)
```

### Dependency Chains

```python
# Dependencies can depend on other dependencies

# 1. Database session
async def get_db() -> AsyncSession:
    ...

# 2. Current user (needs DB)
async def get_current_user(
    db: AsyncSession = Depends(get_db)
) -> User:
    ...

# 3. Admin user (needs current user)
async def get_admin_user(
    current_user: User = Depends(get_current_user)
) -> User:
    ...

# 4. Post service (needs DB)
def get_post_service(
    db: AsyncSession = Depends(get_db)
) -> PostService:
    ...

# Usage: FastAPI resolves the entire chain
@router.post("/posts/")
async def create_post(
    post_data: PostCreate,
    admin: User = Depends(get_admin_user),  # Triggers entire auth chain
    service: PostService = Depends(get_post_service)  # Gets DB session
) -> PostResponse:
    return await service.create_post(post_data, admin)
```

---

## Path Operations

### HTTP Methods

```python
# GET - Retrieve data
@router.get("/items/{item_id}")
async def get_item(item_id: int) -> ItemResponse:
    ...

# POST - Create new resource
@router.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(item: ItemCreate) -> ItemResponse:
    ...

# PUT - Update entire resource
@router.put("/items/{item_id}")
async def update_item(item_id: int, item: ItemUpdate) -> ItemResponse:
    ...

# PATCH - Partial update
@router.patch("/items/{item_id}")
async def partial_update(item_id: int, item: ItemPartialUpdate) -> ItemResponse:
    ...

# DELETE - Remove resource
@router.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int) -> None:
    ...
```

### Query Parameters

```python
@router.get("/items/")
async def list_items(
    skip: int = 0,
    limit: int = 100,
    search: str | None = None,
    is_active: bool = True,
    service: ItemService = Depends(get_item_service)
) -> list[ItemResponse]:
    """
    List items with filtering and pagination.

    - skip: Number of items to skip (offset)
    - limit: Max number of items to return
    - search: Search term for item name
    - is_active: Filter by active status
    """
    return await service.list_items(skip, limit, search, is_active)
```

### Request Body

```python
from pydantic import BaseModel

class ItemCreate(BaseModel):
    name: str
    description: str
    price: float
    is_active: bool = True

@router.post("/items/")
async def create_item(
    item: ItemCreate,  # Automatically parsed from JSON body
    service: ItemService = Depends(get_item_service)
) -> ItemResponse:
    return await service.create_item(item)
```

### Path Parameters

```python
from enum import Enum

class ItemType(str, Enum):
    BOOK = "book"
    ELECTRONICS = "electronics"
    CLOTHING = "clothing"

@router.get("/items/{item_type}/{item_id}")
async def get_item_by_type(
    item_type: ItemType,  # Enum validation
    item_id: int,  # Type validation
    service: ItemService = Depends(get_item_service)
) -> ItemResponse:
    return await service.get_item(item_id, item_type)
```

---

## Request/Response Models

### Pydantic Response Models

```python
from pydantic import BaseModel, Field
from datetime import datetime

class UserResponse(BaseModel):
    """User response schema (excludes password)."""
    id: int
    email: str
    username: str
    is_active: bool
    is_admin: bool
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True  # Enable ORM mode


class UserListResponse(BaseModel):
    """Paginated user list response."""
    total: int
    items: list[UserResponse]
    skip: int
    limit: int


# Usage in router
@router.get("/users/", response_model=UserListResponse)
async def list_users(
    skip: int = 0,
    limit: int = 100,
    service: UserService = Depends(get_user_service)
) -> UserListResponse:
    """List users with pagination metadata."""
    users, total = await service.list_users_with_count(skip, limit)
    return UserListResponse(
        total=total,
        items=users,
        skip=skip,
        limit=limit
    )
```

### Request Models with Validation

```python
from pydantic import BaseModel, EmailStr, field_validator

class UserCreate(BaseModel):
    """User creation schema."""
    email: EmailStr  # Email validation
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=8)

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        """Ensure username is alphanumeric."""
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v

    @field_validator('password')
    @classmethod
    def password_strength(cls, v: str) -> str:
        """Validate password strength."""
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v
```

---

## Error Handling

### HTTPException

```python
from fastapi import HTTPException, status

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    """Get user by ID."""
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )
    return user
```

### Custom Exception Handlers

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class CustomException(Exception):
    """Custom business exception."""
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code

@app.exception_handler(CustomException)
async def custom_exception_handler(
    request: Request,
    exc: CustomException
) -> JSONResponse:
    """Handle custom exceptions."""
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.message}
    )
```

---

## Router Registration

```python
# app/main.py
from fastapi import FastAPI
from app.routers import user_router, post_router, auth_router

app = FastAPI(
    title="My API",
    description="API with clean architecture",
    version="1.0.0"
)

# Include routers
app.include_router(auth_router.router)
app.include_router(user_router.router)
app.include_router(post_router.router)
```

---

## Best Practices Summary

1. **Keep routers thin** - Only routing logic
2. **Use dependency injection** - For all shared resources
3. **Response models** - Always specify for documentation
4. **Type hints** - On all parameters and returns
5. **Docstrings** - For API documentation
6. **Status codes** - Use FastAPI constants
7. **Validation** - In Pydantic models, not routers
8. **Error handling** - HTTPException for HTTP errors
