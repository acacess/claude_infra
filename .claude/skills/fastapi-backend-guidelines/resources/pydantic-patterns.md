# Pydantic Patterns - FastAPI Validation

Complete guide to using Pydantic for validation, schemas, and configuration in FastAPI.

## Table of Contents

- [Request/Response Schemas](#requestresponse-schemas)
- [Validation](#validation)
- [Pydantic Settings](#pydantic-settings)
- [Model Configuration](#model-configuration)
- [Best Practices](#best-practices)

---

## Request/Response Schemas

### Basic Schema Pattern

```python
from pydantic import BaseModel, Field
from datetime import datetime

class UserBase(BaseModel):
    """Base user schema with shared fields."""
    email: str = Field(..., description="User email address")
    username: str = Field(..., min_length=3, max_length=50)

class UserCreate(UserBase):
    """Schema for user creation."""
    password: str = Field(..., min_length=8, description="User password")

class UserUpdate(BaseModel):
    """Schema for user updates (all optional)."""
    email: str | None = None
    username: str | None = Field(None, min_length=3, max_length=50)
    password: str | None = Field(None, min_length=8)

class UserResponse(UserBase):
    """Schema for user responses (excludes password)."""
    id: int
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True  # Enable ORM mode
```

### Usage in FastAPI

```python
from fastapi import APIRouter, Depends
from app.services.user_service import UserService

router = APIRouter()

@router.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(
    user_data: UserCreate,  # Request validation
    service: UserService = Depends(get_user_service)
) -> UserResponse:  # Response validation
    """Create a new user."""
    return await service.create_user(user_data)
```

---

## Validation

### Field Validators

```python
from pydantic import BaseModel, field_validator, EmailStr
import re

class UserCreate(BaseModel):
    email: EmailStr  # Built-in email validation
    username: str
    password: str
    age: int

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
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        if not any(c in '!@#$%^&*' for c in v):
            raise ValueError('Password must contain special character')
        return v

    @field_validator('age')
    @classmethod
    def age_range(cls, v: int) -> int:
        """Ensure age is in valid range."""
        if v < 13:
            raise ValueError('Must be at least 13 years old')
        if v > 120:
            raise ValueError('Invalid age')
        return v
```

### Model Validators

```python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode='after')
    def check_dates(self) -> 'DateRange':
        """Validate end date is after start date."""
        if self.end_date <= self.start_date:
            raise ValueError('end_date must be after start_date')
        return self
```

### Custom Validators with Dependencies

```python
class PostCreate(BaseModel):
    title: str
    content: str
    category_id: int

    @field_validator('title')
    @classmethod
    def title_not_empty(cls, v: str) -> str:
        """Ensure title is not empty or whitespace."""
        if not v.strip():
            raise ValueError('Title cannot be empty')
        return v.strip()

    @field_validator('content')
    @classmethod
    def content_length(cls, v: str) -> str:
        """Validate content length."""
        if len(v) < 100:
            raise ValueError('Content must be at least 100 characters')
        if len(v) > 10000:
            raise ValueError('Content must not exceed 10,000 characters')
        return v
```

---

## Pydantic Settings

### Configuration Management

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings from environment variables."""

    # App settings
    APP_NAME: str = "My FastAPI App"
    DEBUG: bool = False
    API_V1_PREFIX: str = "/api/v1"

    # Database
    DATABASE_URL: str
    DB_POOL_SIZE: int = 10
    DB_MAX_OVERFLOW: int = 20

    # Security
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # Sentry
    SENTRY_DSN: str | None = None
    SENTRY_ENVIRONMENT: str = "development"

    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True
    )

@lru_cache()
def get_settings() -> Settings:
    """Get cached settings instance."""
    return Settings()

# Usage
settings = get_settings()
```

### Environment-Specific Settings

```python
from pydantic_settings import BaseSettings
import os

class BaseConfig(BaseSettings):
    """Base configuration."""
    APP_NAME: str = "My App"
    DEBUG: bool = False

class DevelopmentConfig(BaseConfig):
    """Development configuration."""
    DEBUG: bool = True
    # For Supabase: "postgresql+asyncpg://postgres:[PASSWORD]@[PROJECT_REF].supabase.co:5432/postgres"
    DATABASE_URL: str = "postgresql://localhost/dev_db"

class ProductionConfig(BaseConfig):
    """Production configuration."""
    DEBUG: bool = False
    # For Supabase: Use connection string from environment variable
    DATABASE_URL: str  # Must be provided via env

class TestConfig(BaseConfig):
    """Test configuration."""
    TESTING: bool = True
    # For Supabase: Use test project connection string
    DATABASE_URL: str = "postgresql://localhost/test_db"

def get_config() -> BaseConfig:
    """Get config based on environment."""
    env = os.getenv("APP_ENV", "development")
    configs = {
        "development": DevelopmentConfig,
        "production": ProductionConfig,
        "test": TestConfig
    }
    return configs[env]()
```

---

## Model Configuration

### ORM Mode (from_attributes)

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base
from pydantic import BaseModel

Base = declarative_base()

class UserModel(Base):
    """SQLAlchemy model."""
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    username = Column(String)

class UserResponse(BaseModel):
    """Pydantic schema."""
    id: int
    email: str
    username: str

    class Config:
        from_attributes = True  # Enable ORM mode

# Usage
user_orm = UserModel(id=1, email="test@example.com", username="testuser")
user_pydantic = UserResponse.model_validate(user_orm)
```

### JSON Schema Customization

```python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    email: str = Field(
        ...,
        description="User email address",
        examples=["user@example.com"]
    )
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        pattern="^[a-zA-Z0-9_]+$",
        description="Username (alphanumeric and underscore only)",
        examples=["john_doe"]
    )
    age: int = Field(
        ...,
        ge=13,
        le=120,
        description="User age",
        examples=[25]
    )

    class Config:
        json_schema_extra = {
            "example": {
                "email": "john@example.com",
                "username": "john_doe",
                "age": 30
            }
        }
```

---

## Best Practices

### 1. Separate Create/Update/Response Schemas

```python
# ✅ GOOD: Separate schemas for different operations
class PostBase(BaseModel):
    title: str
    content: str

class PostCreate(PostBase):
    pass

class PostUpdate(BaseModel):
    title: str | None = None
    content: str | None = None

class PostResponse(PostBase):
    id: int
    author_id: int
    created_at: datetime
    class Config:
        from_attributes = True

# ❌ BAD: Single schema for everything
class Post(BaseModel):
    id: int | None = None  # Confusing - is this create or response?
    title: str
    content: str
    author_id: int | None = None
```

### 2. Use Field() for Constraints

```python
# ✅ GOOD: Explicit constraints with Field()
class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=50)
    age: int = Field(ge=13, le=120)
    email: EmailStr = Field(description="User email")

# ❌ BAD: No validation
class UserCreate(BaseModel):
    username: str
    age: int
    email: str
```

### 3. Use Validators for Complex Logic

```python
# ✅ GOOD: Validator for complex logic
class PaymentCreate(BaseModel):
    amount: float
    currency: str

    @field_validator('amount')
    @classmethod
    def amount_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError('Amount must be positive')
        if v > 100000:
            raise ValueError('Amount exceeds maximum')
        return round(v, 2)  # Round to 2 decimal places

# ❌ BAD: No validation for business rules
class PaymentCreate(BaseModel):
    amount: float
    currency: str
```

### 4. Use Type Hints Consistently

```python
# ✅ GOOD: Clear type hints
from typing import Any

class UserResponse(BaseModel):
    id: int
    metadata: dict[str, Any]
    tags: list[str]
    profile: dict[str, Any] | None = None

# ❌ BAD: Vague or missing types
class UserResponse(BaseModel):
    id: int
    metadata: dict  # What are the keys/values?
    tags: list  # List of what?
    profile = None  # What type when not None?
```

### 5. Leverage Pydantic Settings

```python
# ✅ GOOD: Pydantic Settings for config
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    SECRET_KEY: str
    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()

# ❌ BAD: Manual env var loading
import os
DATABASE_URL = os.getenv("DATABASE_URL")
SECRET_KEY = os.getenv("SECRET_KEY")
```

---

## Key Takeaways

1. **Use separate schemas** for Create/Update/Response operations
2. **Field() constraints** for simple validation
3. **Validators** for complex business rules
4. **Pydantic Settings** for configuration management
5. **from_attributes = True** for SQLAlchemy integration
6. **Type hints** on all fields
7. **Model validators** for cross-field validation
