# Complete Examples - Full Reflex Implementation

End-to-end working examples demonstrating all Reflex patterns together.

## Table of Contents

- [Complete Feature: Blog Posts](#complete-feature-blog-posts)
- [Authentication Flow](#authentication-flow)
- [Form with Validation](#form-with-validation)

---

## Complete Feature: Blog Posts

### 1. State Class

```python
# features/posts/states/post_state.py
import reflex as rx
import httpx
from typing import Any

class PostState(rx.State):
    """State for blog post management."""

    # Data
    posts: list[dict[str, Any]] = []
    current_post: dict[str, Any] | None = None

    # UI State
    loading: bool = False
    error: str = ""
    success_message: str = ""

    # Form state
    form_title: str = ""
    form_content: str = ""
    form_is_published: bool = False

    # Computed vars
    @rx.var
    def published_posts(self) -> list[dict[str, Any]]:
        """Get only published posts."""
        return [p for p in self.posts if p.get("is_published", False)]

    @rx.var
    def draft_posts(self) -> list[dict[str, Any]]:
        """Get only draft posts."""
        return [p for p in self.posts if not p.get("is_published", False)]

    @rx.var
    def posts_count(self) -> int:
        """Total number of posts."""
        return len(self.posts)

    @rx.var
    def form_is_valid(self) -> bool:
        """Check if form is valid."""
        return bool(self.form_title.strip() and self.form_content.strip())

    # Event handlers
    async def load_posts(self):
        """Load all posts from API."""
        self.loading = True
        self.error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get("http://localhost:8000/posts/")
                response.raise_for_status()
                data = response.json()
                self.posts = data.get("items", [])
        except httpx.HTTPError as e:
            self.error = f"Failed to load posts: {str(e)}"
        except Exception as e:
            self.error = f"Unexpected error: {str(e)}"
        finally:
            self.loading = False

    async def load_post(self, post_id: int):
        """Load single post by ID."""
        self.loading = True
        self.error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(f"http://localhost:8000/posts/{post_id}")
                response.raise_for_status()
                self.current_post = response.json()
        except httpx.HTTPError as e:
            self.error = f"Failed to load post: {str(e)}"
        finally:
            self.loading = False

    async def create_post(self):
        """Create a new post."""
        if not self.form_is_valid:
            self.error = "Please fill in all required fields"
            return

        self.loading = True
        self.error = ""
        self.success_message = ""

        try:
            # Get auth token (assuming AuthState exists)
            auth_state = await self.get_state(AuthState)

            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "http://localhost:8000/posts/",
                    json={
                        "title": self.form_title,
                        "content": self.form_content,
                        "is_published": self.form_is_published,
                    },
                    headers={"Authorization": f"Bearer {auth_state.access_token}"}
                )
                response.raise_for_status()

                # Clear form
                self.form_title = ""
                self.form_content = ""
                self.form_is_published = False

                self.success_message = "Post created successfully!"

                # Reload posts
                await self.load_posts()

        except httpx.HTTPError as e:
            self.error = f"Failed to create post: {str(e)}"
        finally:
            self.loading = False

    async def update_post(self, post_id: int, title: str, content: str, is_published: bool):
        """Update an existing post."""
        self.loading = True
        self.error = ""

        try:
            auth_state = await self.get_state(AuthState)

            async with httpx.AsyncClient() as client:
                response = await client.put(
                    f"http://localhost:8000/posts/{post_id}",
                    json={"title": title, "content": content, "is_published": is_published},
                    headers={"Authorization": f"Bearer {auth_state.access_token}"}
                )
                response.raise_for_status()

                self.success_message = "Post updated successfully!"
                await self.load_posts()

        except httpx.HTTPError as e:
            self.error = f"Failed to update post: {str(e)}"
        finally:
            self.loading = False

    async def delete_post(self, post_id: int):
        """Delete a post."""
        self.loading = True
        self.error = ""

        try:
            auth_state = await self.get_state(AuthState)

            async with httpx.AsyncClient() as client:
                response = await client.delete(
                    f"http://localhost:8000/posts/{post_id}",
                    headers={"Authorization": f"Bearer {auth_state.access_token}"}
                )
                response.raise_for_status()

                self.success_message = "Post deleted successfully!"
                # Remove from local state
                self.posts = [p for p in self.posts if p["id"] != post_id]

        except httpx.HTTPError as e:
            self.error = f"Failed to delete post: {str(e)}"
        finally:
            self.loading = False

    def set_form_title(self, value: str):
        """Set form title."""
        self.form_title = value

    def set_form_content(self, value: str):
        """Set form content."""
        self.form_content = value

    def set_form_is_published(self, value: bool):
        """Set form is_published."""
        self.form_is_published = value

    def clear_messages(self):
        """Clear error and success messages."""
        self.error = ""
        self.success_message = ""
```

### 2. Components

```python
# features/posts/components/post_card.py
import reflex as rx
from ..states.post_state import PostState

def post_card(post: dict) -> rx.Component:
    """Post card component."""
    return rx.card(
        rx.vstack(
            rx.hstack(
                rx.heading(post["title"], size="5"),
                rx.spacer(),
                rx.badge(
                    "Published" if post["is_published"] else "Draft",
                    color="green" if post["is_published"] else "gray"
                ),
            ),
            rx.text(post["content"][:200] + "..." if len(post["content"]) > 200 else post["content"]),
            rx.text(f"By: User #{post['author_id']}", size="1", color="gray"),
            rx.hstack(
                rx.button(
                    "View",
                    on_click=lambda: rx.redirect(f"/posts/{post['id']}")
                ),
                rx.button(
                    "Delete",
                    color="red",
                    on_click=lambda: PostState.delete_post(post["id"])
                ),
                spacing="2",
            ),
            spacing="3",
            width="100%",
        ),
        width="100%",
    )


# features/posts/components/post_form.py
def post_form() -> rx.Component:
    """Post creation form."""
    return rx.card(
        rx.vstack(
            rx.heading("Create New Post", size="6"),

            # Title input
            rx.vstack(
                rx.text("Title", weight="bold"),
                rx.input(
                    placeholder="Enter post title",
                    value=PostState.form_title,
                    on_change=PostState.set_form_title,
                    width="100%",
                ),
                width="100%",
            ),

            # Content textarea
            rx.vstack(
                rx.text("Content", weight="bold"),
                rx.text_area(
                    placeholder="Enter post content",
                    value=PostState.form_content,
                    on_change=PostState.set_form_content,
                    width="100%",
                    rows="6",
                ),
                width="100%",
            ),

            # Published checkbox
            rx.hstack(
                rx.checkbox(
                    checked=PostState.form_is_published,
                    on_change=PostState.set_form_is_published,
                ),
                rx.text("Publish immediately"),
            ),

            # Error message
            rx.cond(
                PostState.error != "",
                rx.callout(
                    PostState.error,
                    icon="triangle_alert",
                    color="red",
                ),
            ),

            # Success message
            rx.cond(
                PostState.success_message != "",
                rx.callout(
                    PostState.success_message,
                    icon="check",
                    color="green",
                ),
            ),

            # Submit button
            rx.button(
                "Create Post",
                on_click=PostState.create_post,
                disabled=~PostState.form_is_valid | PostState.loading,
                loading=PostState.loading,
                width="100%",
            ),

            spacing="4",
            width="100%",
        ),
        width="100%",
    )
```

### 3. Pages

```python
# pages/posts/index.py
import reflex as rx
from features.posts.states.post_state import PostState
from features.posts.components.post_card import post_card

@rx.page(
    route="/posts",
    title="Blog Posts",
    on_load=PostState.load_posts
)
def posts_page() -> rx.Component:
    """Posts listing page."""
    return rx.container(
        rx.vstack(
            # Header
            rx.heading("Blog Posts", size="8"),

            # Stats
            rx.hstack(
                rx.text(f"Total: {PostState.posts_count}"),
                rx.text(f"Published: {len(PostState.published_posts)}"),
                rx.text(f"Drafts: {len(PostState.draft_posts)}"),
                spacing="4",
            ),

            # Loading state
            rx.cond(
                PostState.loading,
                rx.spinner(),
            ),

            # Error state
            rx.cond(
                PostState.error != "",
                rx.callout(
                    PostState.error,
                    icon="triangle_alert",
                    color="red",
                ),
            ),

            # Post list
            rx.cond(
                PostState.posts_count > 0,
                rx.vstack(
                    rx.foreach(PostState.posts, post_card),
                    spacing="4",
                    width="100%",
                ),
                rx.text("No posts yet. Create your first post!"),
            ),

            spacing="6",
            width="100%",
            max_width="800px",
            padding="4",
        ),
        size="4",
    )


# pages/posts/new.py
from features.posts.components.post_form import post_form

@rx.page(route="/posts/new", title="Create Post")
def new_post_page() -> rx.Component:
    """New post creation page."""
    return rx.container(
        rx.vstack(
            rx.heading("Create New Post", size="8"),
            post_form(),
            spacing="6",
            width="100%",
            max_width="600px",
            padding="4",
        ),
        size="4",
    )


# pages/posts/[id].py
@rx.page(route="/posts/[id]", title="Post Detail", on_load=PostState.load_post)
def post_detail_page() -> rx.Component:
    """Individual post detail page."""
    return rx.container(
        rx.cond(
            PostState.loading,
            rx.spinner(),
            rx.cond(
                PostState.current_post.is_some(),
                rx.vstack(
                    # Back button
                    rx.link(
                        rx.button("← Back to Posts"),
                        href="/posts"
                    ),

                    # Post content
                    rx.card(
                        rx.vstack(
                            rx.hstack(
                                rx.heading(PostState.current_post["title"], size="8"),
                                rx.spacer(),
                                rx.badge(
                                    "Published" if PostState.current_post["is_published"] else "Draft"
                                ),
                            ),
                            rx.text(PostState.current_post["content"]),
                            rx.text(
                                f"Author: User #{PostState.current_post['author_id']}",
                                size="1",
                                color="gray"
                            ),
                            spacing="4",
                            width="100%",
                        )
                    ),

                    spacing="6",
                    width="100%",
                ),
                rx.text("Post not found"),
            ),
        ),
        size="4",
        padding="4",
    )
```

---

## Authentication Flow

```python
# features/auth/states/auth_state.py
import reflex as rx
import httpx
from typing import Any

class AuthState(rx.State):
    """Authentication state."""

    access_token: str = ""
    refresh_token: str = ""
    user: dict[str, Any] | None = None

    # Form state
    login_email: str = ""
    login_password: str = ""
    login_error: str = ""
    login_loading: bool = False

    @rx.var
    def is_authenticated(self) -> bool:
        """Check if user is authenticated."""
        return self.user is not None

    @rx.var
    def username(self) -> str:
        """Get current username."""
        return self.user.get("username", "") if self.user else ""

    async def login(self):
        """Login user."""
        self.login_loading = True
        self.login_error = ""

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "http://localhost:8000/auth/login",
                    json={
                        "email": self.login_email,
                        "password": self.login_password
                    }
                )
                response.raise_for_status()

                data = response.json()
                self.access_token = data["access_token"]
                self.refresh_token = data["refresh_token"]
                self.user = data["user"]

                # Clear form
                self.login_email = ""
                self.login_password = ""

                # Redirect to dashboard
                return rx.redirect("/dashboard")

        except httpx.HTTPError as e:
            self.login_error = "Invalid email or password"
        finally:
            self.login_loading = False

    def logout(self):
        """Logout user."""
        self.access_token = ""
        self.refresh_token = ""
        self.user = None
        return rx.redirect("/")


# pages/login.py
@rx.page(route="/login", title="Login")
def login_page() -> rx.Component:
    """Login page."""
    return rx.container(
        rx.card(
            rx.vstack(
                rx.heading("Login", size="8"),

                rx.vstack(
                    rx.text("Email", weight="bold"),
                    rx.input(
                        placeholder="email@example.com",
                        type="email",
                        value=AuthState.login_email,
                        on_change=AuthState.set_login_email,
                        width="100%",
                    ),
                    width="100%",
                ),

                rx.vstack(
                    rx.text("Password", weight="bold"),
                    rx.input(
                        placeholder="••••••••",
                        type="password",
                        value=AuthState.login_password,
                        on_change=AuthState.set_login_password,
                        width="100%",
                    ),
                    width="100%",
                ),

                rx.cond(
                    AuthState.login_error != "",
                    rx.callout(
                        AuthState.login_error,
                        icon="triangle_alert",
                        color="red",
                    ),
                ),

                rx.button(
                    "Login",
                    on_click=AuthState.login,
                    loading=AuthState.login_loading,
                    width="100%",
                ),

                spacing="4",
                width="100%",
            ),
            max_width="400px",
        ),
        size="2",
    )
```

---

## Key Takeaways

This complete example demonstrates:

1. **State Management**: Async event handlers, computed vars, form state
2. **Component Organization**: Reusable components with clear responsibilities
3. **Data Fetching**: HTTP calls with httpx, error handling
4. **Routing**: Multiple pages with @rx.page decorator
5. **Loading States**: Conditional rendering based on loading state
6. **Error Handling**: User-friendly error messages
7. **Type Safety**: Type hints throughout
8. **Substates**: Modular state organization (AuthState, PostState)
9. **Form Handling**: Controlled inputs with validation
10. **Real-world Patterns**: Authentication, CRUD operations, navigation
