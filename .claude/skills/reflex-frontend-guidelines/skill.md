---
name: reflex-frontend-guidelines
description: Frontend development guidelines for Reflex (Python web framework). Modern patterns including Reflex State classes, component organization, rx.* components, routing, styling, data fetching, performance optimization, and Python type hints. Use when creating pages, components, state management, routing, styling, or working with Reflex frontend code.
---

# Reflex Frontend Development Guidelines

## Purpose

Comprehensive guide for Reflex development using Python 3.12+, emphasizing clean state management, component organization, and performance optimization.

## When to Use This Skill

Automatically activates when working on:
- Creating new Reflex pages or components
- Building new features with State classes
- Managing application state
- Setting up routing
- Styling Reflex components
- Data fetching from APIs
- Performance optimization
- Organizing Reflex code

---

## Quick Start

### New Page Checklist

Creating a Reflex page? Follow this checklist:

- [ ] Use `@rx.page()` decorator with route and title
- [ ] Consume backend data through typed client helpers (never raw dicts)
- [ ] Create corresponding State class if needed
- [ ] Use `rx.*` components for UI
- [ ] Type hints on all State properties and methods
- [ ] Follow file organization (features/ pattern)
- [ ] Use Reflex styling system (style dicts or Tailwind)
- [ ] Implement error handling for API calls
- [ ] Add loading states for async operations
- [ ] Capture and report errors via shared Sentry helper

### New Feature Checklist

Creating a feature? Set up this structure:

- [ ] Create `features/{feature-name}/` directory
- [ ] Create subdirectories: `states/`, `components/`, `api/`, `utils/`
- [ ] Create State class in `states/{feature}_state.py`
- [ ] Create components in `components/`
- [ ] Create typed API client in `api/{feature}_api.py` using shared schemas
- [ ] Export public API from feature `__init__.py`
- [ ] Add routes to main app
- [ ] Wire feature config (API base URL, Supabase keys) through centralized settings
- [ ] Add component/state tests (unit + e2e) that exercise FastAPI endpoints

---

## Architecture Overview

### Reflex State Management

```
User Interaction (Frontend)
    ↓
Event Handler (State method)
    ↓
State Update (Python backend)
    ↓
WebSocket Sync
    ↓
UI Re-render (Frontend)
```

**Key Principle:** State lives on the backend, synced via WebSocket.

---

## Directory Structure

```
app/
├── pages/                 # Page components
│   ├── index.py          # Home page
│   ├── about.py          # About page
│   └── posts/
│       ├── index.py      # Post list
│       └── [id].py       # Post detail
├── features/             # Feature modules
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── states/
│   │   │   └── auth_state.py
│   │   ├── components/
│   │   │   ├── login_form.py
│   │   │   └── register_form.py
│   │   └── api/
│   │       └── auth_api.py
│   └── posts/
│       ├── __init__.py
│       ├── states/
│       │   └── post_state.py
│       ├── components/
│       │   └── post_card.py
│       └── api/
│           └── post_api.py
├── components/           # Shared components
│   ├── navbar.py
│   └── footer.py
├── styles/               # Global styles
│   └── theme.py
├── utils/                # Utilities
│   └── api_client.py
└── app.py                # Main app
```

**Naming Conventions:**
- Pages: `snake_case.py` - `index.py`, `about.py`
- States: `PascalCase` class - `PostState`, `AuthState`
- Components: `snake_case` function - `navbar()`, `post_card()`
- Files: `snake_case.py`

---

## Core Principles (8 Key Rules)

### 1. State Classes for All State Management

```python
import reflex as rx

# ❌ NEVER: Global variables for state
posts = []

# ✅ ALWAYS: State class
class PostState(rx.State):
    posts: list[dict] = []
    loading: bool = False
    error: str = ""

    async def load_posts(self):
        """Load posts from API."""
        self.loading = True
        try:
            # API call
            self.posts = await fetch_posts()
        except Exception as e:
            self.error = str(e)
        finally:
            self.loading = False
```

### 2. Type Hints on Everything

```python
from typing import Any

class UserState(rx.State):
    users: list[dict[str, Any]] = []
    current_user: dict[str, Any] | None = None
    page: int = 1

    async def get_user(self, user_id: int) -> None:
        """Get user by ID with type hints."""
        ...
```

### 3. Use rx.* Components

```python
import reflex as rx

def user_card(user: dict) -> rx.Component:
    """User card component using Reflex components."""
    return rx.card(
        rx.heading(user["name"], size="5"),
        rx.text(user["email"]),
        rx.button("View Profile", on_click=UserState.view_profile),
    )
```

### 4. Event Handlers are Async

