---
name: fastapi-backend-guidelines
description: Comprehensive backend development guide for Python 3.12/FastAPI/SQLAlchemy microservices. Use when creating routes, dependencies, services, repositories, middleware, or working with FastAPI APIs, SQLAlchemy database access, Sentry error tracking, Pydantic validation, async patterns, and dependency injection. Covers layered architecture (routers → dependencies → services → repositories), error handling, performance monitoring, testing strategies, and best practices.
---

# FastAPI Backend Development Guidelines

## Purpose

Establish consistency and best practices across FastAPI backend services using modern Python 3.12, async patterns, SQLAlchemy 2.0, and dependency injection.

## When to Use This Skill

Automatically activates when working on:
- Creating or modifying FastAPI routes, endpoints, APIs
- Building dependencies, services, repositories
- Implementing middleware (auth, validation, error handling)
- Database operations with SQLAlchemy
- Error tracking with Sentry
- Input validation with Pydantic
- Configuration management
- Backend testing and refactoring

---

## Quick Start

### New Backend Feature Checklist

- [ ] **Router**: Clean definition in `app/routers/`
- [ ] **Dependencies**: Use FastAPI `Depends()` pattern
- [ ] **Service**: Business logic with dependency injection
- [ ] **Repository**: Database access with SQLAlchemy
- [ ] **Validation**: Pydantic models for request/response
- [ ] **Sentry**: Error tracking integration
- [ ] **Tests**: Unit + integration tests with pytest
- [ ] **Config**: Use Pydantic Settings

### New Microservice Checklist

- [ ] Directory structure (see [architecture-overview.md](resources/architecture-overview.md))
- [ ] Sentry SDK initialization
- [ ] Pydantic Settings configuration
- [ ] SQLAlchemy async engine setup
- [ ] FastAPI dependency injection setup
- [ ] Middleware stack (CORS, auth, logging)
- [ ] Error handlers
- [ ] Alembic migrations
- [ ] Testing framework (pytest + pytest-asyncio)

---

## Architecture Overview

### Layered Architecture

```
HTTP Request
    ↓
FastAPI Router (routing only)
    ↓
Dependencies (auth, validation)
    ↓
Services (business logic)
    ↓
Repositories (data access)
    ↓
Database (SQLAlchemy)
```

**Key Principle:** Each layer has ONE responsibility.

See [architecture-overview.md](resources/architecture-overview.md) for complete details.

---

## Directory Structure

```
service/
├── app/
│   ├── routers/           # FastAPI routers
│   ├── dependencies/      # Dependency injection
│   ├── services/          # Business logic
│   ├── repositories/      # Data access layer
│   ├── models/            # SQLAlchemy models
│   ├── schemas/           # Pydantic schemas
│   ├── middleware/        # FastAPI middleware
│   ├── core/              # Config, security, database
│   │   ├── config.py      # Pydantic Settings
│   │   ├── database.py    # SQLAlchemy setup
│   │   └── security.py    # Auth utilities
│   ├── utils/             # Utilities
│   └── main.py            # FastAPI app
├── tests/                 # pytest tests
├── alembic/               # Database migrations
├── alembic.ini            # Alembic config
├── pyproject.toml         # Dependencies
└── README.md
```

**Naming Conventions:**
- Routers: `snake_case` - `user_router.py`
- Services: `snake_case` - `user_service.py`
- Repositories: `snake_case` - `user_repository.py`
- Models: `PascalCase` - `User` (class name)
- Schemas: `PascalCase` - `UserCreate`, `UserResponse`

---

## Core Principles (8 Key Rules)

### 1. Routers Only Route, Dependencies Handle Logic Flow

```python
# ❌ NEVER: Business logic in routers
@router.post("/submit")
async def submit(data: dict):
    # 200 lines of logic
    pass

# ✅ ALWAYS: Delegate to service via dependency
@router.post("/submit")
async def submit(
    data: SubmissionCreate,
    service: SubmissionService = Depends(get_submission_service)
) -> SubmissionResponse:
    return await service.create_submission(data)
```

### 2. Use Dependency Injection for Everything

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
) -> UserResponse:
    return await user_service.get_user(db, user_id)
```

### 3. All Errors to Sentry

```python
import sentry_sdk

try:
    await operation()
except Exception as e:
    sentry_sdk.capture_exception(e)
    raise HTTPException(status_code=500, detail="Internal server error")
```

### 4. Use Pydantic Settings, NEVER os.getenv

```python
# ❌ NEVER
import os
timeout = os.getenv("TIMEOUT_MS")

# ✅ ALWAYS
from app.core.config import settings

timeout = settings.timeout_ms
```

### 5. Validate All Input/Output with Pydantic

```python
from pydantic import BaseModel, EmailStr, field_validator

