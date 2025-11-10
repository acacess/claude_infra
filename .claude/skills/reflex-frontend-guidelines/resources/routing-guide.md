# Routing Guide - Reflex Navigation

Complete guide to routing in Reflex applications.

## Basic Routing

### Page Decorator

```python
import reflex as rx

@rx.page(route="/", title="Home")
def index() -> rx.Component:
    return rx.heading("Welcome Home")

@rx.page(route="/about", title="About Us")
def about() -> rx.Component:
    return rx.heading("About Us")
```

### Dynamic Routes

```python
@rx.page(route="/posts/[id]", title="Post Detail")
def post_detail() -> rx.Component:
    """Dynamic route with ID parameter."""
    return rx.container(
        rx.heading(f"Post {PostState.current_id}"),
        # Content...
    )

class PostState(rx.State):
    current_id: str = ""

    @rx.var
    def post_id_from_url(self) -> str:
        """Get ID from URL."""
        return self.router.page.params.get("id", "")
```

## Navigation

### Links

```python
def navbar() -> rx.Component:
    return rx.hstack(
        rx.link("Home", href="/"),
        rx.link("Posts", href="/posts"),
        rx.link("About", href="/about"),
        spacing="4"
    )
```

### Programmatic Navigation

```python
class NavigationState(rx.State):
    def go_to_home(self):
        return rx.redirect("/")

    def go_to_post(self, post_id: int):
        return rx.redirect(f"/posts/{post_id}")

# Usage
rx.button("Go Home", on_click=NavigationState.go_to_home)
```

## Key Takeaways

1. **@rx.page** decorator for pages
2. **[param]** for dynamic routes
3. **rx.link** for navigation
4. **rx.redirect** for programmatic navigation
5. **router.page.params** to access URL parameters
