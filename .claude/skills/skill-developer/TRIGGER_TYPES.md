# Trigger Types - Complete Guide

Complete reference for configuring skill triggers in Claude Code's skill auto-activation system.

## Table of Contents

- [Keyword Triggers (Explicit)](#keyword-triggers-explicit)
- [Intent Pattern Triggers (Implicit)](#intent-pattern-triggers-implicit)
- [File Path Triggers](#file-path-triggers)
- [Content Pattern Triggers](#content-pattern-triggers)
- [Best Practices Summary](#best-practices-summary)

---

## Keyword Triggers (Explicit)

### How It Works

Case-insensitive substring matching in user's prompt.

### Use For

Topic-based activation where user explicitly mentions the subject.

### Configuration

```json
"promptTriggers": {
  "keywords": ["layout", "grid", "toolbar", "submission"]
}
```

### Example

- User prompt: "how does the **layout** system work?"
- Matches: "layout" keyword
- Activates: `project-catalog-developer`

### Best Practices

- Use specific, unambiguous terms
- Include common variations ("layout", "layout system", "grid layout")
- Avoid overly generic words ("system", "work", "create")
- Test with real prompts

---

## Intent Pattern Triggers (Implicit)

### How It Works

Regex pattern matching to detect user's intent even when they don't mention the topic explicitly.

### Use For

Action-based activation where user describes what they want to do rather than the specific topic.

### Configuration

```json
"promptTriggers": {
  "intentPatterns": [
    "(create|add|implement).*?(feature|endpoint)",
    "(how does|explain).*?(layout|workflow)"
  ]
}
```

### Examples

**Database Work:**
- User prompt: "add user tracking feature"
- Matches: `(add).*?(feature)`
- Activates: `database-verification`, `error-tracking`

**Component Creation:**
- User prompt: "create a dashboard widget"
- Matches: `(create).*?(component)` (if component in pattern)
- Activates: `frontend-dev-guidelines`

### Best Practices

- Capture common action verbs: `(create|add|modify|build|implement)`
- Include domain-specific nouns: `(feature|endpoint|component|workflow)`
- Use non-greedy matching: `.*?` instead of `.*`
- Test patterns thoroughly with regex tester (https://regex101.com/)
- Don't make patterns too broad (causes false positives)
- Don't make patterns too specific (causes false negatives)

### Common Pattern Examples

```regex
# Database Work
(add|create|implement).*?(user|login|auth|feature)

# Explanations
(how does|explain|what is|describe).*?

# Frontend Work
(create|add|make|build).*?(component|UI|page|modal|dialog)

# Error Handling
(fix|handle|catch|debug).*?(error|exception|bug)

# Workflow Operations
(create|add|modify).*?(workflow|step|branch|condition)
```

---

## File Path Triggers

### How It Works

Glob pattern matching against the file path being edited.

### Use For

Domain/area-specific activation based on file location in the project.

### Configuration

```json
"fileTriggers": {
  "pathPatterns": [
    "app/features/**/*.py",
    "app/routers/**/*.py"
  ],
  "pathExclusions": [
    "**/tests/**/*.py",
    "**/test_*.py",
    "**/*_test.py"
  ]
}
```

### Glob Pattern Syntax

- `**` = Any number of directories (including zero)
- `*` = Any characters within a directory name
- Examples:
  - `app/features/**/*.py` = All Reflex feature modules
  - `**/supabase/migrations/**/*.sql` = Supabase migration files anywhere in project
  - `app/routers/**/*.py` = All FastAPI router modules

### Example

- File being edited: `app/features/dashboard/components/card.py`
- Matches: `app/features/**/*.py`
- Activates: `frontend-dev-guidelines`

### Best Practices

- Be specific to avoid false positives
- Use exclusions for test files: `**/test_*.py` or `**/*_test.py`
- Consider subdirectory structure
- Test patterns with actual file paths
- Use narrower patterns when possible: `app/services/**` not `app/**`

### Common Path Patterns

```glob
# Reflex Frontend
app/pages/**/*.py             # Page modules decorated with @rx.page
app/features/**/*.py          # Feature state/components
app/components/**/*.py        # Shared components

# FastAPI Backend
app/routers/**/*.py           # APIRouter definitions
app/services/**/*.py          # Business logic layer
app/repositories/**/*.py      # Database access layer

# Supabase / Database
supabase/migrations/**/*.sql  # SQL migrations
alembic/versions/**/*.py      # Alembic migrations
app/db/**/*.py                # Database utilities

# Workflow / Utilities
app/workflows/**/*.py
scripts/**/*.py

# Test Exclusions
**/tests/**/*.py
**/test_*.py
**/*_test.py
```

---

## Content Pattern Triggers

### How It Works

Regex pattern matching against the file's actual content (what's inside the file).

### Use For

Technology-specific activation based on what the code imports or uses (Supabase, FastAPI routes, specific libraries).

### Configuration

```json
"fileTriggers": {
  "contentPatterns": [
    "from supabase import",
    "import supabase",
    "create_client",
    "\\.table\\(",
    "\\.select\\("
  ]
}
```

### Examples

**Supabase Detection:**
- File contains: `from supabase import create_client`
- Matches: `from supabase import`
- Activates: `database-verification`

**FastAPI Route Detection:**
- File contains: `@app.get("/users")` or `@app.post("/users")`
- Matches: `@app\\.(get|post|put|delete|patch)`
- Activates: `error-tracking`

### Best Practices

- Match imports: `from supabase import` or `import supabase`
- Escape special regex chars: `\\.table\\(` not `.table(`
- Patterns use case-insensitive flag
- Test against real file content
- Make patterns specific enough to avoid false matches

### Common Content Patterns

```regex
# Supabase/Database
from supabase import             # Supabase imports
import supabase                  # Supabase imports (alternative)
create_client                    # Supabase client creation
supabase\.                       # supabase.something
\.table\(                        # Supabase query methods
\.select\(
\.insert\(
\.update\(
\.delete\(
\.from_\(

# FastAPI Routes
@app\.(get|post|put|delete|patch) # FastAPI route decorators
from fastapi import              # FastAPI imports
APIRouter                        # FastAPI router

# Error Handling
try\s*:                          # Try blocks
except\s*:                       # Except blocks
raise\s+                         # Raise statements

# Reflex/Components
import reflex as rx              # Reflex imports
rx\.                             # Reflex component usage
def\s+\w+\(\)\s*->\s*rx\.       # Reflex component functions
```

---

## Best Practices Summary

### DO:
✅ Use specific, unambiguous keywords
✅ Test all patterns with real examples
✅ Include common variations
✅ Use non-greedy regex: `.*?`
✅ Escape special characters in content patterns
✅ Add exclusions for test files
✅ Make file path patterns narrow and specific

### DON'T:
❌ Use overly generic keywords ("system", "work")
❌ Make intent patterns too broad (false positives)
❌ Make patterns too specific (false negatives)
❌ Forget to test with regex tester (https://regex101.com/)
❌ Use greedy regex: `.*` instead of `.*?`
❌ Match too broadly in file paths

### Testing Your Triggers

**Test keyword/intent triggers:**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  python .claude/hooks/skill-activation-prompt.py
```

**Test file path/content triggers:**
```bash
cat <<'EOF' | python .claude/hooks/skill-verification-guard.py
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/test/file.py"}
}
EOF
```

---

**Related Files:**
- [SKILL.md](SKILL.md) - Main skill guide
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - Complete skill-rules.json schema
- [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md) - Ready-to-use pattern library