class UserCreate(BaseModel):
    email: EmailStr
    username: str

    @field_validator('username')
    @classmethod
    def validate_username(cls, v: str) -> str:
        if len(v) < 3:
            raise ValueError('Username too short')
        return v
```

### 6. Use Repository Pattern for Data Access

```python
# Service → Repository → Database
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, user_id: int) -> User | None:
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
```

### 7. Async/Await for All I/O Operations

```python
# ✅ ALWAYS use async for I/O
async def get_user(db: AsyncSession, user_id: int) -> User:
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()
```

### 8. Comprehensive Testing with pytest

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post(
        "/users/",
        json={"email": "test@example.com", "username": "testuser"}
    )
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"
```

---

## Common Imports

```python
# FastAPI
from fastapi import FastAPI, APIRouter, Depends, HTTPException, status
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware

# Pydantic
from pydantic import BaseModel, Field, EmailStr, field_validator
from pydantic_settings import BaseSettings

# SQLAlchemy
from sqlalchemy import select, insert, update, delete
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import declarative_base, sessionmaker

# Sentry
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

# Typing
from typing import AsyncGenerator, Optional, List
```

---

## Quick Reference

### HTTP Status Codes

| Code | Use Case | FastAPI Constant |
|------|----------|------------------|
| 200 | Success | `status.HTTP_200_OK` |
| 201 | Created | `status.HTTP_201_CREATED` |
| 204 | No Content | `status.HTTP_204_NO_CONTENT` |
| 400 | Bad Request | `status.HTTP_400_BAD_REQUEST` |
| 401 | Unauthorized | `status.HTTP_401_UNAUTHORIZED` |
| 403 | Forbidden | `status.HTTP_403_FORBIDDEN` |
| 404 | Not Found | `status.HTTP_404_NOT_FOUND` |
| 422 | Validation Error | `status.HTTP_422_UNPROCESSABLE_ENTITY` |
| 500 | Server Error | `status.HTTP_500_INTERNAL_SERVER_ERROR` |

---

## Anti-Patterns to Avoid

❌ Business logic in routers
❌ Direct `os.getenv()` usage
❌ Missing error handling
❌ No input validation
❌ Sync functions for I/O operations
❌ Direct SQLAlchemy queries in routers
❌ Missing type hints
❌ `print()` instead of logging/Sentry

---

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](resources/architecture-overview.md) |
| Create routers/endpoints | [routers-and-dependencies.md](resources/routers-and-dependencies.md) |
| Organize business logic | [services-and-repositories.md](resources/services-and-repositories.md) |
| Validate input/output | [pydantic-patterns.md](resources/pydantic-patterns.md) |
| Add error tracking | [sentry-and-monitoring.md](resources/sentry-and-monitoring.md) |
| Create middleware | [middleware-guide.md](resources/middleware-guide.md) |
| Database access | [sqlalchemy-patterns.md](resources/sqlalchemy-patterns.md) |
| Manage config | [configuration.md](resources/configuration.md) |
| Handle async/errors | [async-and-errors.md](resources/async-and-errors.md) |
| Write tests | [testing-guide.md](resources/testing-guide.md) |
| See examples | [complete-examples.md](resources/complete-examples.md) |

---

## Resource Files

### [architecture-overview.md](resources/architecture-overview.md)
Layered architecture, request lifecycle, separation of concerns

### [routers-and-dependencies.md](resources/routers-and-dependencies.md)
FastAPI routers, dependency injection patterns, path operations

### [services-and-repositories.md](resources/services-and-repositories.md)
Service layer patterns, repository pattern, dependency injection

### [pydantic-patterns.md](resources/pydantic-patterns.md)
Pydantic models, validators, settings, response schemas

### [sentry-and-monitoring.md](resources/sentry-and-monitoring.md)
Sentry SDK initialization, error capture, performance monitoring

### [middleware-guide.md](resources/middleware-guide.md)
Auth middleware, CORS, logging, request context

### [sqlalchemy-patterns.md](resources/sqlalchemy-patterns.md)
Async SQLAlchemy, models, relationships, migrations with Alembic

### [configuration.md](resources/configuration.md)
Pydantic Settings, environment configs, secrets management

### [async-and-errors.md](resources/async-and-errors.md)
Async patterns, error handling, custom exceptions

### [testing-guide.md](resources/testing-guide.md)
pytest, async testing, fixtures, test database

### [complete-examples.md](resources/complete-examples.md)
Full working examples, end-to-end implementations

---

## Related Skills

- **reflex-frontend-guidelines** - Frontend patterns that consume these APIs
- **error-tracking** - Sentry integration patterns
- **skill-developer** - Meta-skill for creating and managing skills

---

**Skill Status**: COMPLETE ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 11 resource files ✅