```python
class DataState(rx.State):
    data: list = []

    async def fetch_data(self):
        """Async event handler for data fetching."""
        async with httpx.AsyncClient() as client:
            response = await client.get("/api/data")
            self.data = response.json()
```

### 5. Component Functions Return rx.Component

```python
def navbar() -> rx.Component:
    """Navbar component with explicit return type."""
    return rx.hstack(
        rx.heading("My App"),
        rx.spacer(),
        rx.link("Home", href="/"),
        rx.link("About", href="/about"),
    )
```

### 6. Pages Use @rx.page Decorator

```python
import reflex as rx

@rx.page(route="/posts", title="Posts")
def posts_page() -> rx.Component:
    """Posts listing page."""
    return rx.container(
        rx.heading("All Posts"),
        # ... content
    )
```

### 7. Style with Dicts or Tailwind

```python
# Style dict approach
card_style = {
    "padding": "1rem",
    "border_radius": "0.5rem",
    "background": "#f0f0f0",
}

def styled_card() -> rx.Component:
    return rx.box(
        rx.text("Content"),
        style=card_style
    )

# Tailwind approach (if enabled)
def tailwind_card() -> rx.Component:
    return rx.box(
        rx.text("Content"),
        class_name="p-4 rounded-lg bg-gray-100"
    )
```

### 8. Organize by Features, Not Type

```python
# ✅ GOOD: Feature-based
features/
  posts/
    states/post_state.py
    components/post_card.py
    api/post_api.py

# ❌ BAD: Type-based
states/
  post_state.py
  user_state.py
components/
  post_card.py
  user_card.py
```

### 9. Typed API Clients Mirror Backend Schemas

```python
from typing import Annotated
import httpx
from pydantic import BaseModel

from app.config import settings

class PostResponse(BaseModel):
    id: int
    title: str
    body: str

_client = httpx.AsyncClient(base_url=settings.api_base_url, timeout=settings.api_timeout)

async def fetch_posts() -> list[PostResponse]:
    """Call FastAPI backend using shared schema."""
    response = await _client.get("/posts", headers=settings.auth_headers())
    response.raise_for_status()
    return [PostResponse.model_validate(obj) for obj in response.json()]
```

**Why:** Backend guideline mandates Pydantic validation on both sides. Keep request/response models in sync (ideally shared package or generated client) so regressions surface early.

---

## Shared Configuration & Environment Handling

- Create a single `app/config.py` (or `features/config.py`) that loads environment values via `pydantic-settings`, matching backend practice. Import this config everywhere instead of `os.getenv`.
- Standard fields: `api_base_url`, `supabase_project_url`, `supabase_public_key`, `api_timeout_ms`, `sentry_dsn`, feature flags.
- Provide helper methods like `auth_headers()` to add Supabase access tokens/JWTs automatically, so components/states never duplicate header logic.
- Support per-environment overrides (local/dev/staging/prod) that mirror backend `settings.env`.

```python
from pydantic_settings import BaseSettings

class FrontendSettings(BaseSettings):
    api_base_url: str
    api_timeout: float = 10.0
    sentry_dsn: str | None = None
    supabase_url: str
    supabase_anon_key: str

settings = FrontendSettings()  # automatically reads env vars / .env files
```

---

## Observability & Error Handling

- Initialize Sentry (or equivalent) in Reflex startup code using the same DSN/project as FastAPI so traces can be linked.
- Wrap async state methods that call the backend in `try/except`, call `sentry_sdk.capture_exception(err)`, set human-friendly error state, and show a toast/banner in the UI.
- For API clients, centralize response handling so HTTP status mapping (200/201/400/401/etc.) mirrors backend `status` constants; bubble validation errors back to form states.
- Use structured logging (Python `logging` module) for local debugging; avoid `print`.

```python
import sentry_sdk
from sentry_sdk.integrations.asyncio import AsyncioIntegration

sentry_sdk.init(
    dsn=settings.sentry_dsn,
    integrations=[AsyncioIntegration()],
    traces_sample_rate=1.0,
)
```

---

## Authentication & Security Alignment

- Store auth tokens in Reflex state (never localStorage). Example: `AuthState` keeps `access_token`, `refresh_token`, and derived `current_user`.
- `AuthState` should expose helpers (`auth_headers`) consumed by every API client; tokens come from Supabase auth responses or FastAPI login endpoints.
- Refresh tokens proactively before expiration—call the backend refresh route from a background task and update state atomically.
- Mirror backend middleware expectations: include CSRF headers if required, send Supabase session info when using Row Level Security, and clear state on 401 responses.

