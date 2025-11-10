# Services and Repositories - FastAPI Best Practices

Complete guide to organizing business logic and data access in FastAPI applications.

## Table of Contents

- [Service Layer](#service-layer)
- [Repository Pattern](#repository-pattern)
- [Dependency Injection](#dependency-injection)
- [Transaction Management](#transaction-management)
- [Best Practices](#best-practices)

---

## Service Layer

### What is a Service?

Services contain **business logic**:
- Orchestrate multiple repositories
- Implement business rules
- Handle complex validations
- Manage transactions
- Call external APIs

### Basic Service Pattern

```python
# app/services/user_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException, status
import sentry_sdk

from app.repositories.user_repository import UserRepository
from app.schemas.user import UserCreate, UserUpdate, UserResponse
from app.core.security import get_password_hash

class UserService:
    """Service for user business logic."""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.user_repo = UserRepository(db)

    async def create_user(self, user_data: UserCreate) -> UserResponse:
        """
        Create a new user.

        Business rules:
        - Email must be unique
        - Password must be hashed
        """
        try:
            # Business rule: check email uniqueness
            existing = await self.user_repo.get_by_email(user_data.email)
            if existing:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Email already registered"
                )

            # Hash password
            hashed_password = get_password_hash(user_data.password)

            # Create via repository
            user = await self.user_repo.create(
                email=user_data.email,
                username=user_data.username,
                hashed_password=hashed_password
            )

            await self.db.commit()
            return UserResponse.model_validate(user)

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to create user"
            )

    async def get_user(self, user_id: int) -> UserResponse:
        """Get user by ID."""
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"User {user_id} not found"
            )
        return UserResponse.model_validate(user)

    async def update_user(
        self,
        user_id: int,
        user_data: UserUpdate
    ) -> UserResponse:
        """Update user with business logic."""
        try:
            user = await self.user_repo.get_by_id(user_id)
            if not user:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=f"User {user_id} not found"
                )

            # Update via repository
            updated_user = await self.user_repo.update(user, user_data)
            await self.db.commit()

            return UserResponse.model_validate(updated_user)

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to update user"
            )
```

### Service Orchestrating Multiple Repositories

```python
class OrderService:
    """Service orchestrating multiple repositories."""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.order_repo = OrderRepository(db)
        self.product_repo = ProductRepository(db)
        self.user_repo = UserRepository(db)

    async def create_order(
        self,
        user_id: int,
        items: list[dict]
    ) -> OrderResponse:
        """
        Create order with business logic across multiple entities.
        """
        try:
            # Validate user exists
            user = await self.user_repo.get_by_id(user_id)
            if not user:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail="User not found"
                )

            # Validate all products exist and calculate total
            total = 0
            order_items = []

            for item in items:
                product = await self.product_repo.get_by_id(item["product_id"])
                if not product:
                    raise HTTPException(
                        status_code=status.HTTP_404_NOT_FOUND,
                        detail=f"Product {item['product_id']} not found"
                    )

                # Business rule: check stock
                if product.stock < item["quantity"]:
                    raise HTTPException(
                        status_code=status.HTTP_400_BAD_REQUEST,
                        detail=f"Insufficient stock for {product.name}"
                    )

                total += product.price * item["quantity"]
                order_items.append({
                    "product_id": product.id,
                    "quantity": item["quantity"],
                    "price": product.price
                })

            # Business rule: minimum order amount
            if total < 10.0:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Minimum order amount is $10"
                )

            # Create order
            order = await self.order_repo.create(
                user_id=user_id,
                total=total,
                items=order_items
            )

            # Update product stock
            for item in order_items:
                await self.product_repo.decrease_stock(
                    item["product_id"],
                    item["quantity"]
                )

            await self.db.commit()
            return OrderResponse.model_validate(order)

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to create order"
            )
```

---

## Repository Pattern

### What is a Repository?

Repositories handle **data access**:
- Execute database queries
- Abstract SQLAlchemy details
- Reusable data access methods
- NO business logic

### Basic Repository Pattern

```python
# app/repositories/user_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete
from sqlalchemy.orm import selectinload

from app.models.user import User
from app.schemas.user import UserUpdate

class UserRepository:
    """Repository for user data access."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, user_id: int) -> User | None:
        """Get user by ID."""
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> User | None:
        """Get user by email."""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_all(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> list[User]:
        """Get all users with pagination."""
        result = await self.db.execute(
            select(User)
            .offset(skip)
            .limit(limit)
            .order_by(User.created_at.desc())
        )
        return list(result.scalars().all())

    async def create(
        self,
        email: str,
        username: str,
        hashed_password: str
    ) -> User:
        """Create new user."""
        user = User(
            email=email,
            username=username,
            hashed_password=hashed_password
        )
        self.db.add(user)
        await self.db.flush()
        await self.db.refresh(user)
        return user

    async def update(
        self,
        user: User,
        user_data: UserUpdate
    ) -> User:
        """Update user fields."""
        update_data = user_data.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            setattr(user, field, value)

        await self.db.flush()
        await self.db.refresh(user)
        return user

    async def delete(self, user: User) -> None:
        """Delete user."""
        await self.db.delete(user)
        await self.db.flush()
```

### Repository with Relationships

```python
class PostRepository:
    """Repository with relationship loading."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id_with_author(self, post_id: int) -> Post | None:
        """Get post with author eagerly loaded."""
        result = await self.db.execute(
            select(Post)
            .where(Post.id == post_id)
            .options(selectinload(Post.author))
        )
        return result.scalar_one_or_none()

    async def get_user_posts_with_comments(
        self,
        user_id: int
    ) -> list[Post]:
        """Get user posts with comments eagerly loaded."""
        result = await self.db.execute(
            select(Post)
            .where(Post.author_id == user_id)
            .options(
                selectinload(Post.comments),
                selectinload(Post.author)
            )
            .order_by(Post.created_at.desc())
        )
        return list(result.scalars().all())
```

---

## Dependency Injection

### Service as Dependency

```python
# app/dependencies/services.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.services.user_service import UserService
from app.services.post_service import PostService

def get_user_service(
    db: AsyncSession = Depends(get_db)
) -> UserService:
    """Dependency for UserService."""
    return UserService(db)

def get_post_service(
    db: AsyncSession = Depends(get_db)
) -> PostService:
    """Dependency for PostService."""
    return PostService(db)


# Usage in router
from app.dependencies.services import get_user_service

@router.post("/users/")
async def create_user(
    user_data: UserCreate,
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    return await service.create_user(user_data)
```

---

## Transaction Management

### Service-Managed Transactions (Recommended)

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

# app/core/database.py
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

# app/dependencies/database.py
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

# Service layer
class ComplexService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_resource(self, data: ResourceCreate) -> ResourceResponse:
        try:
            entity = await self.repo.create(data)
            await self.db.commit()
            await self.db.refresh(entity)
            return ResourceResponse.model_validate(entity)
        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as exc:
            await self.db.rollback()
            sentry_sdk.capture_exception(exc)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to create resource"
            )
```

### Manual Transaction Control

```python
class ComplexService:
    """Service with manual transaction control."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def complex_operation(self, data: dict):
        """Operation requiring manual transaction control."""
        try:
            # Multiple operations in one transaction
            user = await self.user_repo.create(data["user"])
            profile = await self.profile_repo.create(user.id, data["profile"])
            settings = await self.settings_repo.create_default(user.id)

            # Explicit commit
            await self.db.commit()

            return {"user": user, "profile": profile, "settings": settings}

        except Exception as e:
            # Explicit rollback
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise
```

---

## Best Practices

### 1. Services Handle Business Logic, Repositories Handle Data

```python
# ✅ GOOD: Business logic in service
class PostService:
    async def publish_post(self, post_id: int, user: User):
        """Publish post with business rules."""
        post = await self.repo.get_by_id(post_id)

        # Business rules in service
        if post.author_id != user.id and not user.is_admin:
            raise HTTPException(403, "Not authorized")

        if len(post.content) < 100:
            raise HTTPException(400, "Post too short to publish")

        # Data access via repository
        return await self.repo.update_published(post, True)

# ❌ BAD: Business logic in repository
class PostRepository:
    async def publish_post(self, post_id: int, user: User):
        # Don't put business logic here!
        if post.author_id != user.id and not user.is_admin:
            raise HTTPException(403, "Not authorized")
```

### 2. Always Use Type Hints

```python
# ✅ GOOD: Full type hints
class UserService:
    def __init__(self, db: AsyncSession) -> None:
        self.db = db
        self.repo = UserRepository(db)

    async def get_user(self, user_id: int) -> UserResponse:
        user = await self.repo.get_by_id(user_id)
        return UserResponse.model_validate(user)
```

### 3. Handle Errors Appropriately

```python
# ✅ GOOD: Proper error handling
class UserService:
    async def create_user(self, data: UserCreate) -> UserResponse:
        try:
            user = await self.repo.create(data)
            await self.db.commit()
            return UserResponse.model_validate(user)
        except HTTPException:
            await self.db.rollback()
            raise  # Re-raise HTTP exceptions
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)  # Log unexpected errors
            raise HTTPException(500, "Internal error")
```

### 4. Keep Repositories Thin

```python
# ✅ GOOD: Simple data access
class UserRepository:
    async def get_active_users(self) -> list[User]:
        result = await self.db.execute(
            select(User).where(User.is_active == True)
        )
        return list(result.scalars().all())

# ❌ BAD: Complex logic in repository
class UserRepository:
    async def get_premium_users_with_recent_activity(self):
        # Too much logic for repository!
        users = await self.get_all()
        premium = [u for u in users if u.subscription == "premium"]
        recent = [u for u in premium if u.last_login > threshold]
        # ... more filtering
        return recent
```

---

## Key Takeaways

1. **Services** = Business logic, orchestration, validation
2. **Repositories** = Data access, queries, database operations
3. **Use dependency injection** for testability
4. **Services use repositories**, never the reverse
5. **Transaction management** in services or database dependency
6. **Type everything** for better IDE support
7. **Handle errors appropriately** - rollback on failure
