# Component Patterns - Reflex Best Practices

Complete guide to creating reusable and maintainable Reflex components.

## Component Basics

### Component Function Pattern

```python
import reflex as rx

def user_card(name: str, email: str) -> rx.Component:
    """
    User card component.

    Args:
        name: User's full name
        email: User's email address
    """
    return rx.card(
        rx.vstack(
            rx.heading(name, size="5"),
            rx.text(email, color="gray"),
            spacing="2",
        ),
        width="300px",
    )
```

**Key Points:**
- Function that returns `rx.Component`
- Type hints on parameters and return type
- Docstring explaining the component
- Use `rx.*` components for UI

### Component with State

```python
class UserState(rx.State):
    """State for user management."""
    users: list[dict] = []
    loading: bool = False

def user_list() -> rx.Component:
    """Display list of users."""
    return rx.cond(
        UserState.loading,
        rx.spinner(),
        rx.vstack(
            rx.foreach(
                UserState.users,
                lambda user: user_card(user["name"], user["email"])
            ),
            spacing="4",
        )
    )
```

## Composition Patterns

### Nested Components

```python
def post_header(title: str, author: str, date: str) -> rx.Component:
    """Post header component."""
    return rx.hstack(
        rx.heading(title, size="6"),
        rx.spacer(),
        rx.vstack(
            rx.text(f"By {author}", size="2"),
            rx.text(date, size="1", color="gray"),
            align="end",
        ),
        width="100%",
    )

def post_content(content: str) -> rx.Component:
    """Post content component."""
    return rx.text(content, size="3")

def post_card(post: dict) -> rx.Component:
    """Complete post card using nested components."""
    return rx.card(
        rx.vstack(
            post_header(post["title"], post["author"], post["date"]),
            post_content(post["content"]),
            spacing="4",
        ),
        width="100%",
    )
```

### Component with Children

```python
def section(
    heading: str,
    *children: rx.Component,
    **props
) -> rx.Component:
    """
    Reusable section component.

    Args:
        heading: Section heading
        *children: Child components
        **props: Additional props for the container
    """
    return rx.box(
        rx.heading(heading, size="7", margin_bottom="4"),
        rx.vstack(*children, spacing="4"),
        padding="6",
        **props
    )

# Usage
def dashboard() -> rx.Component:
    return section(
        "Dashboard",
        rx.text("Welcome back!"),
        rx.button("Get Started"),
        background="gray.50",
    )
```

## Conditional Rendering

### Using rx.cond

```python
def user_profile() -> rx.Component:
    """User profile with conditional rendering."""
    return rx.container(
        rx.cond(
            UserState.user.is_some(),
            # User is logged in
            rx.vstack(
                rx.heading(f"Welcome, {UserState.user['name']}"),
                rx.button("Logout", on_click=UserState.logout),
            ),
            # User is not logged in
            rx.vstack(
                rx.heading("Please log in"),
                rx.button("Login", on_click=lambda: rx.redirect("/login")),
            ),
        )
    )
```

### Multiple Conditions

```python
def status_badge(status: str) -> rx.Component:
    """Badge with color based on status."""
    return rx.match(
        status,
        ("active", rx.badge("Active", color="green")),
        ("pending", rx.badge("Pending", color="yellow")),
        ("inactive", rx.badge("Inactive", color="gray")),
        rx.badge(status, color="blue"),  # default
    )
```

## List Rendering

### Using rx.foreach

```python
class PostState(rx.State):
    posts: list[dict] = []

def post_list() -> rx.Component:
    """Render list of posts."""
    return rx.vstack(
        rx.foreach(
            PostState.posts,
            lambda post: rx.card(
                rx.heading(post["title"]),
                rx.text(post["content"]),
            )
        ),
        spacing="4",
    )
```

### With Index

```python
def numbered_list(items: list[str]) -> rx.Component:
    """List with numbers."""
    return rx.vstack(
        rx.foreach(
            enumerate(items, start=1),
            lambda item: rx.hstack(
                rx.text(f"{item[0]}.", weight="bold"),
                rx.text(item[1]),
            )
        ),
        spacing="2",
    )
```

## Event Handlers

### Simple Events

```python
class CounterState(rx.State):
    count: int = 0

    def increment(self):
        self.count += 1

def counter() -> rx.Component:
    return rx.hstack(
        rx.button("-", on_click=CounterState.decrement),
        rx.text(CounterState.count, size="5"),
        rx.button("+", on_click=CounterState.increment),
    )
```

### Events with Parameters

```python
class TodoState(rx.State):
    todos: list[dict] = []

    def delete_todo(self, todo_id: int):
        """Delete todo by ID."""
        self.todos = [t for t in self.todos if t["id"] != todo_id]

def todo_list() -> rx.Component:
    return rx.vstack(
        rx.foreach(
            TodoState.todos,
            lambda todo: rx.hstack(
                rx.text(todo["text"]),
                rx.button(
                    "Delete",
                    on_click=lambda: TodoState.delete_todo(todo["id"])
                ),
            )
        )
    )
```

## Styling Components

### Inline Styles

```python
def styled_button(text: str) -> rx.Component:
    """Button with custom styles."""
    return rx.button(
        text,
        background="blue.500",
        color="white",
        padding="4",
        border_radius="lg",
        _hover={
            "background": "blue.600",
        },
    )
```

### Style Dictionary

```python
card_style = {
    "padding": "6",
    "border_radius": "lg",
    "box_shadow": "md",
    "background": "white",
}

def info_card(title: str, content: str) -> rx.Component:
    """Card with predefined styles."""
    return rx.box(
        rx.heading(title, size="5"),
        rx.text(content),
        **card_style
    )
```

## Best Practices

### 1. Keep Components Small and Focused

```python
# ✅ GOOD: Small, single-purpose components
def user_avatar(url: str, size: str = "md") -> rx.Component:
    return rx.avatar(src=url, size=size)

def user_name(name: str) -> rx.Component:
    return rx.heading(name, size="5")

def user_card(user: dict) -> rx.Component:
    return rx.hstack(
        user_avatar(user["avatar"]),
        user_name(user["name"]),
    )

# ❌ BAD: Monolithic component
def everything_component():
    return rx.vstack(
        # 200 lines of mixed concerns
        ...
    )
```

### 2. Use Type Hints

```python
# ✅ GOOD: Full type hints
from typing import Any

def product_card(
    product: dict[str, Any],
    on_add_to_cart: callable | None = None
) -> rx.Component:
    ...

# ❌ BAD: No type hints
def product_card(product, on_add_to_cart=None):
    ...
```

### 3. Document Your Components

```python
# ✅ GOOD: Clear docstring
def alert_box(
    message: str,
    severity: str = "info"
) -> rx.Component:
    """
    Display an alert message.

    Args:
        message: The alert message to display
        severity: Alert severity (info, warning, error, success)

    Returns:
        Alert component
    """
    ...
```

### 4. Use Computed Vars for Derived UI

```python
class ShoppingState(rx.State):
    items: list[dict] = []

    @rx.var
    def total_price(self) -> float:
        """Calculate total - cached until items change."""
        return sum(item["price"] * item["qty"] for item in self.items)

def cart_total() -> rx.Component:
    return rx.text(f"Total: ${ShoppingState.total_price:.2f}")
```

## Key Takeaways

1. **Component functions** return `rx.Component`
2. **Type hints** on all parameters and returns
3. **Composition** - build complex UIs from simple components
4. **rx.cond** for conditional rendering
5. **rx.foreach** for list rendering
6. **Event handlers** with lambda for parameters
7. **Keep components small** - single responsibility
8. **Document components** with docstrings
