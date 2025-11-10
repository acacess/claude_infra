# File Organization - Reflex Project Structure

Complete guide to organizing Reflex application files.

## Recommended Structure

```
app/
├── pages/                  # Page components (@rx.page)
│   ├── index.py           # Home page (/)
│   ├── about.py           # About page (/about)
│   └── posts/
│       ├── index.py       # Post list (/posts)
│       └── [id].py        # Post detail (/posts/[id])
├── features/              # Feature modules
│   ├── auth/
│   │   ├── __init__.py   # Public API exports
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
├── components/            # Shared components
│   ├── navbar.py
│   ├── footer.py
│   └── loading_spinner.py
├── styles/                # Global styles
│   └── theme.py
├── utils/                 # Utilities
│   └── api_client.py
└── app.py                # Main Reflex app

