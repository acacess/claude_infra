# Architecture Overview - FastAPI Backend

Complete guide to layered architecture for FastAPI microservices.

## Table of Contents

- [Layered Architecture Philosophy](#layered-architecture-philosophy)
- [Request Lifecycle](#request-lifecycle)
- [Layer Responsibilities](#layer-responsibilities)
- [Dependency Injection](#dependency-injection)
- [Error Flow](#error-flow)

---

## Layered Architecture Philosophy

### The Four Layers

```
┌─────────────────────────────────────────┐
│         FastAPI Router Layer            │
│  (Routing, basic validation, HTTP)      │
└─────────────────┬───────────────────────┘
                  │ Depends()
┌─────────────────▼───────────────────────┐
│        Dependency Layer                  │
│  (Auth, DB session, service injection)   │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│          Service Layer                   │
│      (Business logic, orchestration)     │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│        Repository Layer                  │
│     (Database access, queries)           │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         Database (Supabase/PostgreSQL)   │
└─────────────────────────────────────────┘
```

### Core Principle: Single Responsibility

Each layer has EXACTLY ONE job:

1. **Router**: Define HTTP endpoints, delegate to services
2. **Dependencies**: Provide dependencies (DB, auth, services)
3. **Service**: Implement business logic
4. **Repository**: Execute database queries

---

## Request Lifecycle

### Example: Create User Request

```python
# 1. ROUTER LAYER - app/routers/user_router.py
@router.post("/users/", status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,  # Pydantic validation
    current_user: User = Depends(get_current_active_user),
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    """Create a new user (admin only)."""
    return await service.create_user(user_data, current_user)


# 2. SERVICE LAYER - app/services/user_service.py
class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.repo = UserRepository(db)

    async def create_user(
        self,
        user_data: UserCreate,
        created_by: User
    ) -> User:
        """Business logic for user creation."""
        try:
            # Check if email exists
            existing = await self.repo.get_by_email(user_data.email)
            if existing:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Email already registered"
                )

            # Hash password
            hashed_password = get_password_hash(user_data.password)

            # Create user via repository
            user = await self.repo.create(
                email=user_data.email,
                username=user_data.username,
                hashed_password=hashed_password,
                created_by_id=created_by.id
            )

            # Send welcome email (example of orchestration)
            await send_welcome_email(user.email)

            await self.db.commit()
            await self.db.refresh(user)
            return user

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception:
            await self.db.rollback()
            raise


# 3. REPOSITORY LAYER - app/repositories/user_repository.py
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_email(self, email: str) -> User | None:
        """Get user by email."""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(
        self,
        email: str,
        username: str,
        hashed_password: str,
        created_by_id: int
    ) -> User:
        """Create new user in database."""
        user = User(
            email=email,
            username=username,
            hashed_password=hashed_password,
            created_by_id=created_by_id
        )
        self.db.add(user)
        await self.db.flush()
        return user
```

**Flow:**
1. Router validates Pydantic schema, injects dependencies
2. Service implements business logic (check duplicates, hash password)
3. Repository executes database operations
4. Response flows back up through layers

---

## Layer Responsibilities

### Router Layer

**ONLY responsible for:**
- ✅ Defining HTTP endpoints (method, path)
- ✅ Declaring dependencies (`Depends()`)
- ✅ Specifying request/response schemas
- ✅ Setting status codes
- ✅ Delegating to services

**NEVER responsible for:**
- ❌ Business logic
- ❌ Database queries
- ❌ Complex validation
- ❌ Error handling (beyond HTTPException)

**Example:**
```python
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    """Get user by ID."""
    return await service.get_user(user_id)
```

### Dependency Layer

**Responsibilities:**
- ✅ Database session management
- ✅ Authentication/authorization
- ✅ Service instantiation
- ✅ Request context setup

**Example:**
```python
# app/dependencies/database.py
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Provide database session."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()

# app/dependencies/auth.py
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Get current authenticated user."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials"
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await UserRepository(db).get_by_id(user_id)
    if user is None:
        raise credentials_exception
    return user

# app/dependencies/services.py
def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    """Provide UserService instance."""
    return UserService(db)
```

### Service Layer

**Responsibilities:**
- ✅ Business logic implementation
- ✅ Orchestrating multiple repositories
- ✅ Transaction management
- ✅ Business rule validation
- ✅ Calling external services

**NEVER:**
- ❌ HTTP-specific logic (status codes, headers)
- ❌ Direct database queries (use repositories)

**Example:**
```python
class PostService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.post_repo = PostRepository(db)
        self.user_repo = UserRepository(db)

    async def create_post(
        self,
        post_data: PostCreate,
        author: User
    ) -> Post:
        """Create a new post with business logic."""
        # Business rule: user must be verified
        if not author.is_verified:
            raise HTTPException(
                status_code=403,
                detail="Only verified users can create posts"
            )

        # Business rule: check daily post limit
        today_count = await self.post_repo.count_user_posts_today(author.id)
        if today_count >= 10:
            raise HTTPException(
                status_code=429,
                detail="Daily post limit reached"
            )

        # Create post
        post = await self.post_repo.create(
            title=post_data.title,
            content=post_data.content,
            author_id=author.id
        )

        # Update user stats (orchestration)
        await self.user_repo.increment_post_count(author.id)

        return post
```

### Repository Layer

**Responsibilities:**
- ✅ Database queries (SELECT, INSERT, UPDATE, DELETE)
- ✅ SQLAlchemy model manipulation
- ✅ Query optimization
- ✅ Data access patterns

**NEVER:**
- ❌ Business logic
- ❌ Validation beyond data integrity
- ❌ HTTPException (raise native Python exceptions)

**Example:**
```python
class PostRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, post_id: int) -> Post | None:
        """Get post by ID."""
        result = await self.db.execute(
            select(Post).where(Post.id == post_id)
        )
        return result.scalar_one_or_none()

    async def get_user_posts(
        self,
        user_id: int,
        limit: int = 100,
        offset: int = 0
    ) -> list[Post]:
        """Get posts by user with pagination."""
        result = await self.db.execute(
            select(Post)
            .where(Post.author_id == user_id)
            .order_by(Post.created_at.desc())
            .limit(limit)
            .offset(offset)
        )
        return list(result.scalars().all())

    async def create(
        self,
        title: str,
        content: str,
        author_id: int
    ) -> Post:
        """Create new post."""
        post = Post(
            title=title,
            content=content,
            author_id=author_id
        )
        self.db.add(post)
        await self.db.commit()
        await self.db.refresh(post)
        return post

    async def count_user_posts_today(self, user_id: int) -> int:
        """Count posts created by user today."""
        today = datetime.now().replace(hour=0, minute=0, second=0)
        result = await self.db.execute(
            select(func.count(Post.id))
            .where(Post.author_id == user_id)
            .where(Post.created_at >= today)
        )
        return result.scalar_one()
```

---

## Dependency Injection

### Why Dependency Injection?

**Benefits:**
- Testability (easy to mock dependencies)
- Loose coupling between layers
- Clear dependency graph
- Automatic cleanup (context managers)

### Common Dependency Patterns

```python
# 1. Database Session
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

# 2. Current User (requires DB)
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    # ... validate token and get user
    return user

# 3. Admin User (requires current user)
async def get_current_admin_user(
    current_user: User = Depends(get_current_user)
) -> User:
    if not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Admin access required")
    return current_user

# 4. Service (requires DB)
def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    return UserService(db)

# Usage in router
@router.get("/admin/users/")
async def list_users(
    admin: User = Depends(get_current_admin_user),
    service: UserService = Depends(get_user_service)
) -> list[UserResponse]:
    return await service.list_all_users()
```

---

## Error Flow

### Error Handling Across Layers

```python
# Repository - Raise domain exceptions
class UserRepository:
    async def get_by_id(self, user_id: int) -> User:
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one_or_none()
        if not user:
            raise ValueError(f"User {user_id} not found")
        return user

# Service - Convert to HTTP exceptions
class UserService:
    async def get_user(self, user_id: int) -> User:
        try:
            return await self.repo.get_by_id(user_id)
        except ValueError as e:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=str(e)
            )
        except Exception as e:
            # Log to Sentry
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Internal server error"
            )

# Router - Let FastAPI handle HTTPException
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    # FastAPI automatically converts HTTPException to response
    return await service.get_user(user_id)
```

---

## Key Takeaways

1. **Each layer has ONE job** - Don't mix responsibilities
2. **Use dependency injection** - Makes testing easier
3. **Repository never has business logic** - Only data access
4. **Service orchestrates** - Business rules and multiple repositories
5. **Router is thin** - Just routing and dependency declaration
6. **Errors bubble up** - Repository → Service (convert to HTTP) → Router
