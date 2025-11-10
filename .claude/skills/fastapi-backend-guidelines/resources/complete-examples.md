# Complete Examples - Full FastAPI Implementation

End-to-end working examples demonstrating all patterns together.

## Table of Contents

- [Complete Feature: Blog Posts](#complete-feature-blog-posts)
- [Full CRUD API](#full-crud-api)
- [Authentication Flow](#authentication-flow)
- [Testing Example](#testing-example)

---

## Complete Feature: Blog Posts

### 1. SQLAlchemy Model

```python
# app/models/post.py
from sqlalchemy import Column, Integer, String, Text, Boolean, ForeignKey, DateTime
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.core.database import Base

class Post(Base):
    """Blog post model."""
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False, index=True)
    content = Column(Text, nullable=False)
    is_published = Column(Boolean, default=False, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
```

### 2. Pydantic Schemas

```python
# app/schemas/post.py
from pydantic import BaseModel, Field
from datetime import datetime

class PostBase(BaseModel):
    """Base post schema."""
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1)
    is_published: bool = False

class PostCreate(PostBase):
    """Schema for creating a post."""
    pass

class PostUpdate(BaseModel):
    """Schema for updating a post (all fields optional)."""
    title: str | None = Field(None, min_length=1, max_length=200)
    content: str | None = Field(None, min_length=1)
    is_published: bool | None = None

class PostResponse(PostBase):
    """Schema for post response."""
    id: int
    author_id: int
    created_at: datetime
    updated_at: datetime | None = None

    class Config:
        from_attributes = True

class PostListResponse(BaseModel):
    """Paginated post list response."""
    total: int
    items: list[PostResponse]
    skip: int
    limit: int
```

### 3. Repository Layer

```python
# app/repositories/post_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, and_
from datetime import datetime

from app.models.post import Post

class PostRepository:
    """Repository for post data access."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, post_id: int) -> Post | None:
        """Get post by ID."""
        result = await self.db.execute(
            select(Post).where(Post.id == post_id)
        )
        return result.scalar_one_or_none()

    async def get_published_posts(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> tuple[list[Post], int]:
        """Get published posts with pagination and total count."""
        # Get total count
        count_result = await self.db.execute(
            select(func.count(Post.id)).where(Post.is_published == True)
        )
        total = count_result.scalar_one()

        # Get posts
        result = await self.db.execute(
            select(Post)
            .where(Post.is_published == True)
            .order_by(Post.created_at.desc())
            .offset(skip)
            .limit(limit)
        )
        posts = list(result.scalars().all())

        return posts, total

    async def get_user_posts(
        self,
        user_id: int,
        include_unpublished: bool = False
    ) -> list[Post]:
        """Get all posts by a user."""
        query = select(Post).where(Post.author_id == user_id)

        if not include_unpublished:
            query = query.where(Post.is_published == True)

        result = await self.db.execute(
            query.order_by(Post.created_at.desc())
        )
        return list(result.scalars().all())

    async def count_user_posts_today(self, user_id: int) -> int:
        """Count posts created by user today."""
        today = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)
        result = await self.db.execute(
            select(func.count(Post.id)).where(
                and_(
                    Post.author_id == user_id,
                    Post.created_at >= today
                )
            )
        )
        return result.scalar_one()

    async def create(
        self,
        title: str,
        content: str,
        author_id: int,
        is_published: bool = False
    ) -> Post:
        """Create a new post."""
        post = Post(
            title=title,
            content=content,
            author_id=author_id,
            is_published=is_published
        )
        self.db.add(post)
        await self.db.flush()
        await self.db.refresh(post)
        return post

    async def update(
        self,
        post: Post,
        title: str | None = None,
        content: str | None = None,
        is_published: bool | None = None
    ) -> Post:
        """Update a post."""
        if title is not None:
            post.title = title
        if content is not None:
            post.content = content
        if is_published is not None:
            post.is_published = is_published

        await self.db.flush()
        await self.db.refresh(post)
        return post

    async def delete(self, post: Post) -> None:
        """Delete a post."""
        await self.db.delete(post)
        await self.db.flush()
```

### 4. Service Layer

```python
# app/services/post_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException, status
import sentry_sdk

from app.repositories.post_repository import PostRepository
from app.schemas.post import PostCreate, PostUpdate, PostResponse, PostListResponse
from app.models.user import User
from app.models.post import Post

class PostService:
    """Service for post business logic."""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.repo = PostRepository(db)

    async def create_post(
        self,
        post_data: PostCreate,
        author: User
    ) -> PostResponse:
        """
        Create a new post.

        Business rules:
        - Author must be active
        - Max 10 posts per day per user
        """
        try:
            # Business rule: user must be active
            if not author.is_active:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Only active users can create posts"
                )

            # Business rule: check daily post limit
            today_count = await self.repo.count_user_posts_today(author.id)
            if today_count >= 10:
                raise HTTPException(
                    status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                    detail="Daily post limit (10) reached"
                )

            # Create post
            post = await self.repo.create(
                title=post_data.title,
                content=post_data.content,
                author_id=author.id,
                is_published=post_data.is_published
            )

            await self.db.commit()
            return PostResponse.model_validate(post)

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to create post"
            )

    async def get_post(self, post_id: int) -> PostResponse:
        """Get post by ID."""
        post = await self.repo.get_by_id(post_id)
        if not post:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Post {post_id} not found"
            )
        return PostResponse.model_validate(post)

    async def list_published_posts(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> PostListResponse:
        """List all published posts with pagination."""
        posts, total = await self.repo.get_published_posts(skip, limit)
        return PostListResponse(
            total=total,
            items=[PostResponse.model_validate(p) for p in posts],
            skip=skip,
            limit=limit
        )

    async def update_post(
        self,
        post_id: int,
        post_data: PostUpdate,
        current_user: User
    ) -> PostResponse:
        """
        Update a post.

        Business rules:
        - Only author or admin can update
        """
        try:
            post = await self.repo.get_by_id(post_id)
            if not post:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=f"Post {post_id} not found"
                )

            # Business rule: only author or admin can update
            if post.author_id != current_user.id and not current_user.is_admin:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Not authorized to update this post"
                )

            # Update post
            updated_post = await self.repo.update(
                post,
                title=post_data.title,
                content=post_data.content,
                is_published=post_data.is_published
            )

            await self.db.commit()
            return PostResponse.model_validate(updated_post)

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to update post"
            )

    async def delete_post(self, post_id: int, current_user: User) -> None:
        """
        Delete a post.

        Business rules:
        - Only author or admin can delete
        """
        try:
            post = await self.repo.get_by_id(post_id)
            if not post:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=f"Post {post_id} not found"
                )

            # Business rule: only author or admin can delete
            if post.author_id != current_user.id and not current_user.is_admin:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Not authorized to delete this post"
                )

            await self.repo.delete(post)
            await self.db.commit()

        except HTTPException:
            await self.db.rollback()
            raise
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Failed to delete post"
            )
```

### 5. Router

```python
# app/routers/post_router.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.dependencies.auth import get_current_user
from app.schemas.post import PostCreate, PostUpdate, PostResponse, PostListResponse
from app.services.post_service import PostService
from app.models.user import User

router = APIRouter(
    prefix="/posts",
    tags=["posts"]
)

def get_post_service(db: AsyncSession = Depends(get_db)) -> PostService:
    """Dependency to get PostService instance."""
    return PostService(db)

@router.post(
    "/",
    status_code=status.HTTP_201_CREATED,
    response_model=PostResponse
)
async def create_post(
    post_data: PostCreate,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
) -> PostResponse:
    """
    Create a new post.

    Requires authentication. Max 10 posts per day per user.
    """
    return await service.create_post(post_data, current_user)

@router.get("/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: int,
    service: PostService = Depends(get_post_service)
) -> PostResponse:
    """Get a post by ID."""
    return await service.get_post(post_id)

@router.get("/", response_model=PostListResponse)
async def list_posts(
    skip: int = 0,
    limit: int = 100,
    service: PostService = Depends(get_post_service)
) -> PostListResponse:
    """
    List all published posts.

    Supports pagination with skip and limit parameters.
    """
    return await service.list_published_posts(skip, limit)

@router.put("/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    post_data: PostUpdate,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
) -> PostResponse:
    """
    Update a post.

    Only the author or an admin can update a post.
    """
    return await service.update_post(post_id, post_data, current_user)

@router.delete("/{post_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_post(
    post_id: int,
    current_user: User = Depends(get_current_user),
    service: PostService = Depends(get_post_service)
) -> None:
    """
    Delete a post.

    Only the author or an admin can delete a post.
    """
    await service.delete_post(post_id, current_user)
```

### 6. Tests

```python
# tests/test_post_api.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User
from app.models.post import Post

@pytest.mark.asyncio
async def test_create_post(
    client: AsyncClient,
    test_user: User,
    auth_headers: dict
):
    """Test creating a post."""
    response = await client.post(
        "/posts/",
        json={
            "title": "Test Post",
            "content": "This is a test post",
            "is_published": True
        },
        headers=auth_headers
    )

    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Test Post"
    assert data["content"] == "This is a test post"
    assert data["is_published"] is True
    assert data["author_id"] == test_user.id

@pytest.mark.asyncio
async def test_get_post(
    client: AsyncClient,
    test_post: Post
):
    """Test getting a post by ID."""
    response = await client.get(f"/posts/{test_post.id}")

    assert response.status_code == 200
    data = response.json()
    assert data["id"] == test_post.id
    assert data["title"] == test_post.title

@pytest.mark.asyncio
async def test_list_posts(
    client: AsyncClient,
    db: AsyncSession,
    test_user: User
):
    """Test listing posts with pagination."""
    # Create multiple posts
    for i in range(5):
        post = Post(
            title=f"Post {i}",
            content=f"Content {i}",
            author_id=test_user.id,
            is_published=True
        )
        db.add(post)
    await db.commit()

    response = await client.get("/posts/?skip=0&limit=3")

    assert response.status_code == 200
    data = response.json()
    assert data["total"] == 5
    assert len(data["items"]) == 3
    assert data["skip"] == 0
    assert data["limit"] == 3

@pytest.mark.asyncio
async def test_update_post_as_author(
    client: AsyncClient,
    test_post: Post,
    auth_headers: dict
):
    """Test updating a post as the author."""
    response = await client.put(
        f"/posts/{test_post.id}",
        json={"title": "Updated Title"},
        headers=auth_headers
    )

    assert response.status_code == 200
    data = response.json()
    assert data["title"] == "Updated Title"

@pytest.mark.asyncio
async def test_delete_post_as_author(
    client: AsyncClient,
    test_post: Post,
    auth_headers: dict
):
    """Test deleting a post as the author."""
    response = await client.delete(
        f"/posts/{test_post.id}",
        headers=auth_headers
    )

    assert response.status_code == 204

@pytest.mark.asyncio
async def test_daily_post_limit(
    client: AsyncClient,
    test_user: User,
    auth_headers: dict,
    db: AsyncSession
):
    """Test that users can't create more than 10 posts per day."""
    # Create 10 posts
    for i in range(10):
        post = Post(
            title=f"Post {i}",
            content=f"Content {i}",
            author_id=test_user.id
        )
        db.add(post)
    await db.commit()

    # 11th post should fail
    response = await client.post(
        "/posts/",
        json={
            "title": "11th Post",
            "content": "Should fail"
        },
        headers=auth_headers
    )

    assert response.status_code == 429
    assert "Daily post limit" in response.json()["detail"]
```

---

## Authentication Flow

### 1. Auth Schemas

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr

class LoginRequest(BaseModel):
    """Login request schema."""
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    """Token response schema."""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class LoginResponse(TokenResponse):
    """Login response schema."""
    user: dict  # UserResponse as dict
```

### 2. Auth Service

```python
# app/services/auth_service.py
from datetime import datetime, timedelta
from jose import jwt
from passlib.context import CryptContext
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException, status
import sentry_sdk

from app.repositories.user_repository import UserRepository
from app.schemas.auth import LoginRequest, LoginResponse
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    """Service for authentication logic."""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.user_repo = UserRepository(db)

    def verify_password(self, plain_password: str, hashed_password: str) -> bool:
        """Verify password against hash."""
        return pwd_context.verify(plain_password, hashed_password)

    def create_access_token(self, user_id: int) -> str:
        """Create JWT access token."""
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
        to_encode = {"sub": str(user_id), "exp": expire}
        return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

    def create_refresh_token(self, user_id: int) -> str:
        """Create JWT refresh token."""
        expire = datetime.utcnow() + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
        to_encode = {"sub": str(user_id), "exp": expire, "type": "refresh"}
        return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

    async def login(self, login_data: LoginRequest) -> LoginResponse:
        """
        Authenticate user and return tokens.

        Business rules:
        - User must exist
        - Password must be correct
        - User must be active
        """
        try:
            # Get user by email
            user = await self.user_repo.get_by_email(login_data.email)
            if not user:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid email or password"
                )

            # Verify password
            if not self.verify_password(login_data.password, user.hashed_password):
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid email or password"
                )

            # Check if user is active
            if not user.is_active:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="User account is inactive"
                )

            # Create tokens
            access_token = self.create_access_token(user.id)
            refresh_token = self.create_refresh_token(user.id)

            # Return response with user data
            return LoginResponse(
                access_token=access_token,
                refresh_token=refresh_token,
                user={
                    "id": user.id,
                    "email": user.email,
                    "username": user.username,
                    "is_active": user.is_active,
                    "is_admin": user.is_admin
                }
            )

        except HTTPException:
            raise
        except Exception as e:
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Authentication failed"
            )