```python
class AuthState(rx.State):
    access_token: str | None = None
    refresh_token: str | None = None

    async def login(self, credentials: LoginRequest) -> None:
        try:
            tokens = await auth_api.login(credentials)
        except httpx.HTTPStatusError as err:
            sentry_sdk.capture_exception(err)
            self.error = "Login failed"
            return
        self.access_token = tokens.access_token
        self.refresh_token = tokens.refresh_token
```

---

## Testing Strategy

- **Unit tests:** Use `pytest` + `reflex.testing` utilities to exercise component functions and state methods. Mock API clients to verify that state transitions follow backend contracts.
- **Integration tests:** Run Playwright/Cypress e2e flows against a FastAPI instance seeded with Supabase data. Cover auth, CRUD flows, and error scenarios.
- **Contract tests:** When backend schemas change, regenerate shared Pydantic models or client code and run snapshot tests to ensure the frontend still parses responses correctly.
- Add a short checklist per feature (`tests/{feature}_test.py`) verifying pages render, loading/error states flip, and Sentry gets called on failures.

---

## Common Imports

```python
# Reflex
import reflex as rx

# Typing
from typing import Any

# Async HTTP
import httpx

# Date/time
from datetime import datetime

# Data models
from pydantic import BaseModel
```

---

## Quick Reference

### Reflex Components

| Component | Usage |
|-----------|-------|
| `rx.box` | Container div |
| `rx.container` | Centered container |
| `rx.hstack` | Horizontal stack |
| `rx.vstack` | Vertical stack |
| `rx.heading` | Heading (size="1" to "9") |
| `rx.text` | Text/paragraph |
| `rx.button` | Button |
| `rx.input` | Text input |
| `rx.link` | Hyperlink |
| `rx.card` | Card container |
| `rx.spacer` | Flexible space |

### Event Handling

```python
# On click
rx.button("Click", on_click=StateClass.handler)

# On change
rx.input(on_change=StateClass.set_value)

# On submit
rx.form(on_submit=StateClass.submit_form)

# With lambda (for parameters)
rx.button("Delete", on_click=lambda: StateClass.delete_item(item_id))
```

---

## Anti-Patterns to Avoid

❌ Global variables for state
❌ Missing type hints
❌ Synchronous API calls in State
❌ Direct HTML/CSS strings
❌ Type-based file organization
❌ Missing error handling
❌ No loading states
❌ Overly complex component functions

---

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](resources/architecture-overview.md) |
| Create components | [component-patterns.md](resources/component-patterns.md) |
| Manage state | [state-management.md](resources/state-management.md) |
| Set up routing | [routing-guide.md](resources/routing-guide.md) |
| Style components | [styling-guide.md](resources/styling-guide.md) |
| Fetch data from APIs | [data-fetching.md](resources/data-fetching.md) |
| Organize files | [file-organization.md](resources/file-organization.md) |
| Handle forms | [forms-and-validation.md](resources/forms-and-validation.md) |
| Optimize performance | [performance.md](resources/performance.md) |
| See examples | [complete-examples.md](resources/complete-examples.md) |

---

## Resource Files

### [architecture-overview.md](resources/architecture-overview.md)
Reflex architecture, state synchronization, component lifecycle

### [component-patterns.md](resources/component-patterns.md)
Component structure, composition, reusability patterns

### [state-management.md](resources/state-management.md)
State classes, computed vars, event handlers, substates

### [routing-guide.md](resources/routing-guide.md)
Page decorator, dynamic routes, navigation

### [styling-guide.md](resources/styling-guide.md)
Style dicts, Tailwind integration, theming, responsive design

### [data-fetching.md](resources/data-fetching.md)
Async API calls, httpx usage, error handling, caching

### [file-organization.md](resources/file-organization.md)
Features directory, component organization, imports

### [forms-and-validation.md](resources/forms-and-validation.md)
Form handling, validation, Pydantic integration

### [performance.md](resources/performance.md)
Optimization strategies, memoization, lazy loading

### [complete-examples.md](resources/complete-examples.md)
Full working examples, end-to-end implementations

---

## Related Skills

- **fastapi-backend-guidelines** - Backend APIs that Reflex frontend consumes (FastAPI + Supabase)
- **error-tracking** - Sentry integration (works in Reflex too)
- **skill-developer** - Meta-skill for creating and managing skills

**Note**: This frontend is designed to work with **FastAPI + Supabase backend** (see `fastapi-backend-guidelines`). The API endpoints, request/response formats, and authentication patterns are aligned between both skill sets. Tech stack: **FastAPI + Supabase + Reflex**.

---

**Skill Status**: COMPLETE ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 10 resource files ✅
