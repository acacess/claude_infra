# Data Fetching - Reflex with FastAPI Backend

Complete guide to fetching data from FastAPI backend in Reflex applications.

## HTTP Client Setup

### Using httpx for API Calls

```python
import httpx
import reflex as rx

class ApiConfig:
    """API configuration."""
    BASE_URL = "http://localhost:8000"
    TIMEOUT = 30.0

async def fetch_from_api(
    endpoint: str,
    method: str = "GET",
    data: dict | None = None,
    token: str | None = None
) -> dict:
    """
    Fetch data from FastAPI backend.

    Args:
        endpoint: API endpoint (e.g., "/users/1")
        method: HTTP method (GET, POST, PUT, DELETE)
        data: Request body data
        token: JWT token for authentication

    Returns:
        Response data as dict
    """
    url = f"{ApiConfig.BASE_URL}{endpoint}"
    headers = {}

    if token:
        headers["Authorization"] = f"Bearer {token}"

    async with httpx.AsyncClient(timeout=ApiConfig.TIMEOUT) as client:
        if method == "GET":
            response = await client.get(url, headers=headers)
        elif method == "POST":
            response = await client.post(url, json=data, headers=headers)
        elif method == "PUT":
            response = await client.put(url, json=data, headers=headers)
        elif method == "DELETE":
            response = await client.delete(url, headers=headers)

        response.raise_for_status()
        return response.json()
```

## Data Fetching Patterns

### Basic GET Request

```python
class UserState(rx.State):
    """State for user data."""
    users: list[dict] = []
    loading: bool = False
    error: str = ""

    async def load_users(self):
        """Load users from FastAPI backend."""
        self.loading = True
        self.error = ""

        try:
            data = await fetch_from_api("/users/")
            self.users = data["items"]
        except httpx.HTTPStatusError as e:
            self.error = f"HTTP {e.response.status_code}: {e.response.text}"
        except httpx.RequestError as e:
            self.error = f"Request failed: {str(e)}"
        except Exception as e:
            self.error = f"Unexpected error: {str(e)}"
        finally:
            self.loading = False
```

### POST Request (Create)

```python
class PostState(rx.State):
    """State for post management."""
    posts: list[dict] = []

    # Form state
    form_title: str = ""
    form_content: str = ""

    async def create_post(self, token: str):
        """Create new post via FastAPI."""
        try:
            data = {
                "title": self.form_title,
                "content": self.form_content,
                "is_published": True
            }

            response = await fetch_from_api(
                "/posts/",
                method="POST",
                data=data,
                token=token
            )

            # Add to local state
            self.posts.insert(0, response)

            # Clear form
            self.form_title = ""
            self.form_content = ""

        except Exception as e:
            self.error = str(e)
```

### With Authentication

```python
class AuthState(rx.State):
    """Authentication state."""
    access_token: str = ""
    user: dict | None = None

    async def login(self, email: str, password: str):
        """Login via FastAPI."""
        try:
            data = {"email": email, "password": password}
            response = await fetch_from_api("/auth/login", method="POST", data=data)

            self.access_token = response["access_token"]
            self.user = response["user"]

        except Exception as e:
            self.error = "Invalid credentials"

class DataState(rx.State):
    """State that needs authentication."""

    async def load_protected_data(self):
        """Load data with authentication."""
        auth_state = await self.get_state(AuthState)

        if not auth_state.access_token:
            self.error = "Not authenticated"
            return

        try:
            data = await fetch_from_api(
                "/protected/data",
                token=auth_state.access_token
            )
            self.data = data
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 401:
                # Token expired
                auth_state.access_token = ""
                self.error = "Session expired. Please log in again."
            else:
                self.error = str(e)
```

## Pagination

### Load More Pattern

```python
class PostState(rx.State):
    posts: list[dict] = []
    page: int = 1
    has_more: bool = True
    loading: bool = False

    async def load_posts(self):
        """Load paginated posts."""
        if not self.has_more or self.loading:
            return

        self.loading = True

        try:
            data = await fetch_from_api(f"/posts/?skip={(self.page - 1) * 20}&limit=20")

            self.posts.extend(data["items"])
            self.has_more = data["total"] > len(self.posts)
            self.page += 1

        except Exception as e:
            self.error = str(e)
        finally:
            self.loading = False
```

## Error Handling

### Comprehensive Error Handling

```python
async def safe_api_call(
    endpoint: str,
    method: str = "GET",
    data: dict | None = None,
    token: str | None = None
) -> tuple[dict | None, str | None]:
    """
    Make API call with comprehensive error handling.

    Returns:
        (data, error) tuple - one will be None
    """
    try:
        result = await fetch_from_api(endpoint, method, data, token)
        return (result, None)

    except httpx.HTTPStatusError as e:
        if e.response.status_code == 400:
            return (None, "Invalid request data")
        elif e.response.status_code == 401:
            return (None, "Authentication required")
        elif e.response.status_code == 403:
            return (None, "Access denied")
        elif e.response.status_code == 404:
            return (None, "Resource not found")
        elif e.response.status_code == 422:
            # Validation error from FastAPI
            detail = e.response.json().get("detail", "Validation error")
            return (None, str(detail))
        else:
            return (None, f"Server error: {e.response.status_code}")

    except httpx.TimeoutException:
        return (None, "Request timed out")

    except httpx.NetworkError:
        return (None, "Network error - check your connection")

    except Exception as e:
        return (None, f"Unexpected error: {str(e)}")

# Usage
class UserState(rx.State):
    async def load_user(self, user_id: int):
        data, error = await safe_api_call(f"/users/{user_id}")

        if error:
            self.error = error
            return

        self.current_user = data
```

## Best Practices

### 1. Always Handle Errors

```python
# ✅ GOOD: Comprehensive error handling
async def load_data(self):
    try:
        self.data = await fetch_from_api("/data")
    except Exception as e:
        self.error = str(e)

# ❌ BAD: No error handling
async def load_data(self):
    self.data = await fetch_from_api("/data")
```

### 2. Show Loading States

```python
# ✅ GOOD: Loading indicator
async def load_data(self):
    self.loading = True
    try:
        self.data = await fetch_from_api("/data")
    finally:
        self.loading = False
```

### 3. Use Substates for Auth

```python
# ✅ GOOD: Separate auth state
class AuthState(rx.State):
    token: str = ""

class DataState(rx.State):
    async def load(self):
        auth = await self.get_state(AuthState)
        data = await fetch_from_api("/data", token=auth.token)
```

### 4. Clear Error Messages

```python
# ✅ GOOD: User-friendly errors
except httpx.HTTPStatusError as e:
    if e.response.status_code == 404:
        self.error = "Post not found"
    elif e.response.status_code == 403:
        self.error = "You don't have permission to view this post"

# ❌ BAD: Technical errors exposed to user
except Exception as e:
    self.error = str(e)  # May expose internal details
```

## Key Takeaways

1. **Use httpx** for async HTTP requests
2. **Always handle errors** - network, HTTP, validation
3. **Show loading states** - better UX
4. **Store auth tokens** in AuthState substate
5. **Type hints** on all async methods
6. **Pagination** for large datasets
7. **Clear error messages** for users
