# SQLAlchemy Patterns - Async Database Access

Complete guide to SQLAlchemy 2.0 async patterns for FastAPI with Supabase (PostgreSQL).

**Note**: This stack uses **Supabase** as the database platform. Supabase is built on PostgreSQL, so all SQLAlchemy patterns work seamlessly. Supabase also provides additional features like built-in auth, storage, and realtime subscriptions that can be used alongside SQLAlchemy.

## Database Setup

### Async Engine Configuration

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base
from app.core.config import settings

# Create async engine
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Base for models
Base = declarative_base()
```

## Model Definitions

### Basic Model

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from app.core.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, nullable=False, index=True)
    username = Column(String, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True, nullable=False)
    is_admin = Column(Boolean, default=False, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### Model with Relationships

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String, nullable=False)
    content = Column(Text, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")

class User(Base):
    __tablename__ = "users"
    # ... columns ...
    posts = relationship("Post", back_populates="author")
```

## Query Patterns

### Basic Queries

```python
from sqlalchemy import select

# Get one
result = await db.execute(select(User).where(User.id == user_id))
user = result.scalar_one_or_none()

# Get all with filter
result = await db.execute(
    select(User).where(User.is_active == True).order_by(User.created_at.desc())
)
users = result.scalars().all()

# Count
result = await db.execute(select(func.count(User.id)))
count = result.scalar_one()
```

### Eager Loading

```python
from sqlalchemy.orm import selectinload

# Load with relationships
result = await db.execute(
    select(Post)
    .where(Post.id == post_id)
    .options(selectinload(Post.author), selectinload(Post.comments))
)
post = result.scalar_one_or_none()
```

## Alembic Migrations

### Setup

```bash
# Initialize Alembic
alembic init alembic

# Create migration
alembic revision --autogenerate -m "Add users table"

# Apply migrations
alembic upgrade head
```

### Migration File

```python
# alembic/versions/xxx_add_users.py
def upgrade() -> None:
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )
```

## Key Takeaways

1. **Use async everywhere** - AsyncSession, async engine
2. **selectinload** for eager loading relationships
3. **Alembic** for database migrations
4. **Type hints** on all queries
5. **Connection pooling** for performance
