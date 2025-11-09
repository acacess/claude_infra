# Async and Error Handling - FastAPI Patterns

Complete guide to async patterns and error handling in FastAPI.

## Async Patterns

### Async/Await Basics

```python
# ✅ GOOD: Async for I/O operations
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    return user

# ❌ BAD: Sync with async database
@router.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    # Blocks the event loop!
    user = db.query(User).filter(User.id == user_id).first()
    return user
```

### HTTP Requests

```python
import httpx

async def fetch_external_data(url: str) -> dict:
    """Async HTTP request."""
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

### Concurrent Operations

```python
import asyncio

async def process_users(user_ids: list[int], db: AsyncSession):
    """Process multiple users concurrently."""
    tasks = [fetch_user_data(user_id, db) for user_id in user_ids]
    results = await asyncio.gather(*tasks)
    return results
```

## Error Handling

### HTTPException

```python
from fastapi import HTTPException, status

@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await user_repo.get_by_id(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )
    return user
```

### Custom Exceptions

```python
class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class InsufficientFundsError(Exception):
    def __init__(self, required: float, available: float):
        self.required = required
        self.available = available
        super().__init__(f"Insufficient funds: need {required}, have {available}")

# Exception handlers
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"detail": f"User {exc.user_id} not found"}
    )

@app.exception_handler(InsufficientFundsError)
async def insufficient_funds_handler(request: Request, exc: InsufficientFundsError):
    return JSONResponse(
        status_code=400,
        content={
            "detail": "Insufficient funds",
            "required": exc.required,
            "available": exc.available
        }
    )
```

### Service Layer Error Handling

```python
class UserService:
    async def create_user(self, data: UserCreate) -> UserResponse:
        try:
            user = await self.repo.create(data)
            await self.db.commit()
            return UserResponse.model_validate(user)
        except HTTPException:
            await self.db.rollback()
            raise  # Re-raise HTTP exceptions as-is
        except IntegrityError as e:
            await self.db.rollback()
            if "duplicate key" in str(e):
                raise HTTPException(
                    status_code=status.HTTP_409_CONFLICT,
                    detail="User already exists"
                )
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Database error"
            )
        except Exception as e:
            await self.db.rollback()
            sentry_sdk.capture_exception(e)
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Internal server error"
            )
```

## Key Takeaways

1. **Use async/await** for all I/O operations
2. **HTTPException** for HTTP errors
3. **Custom exceptions** for domain errors
4. **Exception handlers** for consistent responses
5. **Rollback transactions** on errors
6. **Capture to Sentry** for unexpected errors
