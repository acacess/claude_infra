# Common Patterns Library

Ready-to-use regex and glob patterns for skill triggers. Copy and customize for your skills.

---

## Intent Patterns (Regex)

### Feature/Endpoint Creation
```regex
(add|create|implement|build).*?(feature|endpoint|route|service|controller)
```

### Component Creation
```regex
(create|add|make|build).*?(component|UI|page|modal|dialog|form)
```

### Database Work
```regex
(add|create|modify|update).*?(user|table|column|field|schema|migration)
(database|supabase).*?(change|update|query)
```

### Error Handling
```regex
(fix|handle|catch|debug).*?(error|exception|bug)
(add|implement).*?(try|except|error.*?handling)
```

### Explanation Requests
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

### Workflow Operations
```regex
(create|add|modify|update).*?(workflow|step|branch|condition)
(debug|troubleshoot|fix).*?workflow
```

### Testing
```regex
(write|create|add).*?(test|spec|unit.*?test)
```

---

## File Path Patterns (Glob)

### Frontend
```glob
frontend/src/**/*.pyx        # All Reflex components
frontend/src/**/*.py         # All Python files
frontend/src/components/**   # Only components directory
```

### Backend Services
```glob
form/src/**/*.py            # Form service
email/src/**/*.py           # Email service
users/src/**/*.py           # Users service
projects/src/**/*.py        # Projects service
```

### Database
```glob
**/supabase/migrations/**/*.sql      # Supabase migration files
**/supabase/**/*.sql                 # Supabase SQL files
database/src/**/*.py                  # Database scripts
```

### Workflows
```glob
form/src/workflow/**/*.py              # Workflow engine
form/src/workflow-definitions/**/*.json # Workflow definitions
```

### Test Exclusions
```glob
**/test_*.py                # Python tests (test_*.py)
**/*_test.py               # Python tests (*_test.py)
**/tests/**/*.py           # Tests directory
```

---

## Content Patterns (Regex)

### Supabase/Database
```regex
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
```

### FastAPI Routes
```regex
@app\.(get|post|put|delete|patch) # FastAPI route decorators
from fastapi import              # FastAPI imports
APIRouter                        # FastAPI router
```

### Error Handling
```regex
try\s*:                          # Try blocks
except\s*:                       # Except blocks
raise\s+                         # Raise statements
```

### Reflex/Components
```regex
import reflex as rx              # Reflex imports
rx\.                             # Reflex component usage
def\s+\w+\(\)\s*->\s*rx\.       # Reflex component functions
```

---

**Usage Example:**

```json
{
  "my-skill": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|add|build).*?(component|UI|page)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.pyx"
      ],
      "contentPatterns": [
        "import reflex as rx",
        "rx\\."
      ]
    }
  }
}
```

---

**Related Files:**
- [SKILL.md](SKILL.md) - Main skill guide
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - Detailed trigger documentation
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - Complete schema
