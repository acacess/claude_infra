# Styling Guide - Reflex Component Styling

Complete guide to styling Reflex components.

## Styling Approaches

### Inline Styles

```python
import reflex as rx

def styled_button() -> rx.Component:
    return rx.button(
        "Click Me",
        background="blue.500",
        color="white",
        padding="4",
        border_radius="md",
        _hover={
            "background": "blue.600",
        },
        _active={
            "transform": "scale(0.95)",
        },
    )
```

### Style Dictionaries

```python
# Reusable style dictionary
card_style = {
    "padding": "6",
    "border_radius": "lg",
    "box_shadow": "lg",
    "background": "white",
    "_hover": {
        "box_shadow": "xl",
    },
}

def info_card(title: str, content: str) -> rx.Component:
    return rx.box(
        rx.heading(title, size="5"),
        rx.text(content),
        **card_style
    )
```

### Responsive Styles

```python
def responsive_box() -> rx.Component:
    return rx.box(
        rx.text("Responsive content"),
        width=["100%", "100%", "50%", "33%"],  # mobile, tablet, desktop, large
        padding=["2", "4", "6", "8"],
    )
```

## Color System

```python
# Use Chakra UI color palette
rx.box(background="blue.500")    # Primary blue
rx.box(background="red.500")     # Red
rx.box(background="gray.100")    # Light gray
```

## Key Takeaways

1. **Inline styles** for simple components
2. **Style dictionaries** for reusable styles
3. **Responsive arrays** for different screen sizes
4. **Chakra UI colors** for consistency
5. **Pseudo-selectors** (_hover, _active, _focus)
