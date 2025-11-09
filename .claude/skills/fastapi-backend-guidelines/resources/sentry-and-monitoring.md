# Sentry and Monitoring - Error Tracking

Complete guide to Sentry integration in FastAPI for error tracking and performance monitoring.

## Sentry Setup

### Installation

```bash
pip install sentry-sdk[fastapi]
```

### Basic Configuration

```python
# app/main.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from app.core.config import settings

# Initialize Sentry
sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    environment=settings.SENTRY_ENVIRONMENT,
    integrations=[FastApiIntegration()],
    traces_sample_rate=0.1,  # 10% of transactions
    profiles_sample_rate=0.1,  # 10% of profiles
)

app = FastAPI()
```

## Error Capture

### Automatic Error Capture

```python
# Errors are automatically captured by FastAPI integration
@router.get("/users/{user_id}")
async def get_user(user_id: int):
    # Any unhandled exception will be sent to Sentry
    user = await db.execute(select(User).where(User.id == user_id))
    return user
```

### Manual Error Capture

```python
import sentry_sdk

@router.post("/process")
async def process_data(data: dict):
    try:
        result = await complex_operation(data)
        return result
    except ValueError as e:
        # Capture expected errors with context
        sentry_sdk.capture_exception(e)
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        # Capture unexpected errors
        sentry_sdk.capture_exception(e)
        raise HTTPException(status_code=500, detail="Internal error")
```

### Adding Context

```python
# Set user context
sentry_sdk.set_user({"id": user.id, "email": user.email})

# Add tags
sentry_sdk.set_tag("endpoint", "create_order")
sentry_sdk.set_tag("customer_tier", "premium")

# Add breadcrumbs
sentry_sdk.add_breadcrumb(
    category="auth",
    message="User logged in",
    level="info"
)
```

## Performance Monitoring

### Transaction Tracking

```python
@router.post("/orders/")
async def create_order(order_data: OrderCreate):
    with sentry_sdk.start_transaction(op="http.server", name="POST /orders/"):
        # Operation is tracked
        order = await order_service.create(order_data)
        return order
```

### Custom Spans

```python
async def process_payment(amount: float):
    with sentry_sdk.start_span(op="payment", description="Process payment"):
        # This operation's performance is tracked
        result = await payment_gateway.charge(amount)
        return result
```

## Key Takeaways

1. **Initialize Sentry** at app startup
2. **Automatic capture** for unhandled exceptions
3. **Manual capture** with context for expected errors
4. **User context** for better debugging
5. **Performance monitoring** with transactions
