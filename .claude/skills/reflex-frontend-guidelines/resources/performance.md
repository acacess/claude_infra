# Performance Optimization - Reflex Best Practices

Complete guide to optimizing Reflex application performance.

## Computed Vars

### Cache Expensive Calculations

```python
import reflex as rx

class DataState(rx.State):
    items: list[dict] = []
    filter_term: str = ""

    @rx.var
    def filtered_items(self) -> list[dict]:
        """Cached - only recomputes when items or filter_term changes."""
        if not self.filter_term:
            return self.items

        return [
            item for item in self.items
            if self.filter_term.lower() in item["name"].lower()
        ]

    @rx.var
    def total_price(self) -> float:
        """Cached calculation."""
        return sum(item["price"] * item["qty"] for item in self.filtered_items)
```

## Minimize State Updates

```python
# ✅ GOOD: Single state update
class GoodState(rx.State):
    def update_user(self, data: dict):
        self.name = data["name"]
        self.email = data["email"]
        self.age = data["age"]
        # UI re-renders once after method completes

# ❌ BAD: Multiple methods = multiple updates
class BadState(rx.State):
    def update_name(self, name: str):
        self.name = name  # Re-render

    def update_email(self, email: str):
        self.email = email  # Re-render

    def update_age(self, age: int):
        self.age = age  # Re-render
```

## Async Operations

### Use Async for I/O

```python
# ✅ GOOD: Async I/O
class AsyncState(rx.State):
    async def load_data(self):
        """Non-blocking I/O."""
        async with httpx.AsyncClient() as client:
            response = await client.get("/api/data")
            self.data = response.json()

# ❌ BAD: Blocking I/O
class BlockingState(rx.State):
    def load_data(self):
        """Blocks the event loop!"""
        response = requests.get("/api/data")
        self.data = response.json()
```

## Component Optimization

### Keep Components Small

```python
# ✅ GOOD: Small, focused components
def user_avatar(url: str) -> rx.Component:
    return rx.avatar(src=url)

def user_info(name: str, email: str) -> rx.Component:
    return rx.vstack(
        rx.heading(name),
        rx.text(email),
    )

def user_card(user: dict) -> rx.Component:
    return rx.hstack(
        user_avatar(user["avatar"]),
        user_info(user["name"], user["email"]),
    )

# ❌ BAD: Monolithic component
def everything() -> rx.Component:
    # 500 lines of mixed concerns
    ...
```

## Pagination

### Load Data in Chunks

```python
class PostState(rx.State):
    posts: list[dict] = []
    page: int = 1
    loading: bool = False

    async def load_more(self):
        """Load next page of posts."""
        if self.loading:
            return

        self.loading = True

        try:
            data = await fetch_from_api(
                f"/posts/?skip={(self.page - 1) * 20}&limit=20"
            )
            self.posts.extend(data["items"])
            self.page += 1
        finally:
            self.loading = False

# Usage
rx.button("Load More", on_click=PostState.load_more)
```

## Key Takeaways

1. **Computed vars** - cache derived values
2. **Minimize state updates** - batch changes
3. **Async for I/O** - don't block event loop
4. **Small components** - easier to optimize
5. **Pagination** - load data incrementally
6. **Type hints** - better performance hints