```

### 3. Auth Router

```python
# app/routers/auth_router.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.dependencies.services import get_auth_service
from app.schemas.auth import LoginRequest, LoginResponse
from app.services.auth_service import AuthService

router = APIRouter(
    prefix="/auth",
    tags=["auth"]
)

def get_auth_service(db: AsyncSession = Depends(get_db)) -> AuthService:
    """Dependency to get AuthService instance."""
    return AuthService(db)

@router.post(
    "/login",
    response_model=LoginResponse,
    status_code=status.HTTP_200_OK
)
async def login(
    login_data: LoginRequest,
    service: AuthService = Depends(get_auth_service)
) -> LoginResponse:
    """
    Authenticate user and return access/refresh tokens.

    Returns:
    - access_token: JWT token for API authentication
    - refresh_token: JWT token for refreshing access token
    - user: User information (id, email, username, etc.)
    """
    return await service.login(login_data)
```

**Note**: This matches the frontend expectation in `reflex-frontend-guidelines` which expects:
```python
data = response.json()
self.access_token = data["access_token"]
self.refresh_token = data["refresh_token"]
self.user = data["user"]
```

---

## Key Takeaways

This complete example demonstrates:

1. **Layered Architecture**: Router → Service → Repository → Database
2. **Dependency Injection**: All dependencies injected via FastAPI
3. **Pydantic Validation**: Request/response models with validation
4. **Business Logic in Service**: Rules like daily post limit
5. **Error Handling**: HTTPException with proper status codes
6. **Sentry Integration**: Error tracking on exceptions
7. **Testing**: Comprehensive async tests with pytest
8. **Type Safety**: Full type hints throughout
9. **Transaction Management**: Commit on success, rollback on error
10. **Authorization**: User ownership checks in service layer
