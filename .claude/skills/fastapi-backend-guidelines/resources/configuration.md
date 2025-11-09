# Configuration Management - Pydantic Settings

Complete guide to managing configuration in FastAPI applications.

## Pydantic Settings

### Basic Configuration

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings."""

    # App
    APP_NAME: str = "My API"
    DEBUG: bool = False

    # Database
    DATABASE_URL: str

    # Security
    SECRET_KEY: str
    ALGORITHM: str = "HS256"

    # Sentry
    SENTRY_DSN: str | None = None

    model_config = SettingsConfigDict(
        env_file=".env",
        case_sensitive=True
    )

@lru_cache()
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

### Environment File (.env)

```bash
APP_NAME="My FastAPI App"
DEBUG=true
DATABASE_URL="postgresql+asyncpg://user:pass@localhost/db"
SECRET_KEY="your-secret-key-here"
SENTRY_DSN="https://sentry.io/xxx"
```

## Usage in Application

```python
from app.core.config import settings

# In database setup
engine = create_async_engine(settings.DATABASE_URL)

# In routers
@router.get("/")
async def root():
    return {"app": settings.APP_NAME}

# As dependency
from fastapi import Depends
from app.core.config import Settings, get_settings

@router.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"debug": settings.DEBUG}
```

## Key Takeaways

1. **Use Pydantic Settings** - Type-safe configuration
2. **Environment variables** - Never hardcode secrets
3. **Cache settings** - Use @lru_cache()
4. **Type hints** - On all settings fields
5. **.env files** - For local development
