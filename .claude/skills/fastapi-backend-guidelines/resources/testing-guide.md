# Testing Guide - FastAPI Testing Patterns

Complete guide to testing FastAPI applications with pytest.

## Test Setup

### Pytest Configuration

```python
# conftest.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from httpx import AsyncClient

from app.main import app
from app.core.database import Base, get_db
from app.core.config import settings

# Test database URL
# For local testing, use localhost. For Supabase, use your Supabase test project connection string:
# TEST_DATABASE_URL = "postgresql+asyncpg://postgres:[PASSWORD]@[PROJECT_REF].supabase.co:5432/postgres"
TEST_DATABASE_URL = "postgresql+asyncpg://test:test@localhost/test_db"

@pytest.fixture
async def test_db():
    """Create test database."""
    engine = create_async_engine(TEST_DATABASE_URL)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    TestSessionLocal = sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )

    async def override_get_db():
        async with TestSessionLocal() as session:
            yield session

    app.dependency_overrides[get_db] = override_get_db

    yield

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

@pytest.fixture
async def client(test_db):
    """Test client."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client
```

## Unit Tests

### Testing Services

```python
# tests/test_user_service.py
import pytest
from app.services.user_service import UserService
from app.schemas.user import UserCreate

@pytest.mark.asyncio
async def test_create_user(client, test_db):
    """Test user creation."""
    user_data = UserCreate(
        email="test@example.com",
        username="testuser",
        password="Test123!"
    )

    response = await client.post("/users/", json=user_data.model_dump())

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert data["username"] == "testuser"
    assert "id" in data
    assert "hashed_password" not in data

@pytest.mark.asyncio
async def test_create_duplicate_user(client, test_db):
    """Test creating duplicate user fails."""
    user_data = {
        "email": "test@example.com",
        "username": "testuser",
        "password": "Test123!"
    }

    # Create first user
    await client.post("/users/", json=user_data)

    # Try to create duplicate
    response = await client.post("/users/", json=user_data)

    assert response.status_code == 400
    assert "already registered" in response.json()["detail"]
```

### Testing Repositories

```python
@pytest.mark.asyncio
async def test_user_repository_get_by_email(db_session):
    """Test repository get_by_email method."""
    repo = UserRepository(db_session)

    # Create user
    user = await repo.create(
        email="test@example.com",
        username="testuser",
        hashed_password="hashed123"
    )
    await db_session.commit()

    # Fetch by email
    found_user = await repo.get_by_email("test@example.com")

    assert found_user is not None
    assert found_user.email == "test@example.com"
    assert found_user.username == "testuser"
```

## Integration Tests

### API Endpoint Tests

```python
@pytest.mark.asyncio
async def test_user_crud_flow(client):
    """Test complete CRUD flow for users."""

    # Create
    create_data = {
        "email": "crud@example.com",
        "username": "cruduser",
        "password": "Crud123!"
    }
    response = await client.post("/users/", json=create_data)
    assert response.status_code == 201
    user_id = response.json()["id"]

    # Read
    response = await client.get(f"/users/{user_id}")
    assert response.status_code == 200
    assert response.json()["email"] == "crud@example.com"

    # Update
    update_data = {"username": "updated_user"}
    response = await client.put(f"/users/{user_id}", json=update_data)
    assert response.status_code == 200
    assert response.json()["username"] == "updated_user"

    # Delete
    response = await client.delete(f"/users/{user_id}")
    assert response.status_code == 204

    # Verify deleted
    response = await client.get(f"/users/{user_id}")
    assert response.status_code == 404
```

### Testing with Authentication

```python
@pytest.fixture
async def auth_headers(client):
    """Get authentication headers."""
    # Login
    response = await client.post(
        "/auth/login",
        json={"email": "test@example.com", "password": "Test123!"}
    )
    token = response.json()["access_token"]

    return {"Authorization": f"Bearer {token}"}

@pytest.mark.asyncio
async def test_protected_endpoint(client, auth_headers):
    """Test accessing protected endpoint."""
    response = await client.get("/protected", headers=auth_headers)
    assert response.status_code == 200
```

## Mocking

### Mocking External Services

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_with_mocked_external_api(client):
    """Test with mocked external API."""
    mock_response = {"data": "mocked"}

    with patch("app.services.external_service.fetch_data") as mock_fetch:
        mock_fetch.return_value = mock_response

        response = await client.get("/external-data")

        assert response.status_code == 200
        assert response.json() == mock_response
        mock_fetch.assert_called_once()
```

## Test Coverage

```bash
# Run tests with coverage
pytest --cov=app --cov-report=html

# View coverage report
open htmlcov/index.html
```

## Key Takeaways

1. **pytest** with **pytest-asyncio** for async tests
2. **Separate test database** to avoid polluting production
3. **Fixtures** for reusable test setup
4. **@pytest.mark.asyncio** for async test functions
5. **Test both success and failure** cases
6. **Mock external dependencies** for unit tests
7. **Integration tests** for full API flows
