# State Management - Reflex Best Practices

Complete guide to managing state in Reflex applications.

## Table of Contents

- [State Class Basics](#state-class-basics)
- [Event Handlers](#event-handlers)
- [Computed Vars](#computed-vars)
- [Substates](#substates)
- [State Lifecycle](#state-lifecycle)
- [Best Practices](#best-practices)

---

## State Class Basics

### What is State?

In Reflex, **State** is a Python class that:
- Lives on the backend (Python server)
- Syncs to frontend via WebSocket
- Updates trigger UI re-renders
- Persists per user session

### Basic State Class

```python
import reflex as rx

class CounterState(rx.State):
    """Simple counter state."""
    count: int = 0

    def increment(self):
        """Increment counter."""
        self.count += 1

    def decrement(self):
        """Decrement counter."""
        self.count -= 1

    def reset(self):
        """Reset counter to zero."""
        self.count = 0
```

### Using State in Components

```python
def counter() -> rx.Component:
    """Counter component using CounterState."""
    return rx.vstack(
        rx.heading(f"Count: {CounterState.count}"),
        rx.hstack(
            rx.button("-", on_click=CounterState.decrement),
            rx.button("Reset", on_click=CounterState.reset),
            rx.button("+", on_click=CounterState.increment),
        ),
    )
```

---

## Event Handlers

### Synchronous Event Handlers

```python
class TodoState(rx.State):
    """Todo list state."""
    todos: list[str] = []
    new_todo: str = ""

    def set_new_todo(self, value: str):
        """Update new todo input (sync)."""
        self.new_todo = value

    def add_todo(self):
        """Add todo to list (sync)."""
        if self.new_todo.strip():
            self.todos.append(self.new_todo)
            self.new_todo = ""

    def remove_todo(self, index: int):
        """Remove todo by index (sync)."""
        self.todos.pop(index)
```

### Asynchronous Event Handlers

```python
import httpx

class PostState(rx.State):
    """Post management state."""
    posts: list[dict] = []
    loading: bool = False
    error: str = ""

    async def load_posts(self):
        """Load posts from API (async)."""
        self.loading = True
        self.error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get("http://localhost:8000/posts/")
                response.raise_for_status()
                self.posts = response.json()["items"]
        except httpx.HTTPError as e:
            self.error = f"Failed to load posts: {str(e)}"
        except Exception as e:
            self.error = f"Unexpected error: {str(e)}"
        finally:
            self.loading = False

    async def create_post(self, title: str, content: str):
        """Create new post (async)."""
        self.loading = True
        self.error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "http://localhost:8000/posts/",
                    json={"title": title, "content": content},
                    headers={"Authorization": f"Bearer {self.access_token}"}
                )
                response.raise_for_status()
                # Reload posts after creation
                await self.load_posts()
        except httpx.HTTPError as e:
            self.error = f"Failed to create post: {str(e)}"
        finally:
            self.loading = False
```

### Event Handler with Parameters

```python
class UserState(rx.State):
    """User management state."""
    users: list[dict] = []
    selected_user_id: int | None = None

    def select_user(self, user_id: int):
        """Select a user by ID."""
        self.selected_user_id = user_id

    async def delete_user(self, user_id: int):
        """Delete user by ID."""
        async with httpx.AsyncClient() as client:
            await client.delete(f"http://localhost:8000/users/{user_id}")
        # Remove from local state
        self.users = [u for u in self.users if u["id"] != user_id]


# Usage in component
def user_list() -> rx.Component:
    return rx.vstack(
        rx.foreach(
            UserState.users,
            lambda user: rx.hstack(
                rx.text(user["name"]),
                rx.button(
                    "Delete",
                    on_click=lambda: UserState.delete_user(user["id"])
                ),
            )
        )
    )
```

---

## Computed Vars

### What are Computed Vars?

Computed vars are **derived values** that:
- Automatically update when dependencies change
- Cached until dependencies change
- Defined with `@rx.var` decorator

### Basic Computed Var

```python
class ShoppingCartState(rx.State):
    """Shopping cart state."""
    items: list[dict] = []

    @rx.var
    def total_items(self) -> int:
        """Total number of items in cart."""
        return len(self.items)

    @rx.var
    def total_price(self) -> float:
        """Total price of all items."""
        return sum(item["price"] * item["quantity"] for item in self.items)

    @rx.var
    def is_empty(self) -> bool:
        """Check if cart is empty."""
        return len(self.items) == 0
```

### Cached Computed Vars

```python
class DataState(rx.State):
    """Data processing state."""
    raw_data: list[dict] = []
    filter_term: str = ""

    @rx.var
    def filtered_data(self) -> list[dict]:
        """
        Filtered data (cached).

        Only recomputes when raw_data or filter_term changes.
        """
        if not self.filter_term:
            return self.raw_data

        return [
            item for item in self.raw_data
            if self.filter_term.lower() in item["name"].lower()
        ]

    @rx.var
    def filtered_count(self) -> int:
        """Count of filtered items (depends on filtered_data)."""
        return len(self.filtered_data)
```

### Using Computed Vars

```python
def shopping_cart() -> rx.Component:
    """Shopping cart component using computed vars."""
    return rx.vstack(
        rx.heading(f"Cart ({ShoppingCartState.total_items} items)"),
        rx.text(f"Total: ${ShoppingCartState.total_price:.2f}"),
        rx.cond(
            ShoppingCartState.is_empty,
            rx.text("Your cart is empty"),
            rx.vstack(
                rx.foreach(
                    ShoppingCartState.items,
                    lambda item: rx.text(f"{item['name']} - ${item['price']}")
                )
            )
        ),
    )
```

---

## Substates

### Why Substates?

Substates allow you to:
- Organize complex state into modules
- Share state between features
- Create reusable state logic

### Creating Substates

```python
# Base state
class State(rx.State):
    """Main application state."""
    pass

# Authentication substate
class AuthState(State):
    """Authentication state (substate)."""
    access_token: str = ""
    refresh_token: str = ""
    user: dict | None = None

    @rx.var
    def is_authenticated(self) -> bool:
        """Check if user is authenticated."""
        return self.user is not None

    async def login(self, email: str, password: str):
        """Login user."""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "http://localhost:8000/auth/login",
                json={"email": email, "password": password}
            )
            data = response.json()
            self.access_token = data["access_token"]
            self.refresh_token = data["refresh_token"]
            self.user = data["user"]

    def logout(self):
        """Logout user."""
        self.access_token = ""
        self.refresh_token = ""
        self.user = None

# Posts substate
class PostState(State):
    """Post management state (substate)."""
    posts: list[dict] = []
    loading: bool = False

    async def load_posts(self):
        """Load posts (can access AuthState)."""
        self.loading = True
        auth_state = await self.get_state(AuthState)

        async with httpx.AsyncClient() as client:
            response = await client.get(
                "http://localhost:8000/posts/",
                headers={"Authorization": f"Bearer {auth_state.access_token}"}
            )
            self.posts = response.json()["items"]
            self.loading = False
```

### Accessing Parent State from Substate

```python
class UserPreferencesState(State):
    """User preferences substate."""
    theme: str = "light"
    language: str = "en"

    async def save_preferences(self):
        """Save preferences (access parent AuthState)."""
        auth_state = await self.get_state(AuthState)

        if not auth_state.is_authenticated:
            return

        async with httpx.AsyncClient() as client:
            await client.put(
                "http://localhost:8000/users/me/preferences",
                json={"theme": self.theme, "language": self.language},
                headers={"Authorization": f"Bearer {auth_state.access_token}"}
            )
```

---

## State Lifecycle

### State Initialization

```python
class AppState(rx.State):
    """Application state with initialization."""
    initialized: bool = False
    data: list[dict] = []

    async def on_load(self):
        """
        Called when state is first created.

        Use for initial data loading.
        """
        if not self.initialized:
            await self.load_initial_data()
            self.initialized = True

    async def load_initial_data(self):
        """Load initial data from API."""
        async with httpx.AsyncClient() as client:
            response = await client.get("http://localhost:8000/data")
            self.data = response.json()
```

### Using on_load in Pages

```python
@rx.page(route="/dashboard", on_load=AppState.on_load)
def dashboard() -> rx.Component:
    """
    Dashboard page.

    AppState.on_load runs before page renders.
    """
    return rx.container(
        rx.heading("Dashboard"),
        rx.cond(
            AppState.initialized,
            rx.text(f"Loaded {len(AppState.data)} items"),
            rx.spinner()
        )
    )
```

---

## Best Practices

### 1. Type Everything

```python
from typing import Any

class TypedState(rx.State):
    """State with full type hints."""
    items: list[dict[str, Any]] = []
    selected: int | None = None
    count: int = 0
    loading: bool = False

    def select_item(self, item_id: int) -> None:
        """Select item with type hints."""
        self.selected = item_id
```

### 2. Keep Event Handlers Simple

```python
# ✅ GOOD: Simple event handler
class GoodState(rx.State):
    name: str = ""

    def set_name(self, value: str):
        """Simple setter."""
        self.name = value

# ❌ BAD: Complex logic in handler
class BadState(rx.State):
    name: str = ""

    def set_name(self, value: str):
        # Too much logic here
        self.name = value.strip().title()
        self.validate_name()
        self.save_to_database()
        self.update_cache()
```

### 3. Use Computed Vars for Derived Data

```python
# ✅ GOOD: Computed var
class GoodState(rx.State):
    items: list[dict] = []

    @rx.var
    def active_items(self) -> list[dict]:
        """Computed var for active items."""
        return [item for item in self.items if item["active"]]

# ❌ BAD: Manual state update
class BadState(rx.State):
    items: list[dict] = []
    active_items: list[dict] = []  # Don't do this!

    def update_items(self, items: list[dict]):
        self.items = items
        # Manual sync - error prone!
        self.active_items = [item for item in items if item["active"]]
```

### 4. Handle Errors Gracefully

```python
class SafeState(rx.State):
    """State with proper error handling."""
    data: list[dict] = []
    loading: bool = False
    error: str = ""

    async def load_data(self):
        """Load data with error handling."""
        self.loading = True
        self.error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get("http://localhost:8000/data")
                response.raise_for_status()
                self.data = response.json()
        except httpx.HTTPStatusError as e:
            self.error = f"HTTP {e.response.status_code}: {e.response.text}"
        except httpx.RequestError as e:
            self.error = f"Request failed: {str(e)}"
        except Exception as e:
            self.error = f"Unexpected error: {str(e)}"
        finally:
            self.loading = False
```

### 5. Use Substates for Organization

```python
# ✅ GOOD: Organized substates
class State(rx.State):
    """Main state."""
    pass

class AuthState(State):
    """Auth logic."""
    pass

class DataState(State):
    """Data logic."""
    pass

# ❌ BAD: Everything in one state
class MonolithState(rx.State):
    """Everything crammed together."""
    # Auth
    user: dict = {}
    token: str = ""
    # Data
    posts: list = []
    comments: list = []
    # UI
    modal_open: bool = False
    # ... 100 more fields
```

---

## Key Takeaways

1. **State is backend** - Lives on Python server, syncs via WebSocket
2. **Event handlers update state** - Sync or async methods
3. **Computed vars for derived data** - Cached, automatic updates
4. **Substates for organization** - Modular state management
5. **Type everything** - Better IDE support and fewer bugs
6. **Handle errors** - Always use try/except in async handlers
7. **Keep handlers simple** - One responsibility per method
