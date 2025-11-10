# Forms and Validation - Reflex Form Handling

Complete guide to handling forms and validation in Reflex.

## Basic Form Pattern

```python
import reflex as rx

class LoginState(rx.State):
    """Login form state."""
    email: str = ""
    password: str = ""
    error: str = ""
    loading: bool = False

    def set_email(self, value: str):
        self.email = value

    def set_password(self, value: str):
        self.password = value

    async def login(self):
        """Handle login submission."""
        # Validation
        if not self.email or not self.password:
            self.error = "Email and password are required"
            return

        if "@" not in self.email:
            self.error = "Invalid email format"
            return

        self.loading = True
        self.error = ""

        try:
            # Call FastAPI backend
            response = await fetch_from_api(
                "/auth/login",
                method="POST",
                data={"email": self.email, "password": self.password}
            )

            # Store token
            self.access_token = response["access_token"]

            # Redirect to dashboard
            return rx.redirect("/dashboard")

        except Exception as e:
            self.error = "Invalid credentials"
        finally:
            self.loading = False

def login_form() -> rx.Component:
    """Login form component."""
    return rx.card(
        rx.vstack(
            rx.heading("Login", size="7"),

            # Email input
            rx.vstack(
                rx.text("Email", weight="bold"),
                rx.input(
                    placeholder="email@example.com",
                    type="email",
                    value=LoginState.email,
                    on_change=LoginState.set_email,
                    width="100%",
                ),
                width="100%",
                align="start",
            ),

            # Password input
            rx.vstack(
                rx.text("Password", weight="bold"),
                rx.input(
                    placeholder="••••••••",
                    type="password",
                    value=LoginState.password,
                    on_change=LoginState.set_password,
                    width="100%",
                ),
                width="100%",
                align="start",
            ),

            # Error message
            rx.cond(
                LoginState.error != "",
                rx.callout(
                    LoginState.error,
                    icon="triangle_alert",
                    color="red",
                ),
            ),

            # Submit button
            rx.button(
                "Login",
                on_click=LoginState.login,
                loading=LoginState.loading,
                width="100%",
            ),

            spacing="4",
            width="100%",
        ),
        max_width="400px",
    )
```

## Validation Patterns

### Client-Side Validation

```python
class SignupState(rx.State):
    email: str = ""
    password: str = ""
    confirm_password: str = ""

    @rx.var
    def email_is_valid(self) -> bool:
        """Check if email is valid."""
        return "@" in self.email and "." in self.email

    @rx.var
    def password_is_strong(self) -> bool:
        """Check password strength."""
        return (
            len(self.password) >= 8
            and any(c.isupper() for c in self.password)
            and any(c.isdigit() for c in self.password)
        )

    @rx.var
    def passwords_match(self) -> bool:
        """Check if passwords match."""
        return self.password == self.confirm_password

    @rx.var
    def form_is_valid(self) -> bool:
        """Check if entire form is valid."""
        return (
            self.email_is_valid
            and self.password_is_strong
            and self.passwords_match
        )

# Usage in component
rx.button(
    "Sign Up",
    on_click=SignupState.submit,
    disabled=~SignupState.form_is_valid,
)
```

## Form with File Upload

```python
class UploadState(rx.State):
    file_name: str = ""
    uploading: bool = False

    async def handle_upload(self, files: list[rx.UploadFile]):
        """Handle file upload."""
        if not files:
            return

        self.uploading = True
        file = files[0]
        self.file_name = file.filename

        try:
            # Read file content
            content = await file.read()

            # Upload to FastAPI backend
            # (implementation depends on your API)

        finally:
            self.uploading = False

def upload_form() -> rx.Component:
    return rx.vstack(
        rx.upload(
            rx.button("Select File"),
            id="file_upload",
        ),
        rx.button(
            "Upload",
            on_click=lambda: UploadState.handle_upload(
                rx.upload_files(upload_id="file_upload")
            ),
            loading=UploadState.uploading,
        ),
    )
```

## Key Takeaways

1. **State for form fields** - use setters for inputs
2. **Computed vars for validation** - real-time feedback
3. **Error messages** - clear user feedback
4. **Loading states** - disable during submission
5. **Type hints** - on all state fields and methods
6. **Validation before API call** - save backend calls
