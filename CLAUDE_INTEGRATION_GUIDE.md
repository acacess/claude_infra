# Claude Integration Guide

**FOR CLAUDE CODE:** When a user asks you to integrate components from this showcase repository into their project, follow these instructions carefully.

---

## Overview

This repository is a **reference library** of Claude Code infrastructure components. Users will ask you to help integrate specific pieces into their projects. Your role is to:

1. **Ask clarifying questions** about their project structure
2. **Copy the appropriate files**
3. **Customize configurations** for their setup
4. **Verify the integration** works correctly

**Key Principle:** ALWAYS ask before assuming project structure. What works for one project won't work for another.

---

## Tech Stack Compatibility Check

**CRITICAL:** Before integrating a skill, verify the user's tech stack matches the skill requirements.

### Available Skills by Tech Stack

This repository contains skills for **two different tech stacks**:

#### Python Stack (FastAPI + Reflex)

**fastapi-backend-guidelines:**
- Python 3.12+
- FastAPI
- PostgreSQL
- SQLAlchemy ORM (async)
- Pydantic validation
- Sentry

**reflex-frontend-guidelines:**
- Reflex (Python web framework)
- Python 3.12+
- State-based architecture

#### Node.js Stack (Express + React)

**backend-dev-guidelines:**
- Node.js
- Express
- TypeScript
- Prisma ORM
- Zod validation
- Sentry

**frontend-dev-guidelines:**
- React
- TypeScript
- MUI v7
- TanStack Query/Router

### How to Choose the Right Skills

**Before integrating, ask:**
1. "What's your backend stack: Python/FastAPI or Node.js/Express?"
2. "What's your frontend stack: Reflex or React/TypeScript?"

**Then integrate the matching skills:**

| User's Stack | Use These Skills |
|--------------|------------------|
| Python + FastAPI + Reflex | `fastapi-backend-guidelines` + `reflex-frontend-guidelines` |
| Node.js + Express + React | `backend-dev-guidelines` + `frontend-dev-guidelines` |
| Mixed or different | Offer to adapt using skills as templates |

**If user has different stack:**
```
I see you're using [their stack]. This repository has skills for:
- Python (FastAPI + Reflex)
- Node.js (Express + React)

I can:
1. Adapt one of these skills as a template for your stack
2. Extract framework-agnostic patterns (layered architecture, etc.)
3. Skip the stack-specific skills

Which would you prefer?
```

### Skills That Are Tech-Agnostic

These work for ANY tech stack:
- ✅ **skill-developer** - Meta-skill, no tech requirements
- ✅ **route-tester** - Only requires JWT cookie auth (framework agnostic)
- ✅ **error-tracking** - Sentry works with most stacks

---

## General Integration Pattern

When user says: **"Add [component] to my project"**

1. Identify component type (skill/hook/agent/command)
2. **CHECK TECH STACK COMPATIBILITY** (for frontend/backend skills)
3. Ask about their project structure
4. Copy files OR adapt for their stack
5. Customize for their setup
6. Verify integration
7. Provide next steps

---

## Integrating Skills

### Step-by-Step Process

**When user requests a skill** (e.g., "add backend-dev-guidelines"):

#### 1. Understand Their Project

**ASK THESE QUESTIONS:**
- "What's your project structure? Single app, monorepo, or multi-service?"
- "Where is your [backend/frontend] code located?"
- "What frameworks/technologies do you use?"

#### 2. Copy the Skill

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \\
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. Handle skill-rules.json

**Check if it exists:**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**If NO (doesn't exist):**
- Copy the template from showcase
- Remove skills user doesn't want
- Customize for their project

**If YES (exists):**
- Read their current skill-rules.json
- Add the new skill entry
- Merge carefully to avoid breaking existing skills

#### 4. Customize Path Patterns

**CRITICAL:** Update `pathPatterns` in skill-rules.json to match THEIR structure:

**Example - User has monorepo:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "backend/**/*.py",
        "api/**/*.py",
        "services/*/app/**/*.py",
        "packages/*/app/**/*.py"
      ]
    }
  }
}
```

**Example - User has single backend:**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "app/**/*.py",
        "src/**/*.py",
        "backend/**/*.py"
      ]
    }
  }
}
```

**Safe Generic Patterns** (when unsure):
```json
{
  "pathPatterns": [
    "**/*.py",          // All Python files
    "app/**/*.py",      // Common app directory (FastAPI convention)
    "src/**/*.py",      // Common src directory
    "backend/**/*.py"   // Common backend directory
  ]
}
```

#### 5. Verify Integration

```bash
# Check skill was copied
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# Validate skill-rules.json syntax
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**Tell user:** "Try editing a file in [their-backend-path] and the skill should activate."

---

### Skill-Specific Notes

#### fastapi-backend-guidelines (Python/FastAPI)
- **Tech Requirements:** Python 3.12+, FastAPI, Supabase (PostgreSQL), SQLAlchemy (async), Sentry
- **Ask:** "Do you use FastAPI with Supabase?" "Where's your backend code?"
- **If different stack:** Offer to adapt using this as template
- **Customize:** pathPatterns
- **Example paths:** `backend/`, `api/`, `app/`, `services/*/app/`
- **Adaptation tip:** Architecture patterns (Routers→Dependencies→Services→Repositories) transfer to most Python frameworks
- **Note:** This stack uses **Supabase** as the database platform (PostgreSQL with Supabase features)

#### backend-dev-guidelines (Node.js/Express)
- **Tech Requirements:** Node.js, Express, TypeScript, Prisma ORM, Sentry
- **Ask:** "Do you use Express with Prisma?" "Where's your backend code?"
- **If different stack:** Offer to adapt using this as template
- **Customize:** pathPatterns
- **Example paths:** `backend/`, `api/`, `services/*/src/`, `packages/*/src/`
- **Adaptation tip:** Architecture patterns (Routes→Controllers→Services→Repositories) transfer to most Node.js frameworks

#### reflex-frontend-guidelines (Python/Reflex)
- **Tech Requirements:** Reflex, Python 3.12+
- **Ask:** "Do you use Reflex?" "Where's your frontend code?"
- **If different stack:** Offer to create adapted version
- **Customize:** pathPatterns + all framework-specific examples
- **Example paths:** `frontend/`, `web/`, `app/`, `reflex/`
- **Adaptation tip:** State management and file organization patterns may transfer to similar Python web frameworks
- **Note:** This frontend is designed to work with **FastAPI + Supabase backend** (see `fastapi-backend-guidelines`). Tech stack: **FastAPI + Supabase + Reflex**.

#### frontend-dev-guidelines (React/TypeScript)
- **Tech Requirements:** React, TypeScript, MUI v7, TanStack Query/Router
- **Ask:** "Do you use React with TypeScript?" "Where's your frontend code?"
- **If different stack:** Offer to create adapted version (Vue, Angular, etc.)
- **Customize:** pathPatterns + framework-specific examples
- **Example paths:** `frontend/`, `web/`, `apps/web/`, `src/`
- **Adaptation tip:** Component patterns and file organization transfer across React-like frameworks, but library APIs don't

#### route-tester
- **Tech Requirements:** JWT cookie-based authentication (framework agnostic)
- **Ask:** "Do you use JWT cookie-based authentication?"
- **If NO:** "This skill is designed for JWT cookies. Want me to adapt it for [their auth type] or skip it?"
- **Customize:** Service URLs, auth patterns
- **Works with:** Any backend framework using JWT cookies

#### error-tracking
- **Tech Requirements:** Sentry (works with most backends)
- **Ask:** "Do you use Sentry?" "Where's your backend code?"
- **If NO Sentry:** "Want to use this as template for [their error tracking]?"
- **Customize:** pathPatterns
- **Adaptation tip:** Error tracking philosophy transfers to other tools (Rollbar, Bugsnag, etc.)

#### skill-developer
- **Tech Requirements:** None!
- **Copy as-is** - meta-skill, fully generic, teaches skill creation for ANY tech stack

---

## Adapting Skills for Different Tech Stacks

When user's tech stack differs from skill requirements, you have options:

### Option 1: Adapt Existing Skill (Recommended)

**When to use:** User wants similar guidelines but for different tech

**Process:**
1. **Copy the skill as a starting point:**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \\
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **Identify what needs changing:**
   - Framework-specific code examples (Reflex → other framework)
   - Library APIs (Reflex UI components → other component library)
   - Import statements
   - Component patterns

3. **Keep what transfers:**
   - File organization principles
   - Performance optimization strategies
   - Python type hints standards
   - General best practices

4. **Replace examples systematically:**
   - Ask user for equivalent patterns in their stack
   - Update code examples to their framework
   - Keep the overall structure and sections

5. **Update skill name and triggers:**
   - Rename skill appropriately
   - Update skill-rules.json triggers for their stack
   - Test activation

**Example - Adapting frontend-dev-guidelines for other frameworks:**
```
I'll create [framework]-dev-guidelines based on the Reflex skill structure:
- Replace Reflex components → [their framework] components
- Replace Reflex state classes → [their state management]
- Replace Reflex UI components (rx.*) → [their component library]
- Keep: File organization, performance patterns, Python type hints

This will take a few minutes. Sound good?
```

### Option 2: Extract Framework-Agnostic Patterns

**When to use:** Stacks are very different, but core principles apply

**Process:**
1. Read through the existing skill
2. Identify framework-agnostic patterns:
   - Layered architecture (backend)
   - File organization strategies
   - Performance optimization principles
   - Testing strategies
   - Error handling philosophy

3. Create new skill with just those patterns
4. User can add framework-specific examples later

**Example:**
```
The backend-dev-guidelines uses FastAPI, but the layered architecture
(Routers → Dependencies → Services → Repositories) works for Django too.

I can create a skill with:
- Layered architecture pattern
- Separation of concerns principles
- Error handling best practices
- Testing strategies

Then you can add Django-specific examples as you establish patterns.
```

### Option 3: Use as Reference Only

**When to use:** Too different to adapt, but user wants inspiration

**Process:**
1. User browses the existing skill
2. You help create a new skill from scratch
3. Use existing skill's structure as a template
4. Follow modular pattern (main + resource files)

### What Usually Transfers Across Tech Stacks

**Architecture & Organization:**
- ✅ Layered architecture (Routes/Controllers/Services pattern)
- ✅ Separation of concerns
- ✅ File organization strategies (features/ pattern)
- ✅ Progressive disclosure (main + resource files)
- ✅ Repository pattern for data access

**Development Practices:**
- ✅ Error handling philosophy
- ✅ Input validation importance
- ✅ Testing strategies
- ✅ Performance optimization principles
- ✅ Python type hints best practices

**Framework-Specific Code:**
- ❌ Reflex state classes → Don't transfer to other frameworks
- ❌ Reflex UI components (rx.*) → Different component libraries
- ❌ SQLAlchemy queries → Different ORM syntax
- ❌ FastAPI dependencies → Different framework patterns
- ❌ Routing implementations → Framework-specific

### When to Recommend Adaptation vs Skipping

**Recommend adaptation if:**
- User wants similar guidelines for their stack
- Core patterns apply (layered architecture, etc.)
- User has time to help with framework-specific examples

**Recommend skipping if:**
- Stacks are completely different
- User doesn't need those patterns
- Would take too long to adapt
- User prefers creating from scratch

---

## Integrating Hooks

### Essential Hooks (Always Safe to Copy)

#### skill-activation-prompt (UserPromptSubmit)

**Purpose:** Auto-suggests skills based on user prompts

**Integration (NO customization needed):**

```bash
# Copy both files
cp showcase/.claude/hooks/skill-activation-prompt.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/
# If Python hook exists, copy it
if [ -f "showcase/.claude/hooks/skill-activation-prompt.py" ]; then
  cp showcase/.claude/hooks/skill-activation-prompt.py \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
fi

# Make executable
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# Install dependencies if needed
if [ -f "showcase/.claude/hooks/requirements.txt" ]; then
  cp showcase/.claude/hooks/requirements.txt \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && pip install -r requirements.txt
elif [ -f "showcase/.claude/hooks/pyproject.toml" ]; then
  cp showcase/.claude/hooks/pyproject.toml \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && pip install -e .
fi
```

**Add to settings.json:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**This hook is FULLY GENERIC** - works anywhere, no customization needed!

#### post-tool-use-tracker (PostToolUse)

**Purpose:** Tracks file changes for context management

**Integration (NO customization needed):**

```bash
# Copy file
cp showcase/.claude/hooks/post-tool-use-tracker.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# Make executable
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**Add to settings.json:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**This hook is FULLY GENERIC** - auto-detects project structure!

---

### Optional Hooks (Require Heavy Customization)

#### type-check.sh and trigger-build-resolver.sh (Stop hooks)

⚠️ **WARNING:** These hooks are configured for a specific multi-service monorepo structure.

**Before integrating, ask:**
1. "Do you have a monorepo with multiple Python services?"
2. "What are your service directory names?"
3. "Where are your pyproject.toml or setup.py files located?"
4. "Do you use mypy or pyright for type checking?"

**For SIMPLE projects (single service):**
- **RECOMMEND SKIPPING** these hooks
- They're overkill for single-service projects
- User can run `mypy .` or `pyright .` manually instead

**For COMPLEX projects (multi-service monorepo):**

1. Copy the files
2. **MUST EDIT** type-check.sh - find this section:
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. Replace with USER'S actual service names:
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← User's services
        echo "$repo"
        return 0
        ;;
esac
```

4. Update type checking command (mypy or pyright):
```bash
# For mypy
mypy $SERVICE_DIR

# For pyright
pyright $SERVICE_DIR
```

5. Test manually before adding to settings.json:
```bash
./.claude/hooks/type-check.sh
```

**IMPORTANT:** If this hook fails, it will block Stop events. Only add if you're sure it works for their setup.

---

## Integrating Agents

**Agents are STANDALONE** - easiest to integrate!

### Standard Agent Integration

```bash
# Copy the agent file
cp showcase/.claude/agents/[agent-name].md \\
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**That's it!** Agents work immediately, no configuration needed.

### Check for Hardcoded Paths

Some agents may reference paths. **Before copying, read the agent file and check for:**

- `~/git/old-project/` → Should be `$CLAUDE_PROJECT_DIR` or `.`
- `/root/git/project/` → Should be `$CLAUDE_PROJECT_DIR` or `.`
- Hardcoded screenshot paths → Ask user where they want screenshots

**If found, update them:**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \\
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### Agent-Specific Notes

**auth-route-tester / auth-route-debugger:**
- Requires JWT cookie-based authentication in user's project
- Ask: "Do you use JWT cookies for auth?"
- If NO: "These agents are for JWT cookie auth. Skip them or want me to adapt?"

**frontend-error-fixer:**
- May reference screenshot paths
- Ask: "Where should screenshots be saved?"

**All other agents:**
- Copy as-is, they're fully generic

---

## Integrating Slash Commands

```bash
# Copy command file
cp showcase/.claude/commands/[command].md \\
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### Customize Paths

Commands may reference dev docs paths. **Check and update:**

**dev-docs and dev-docs-update:**
- Look for `dev/active/` path references
- Ask: "Where do you want dev documentation stored?"
- Update paths in the command files

**route-research-for-testing:**
- May reference service paths
- Ask about their API structure

---

## Common Patterns & Best Practices

### Pattern: Asking About Project Structure

**DON'T assume:**
- ❌ "I'll add this for your blog-api service"
- ❌ "Configuring for your frontend directory"

**DO ask:**
- ✅ "What's your project structure? Monorepo or single app?"
- ✅ "Where is your backend code located?"
- ✅ "Do you use workspaces or have multiple services?"

### Pattern: Customizing skill-rules.json

**User has monorepo with workspaces:**
```json
{
  "pathPatterns": [
    "backend/**/*.py",
    "frontend/**/*.py",
    "packages/*/app/**/*.py",
    "services/*/app/**/*.py"
  ]
}
```

**User has monorepo with separate backend/frontend:**
```json
{
  "pathPatterns": [
    "backend/app/**/*.py",
    "frontend/**/*.py",
    "api/app/**/*.py"
  ]
}
```

**User has simple structure:**
```json
{
  "pathPatterns": [
    "app/**/*.py",
    "src/**/*.py",
    "backend/**/*.py"
  ]
}
```

### Pattern: settings.json Integration

**NEVER copy the showcase settings.json directly!**

Instead, **extract and merge** the sections they need:

1. Read their existing settings.json
2. Add the hook configurations they want
3. Preserve their existing config

**Example merge:**
```json
{
  // ... their existing config ...
  "hooks": {
    // ... their existing hooks ...
    "UserPromptSubmit": [  // ← Add this section
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Verification Checklist

After integration, **verify these items:**

```bash
# 1. Hooks are executable
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# Should show: -rwxr-xr-x

# 2. skill-rules.json is valid JSON
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# Should parse without errors

# 3. Hook dependencies installed (if Python hooks)
# Check for virtual environment or installed packages
ls $CLAUDE_PROJECT_DIR/.claude/hooks/.venv/ 2>/dev/null || \
  python3 -c "import sys; print(sys.path)" | grep -q hooks
# Should show virtual environment or packages in Python path

# 4. Settings.json is valid JSON
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# Should parse without errors
```

**Then ask user to test:**
- "Try editing a file in [relevant-path] - the skill should activate"
- "Try asking me a question about [topic] - I should suggest the skill"

---

## Common Mistakes to Avoid

### ❌ DON'T: Copy settings.json as-is
**Why:** The Stop hooks reference non-existent services
**DO:** Extract only UserPromptSubmit and PostToolUse sections

### ❌ DON'T: Keep example service names
**Why:** User doesn't have blog-api, auth-service, etc.
**DO:** Ask about their actual structure and update

### ❌ DON'T: Skip making hooks executable
**Why:** Hooks won't run without execute permission
**DO:** Always `chmod +x` after copying

### ❌ DON'T: Assume monorepo structure
**Why:** Most projects are single-service
**DO:** Ask first, then customize

### ❌ DON'T: Add all skills at once
**Why:** Overwhelming and may not all be relevant
**DO:** Ask which skills are relevant to their work

### ❌ DON'T: Copy Stop hooks without testing
**Why:** They can block the Stop event if they fail
**DO:** Test manually first, only add if working

---

## Example Integration Conversations

### Example 1: Adding Backend Skill

**User:** "Add the backend-dev-guidelines skill to my project"

**Claude should respond:**
```
I'll add the backend-dev-guidelines skill. First, I need to understand
your project structure:

1. Is this a monorepo with multiple services, or a single backend application?
2. Where is your backend code located? (e.g., app/, backend/, api/)
3. Do you use FastAPI with SQLAlchemy?
```

**User:** "Monorepo, code is in backend/app/, yes we use FastAPI with SQLAlchemy"

**Claude should:**
1. Copy the skill directory
2. Update skill-rules.json with pathPattern: `"backend/app/**/*.py"`
3. Verify it's copied correctly
4. Respond:
```
✅ Added backend-dev-guidelines skill!

Configuration:
- Skill triggers when editing files in backend/app/
- Activated by keywords like "router", "service", "dependency", "API"

Try it: Edit a file in backend/app/ and you should see the skill suggestion.
```

### Example 2: Setting Up Hooks

**User:** "Set up the skill activation hooks"

**Claude should respond:**
```
I'll set up the skill auto-activation system. This requires two hooks:
1. skill-activation-prompt - Suggests skills based on your prompts
2. post-tool-use-tracker - Tracks file changes

Do you have a .claude/settings.json file already, or should I create one?
```

**User:** "I have settings.json already"

**Claude should:**
1. Read their settings.json
2. Copy both hook files
3. Install dependencies (pip install -r requirements.txt or pip install -e .) in hooks directory
4. Add hook configurations to their settings.json (preserving existing config)
5. Make hooks executable
6. Respond:
```
✅ Skill activation hooks installed!

What I did:
- Copied skill-activation-prompt hook
- Copied post-tool-use-tracker hook
- Installed dependencies
- Added to your settings.json

Test it: Ask me "help with backend development" and I should suggest
the backend-dev-guidelines skill.
```

### Example 3: Adding Agent

**User:** "Add the code-architecture-reviewer agent"

**Claude should:**
```
The code-architecture-reviewer agent reviews code for architectural issues
and best practices. This agent works standalone, no configuration needed.

I'll copy it to your .claude/agents/ directory now.
```

**Then:**
1. Copy the agent file
2. Respond:
```
✅ Added code-architecture-reviewer agent!

Use it by running the Task tool with:
- subagent_type: "code-architecture-reviewer"
- prompt: Description of code to review

The agent will review your code and provide architectural feedback.
```

### Example 3: Tech Stack Mismatch

**User:** "Add the frontend-dev-guidelines skill to my Streamlit project"

**Claude should:**
```
I see you're using Streamlit. The frontend-dev-guidelines skill is specifically
designed for Reflex. I have a few options:

Option 1 (Recommended): I can create a streamlit-dev-guidelines skill using the
Reflex skill as a template. I'll adapt:
- Component patterns for Streamlit (st.* components)
- Replace Reflex state classes with Streamlit session state
- Adapt UI patterns for Streamlit
- Keep: File organization, performance principles, Python type hints

Option 2: Extract only the framework-agnostic patterns (file organization,
performance, testing) without Reflex-specific code.

Option 3: Skip it and help you create Streamlit guidelines from scratch.

Which would you prefer?
```

**User:** "Option 1 sounds good"

**Claude should:**
1. Copy frontend-dev-guidelines as streamlit-dev-guidelines
2. Go through each resource file
3. Replace Reflex patterns with Streamlit components
4. Replace Reflex state classes with Streamlit session state
5. Update skill-rules.json with Streamlit-specific triggers (.py files in frontend/)
6. Respond:
```
✅ Created streamlit-dev-guidelines skill adapted for Streamlit!

What I changed:
- Reflex components → Streamlit st.* components
- Reflex state classes → Streamlit session state
- Reflex UI components (rx.*) → Streamlit UI components
- File patterns → .py files in frontend/

What I kept:
- File organization (features/ pattern)
- Performance optimization strategies
- Python type hints standards
- Loading/error state principles

Try editing a .py file in frontend/ - the skill should activate.
```

---

## Quick Reference Tables

### What Needs Customization?

| Component | Tech Requirements | Customization | What to Ask |
|-----------|------------------|--------------|-------------|
| **skill-developer** | None | ✅ None | Copy as-is |
| **backend-dev-guidelines** | FastAPI/SQLAlchemy/Python | ⚠️ Paths + tech check | "Use FastAPI/SQLAlchemy?" "Where's backend?" |
| **frontend-dev-guidelines** | Reflex | ⚠️⚠️ Paths + framework | "Use Reflex?" "Where's frontend?" |
| **route-tester** | JWT cookies | ⚠️ Auth + paths | "JWT cookie auth?" |
| **error-tracking** | Sentry | ⚠️ Paths | "Use Sentry?" "Where's backend?" |
| **skill-activation-prompt** | ✅ None | Copy as-is |
| **post-tool-use-tracker** | ✅ None | Copy as-is |
| **type-check** | ⚠️⚠️⚠️ Heavy | "Monorepo or single service?" "mypy or pyright?" |
| **All agents** | ✅ Minimal | Check paths |
| **All commands** | ⚠️ Paths | "Where for dev docs?" |

### When to Recommend Skipping

| Component | Skip If... |
|-----------|-----------|
| **type-check hooks** | Single-service project or different type checking setup |
| **route-tester** | Not using JWT cookie authentication |
| **frontend-dev-guidelines** | Not using Reflex |
| **auth agents** | Not using JWT cookie auth |

---

## Final Tips for Claude

**When user says "add everything":**
- Start with essentials: skill-activation hooks + 1-2 relevant skills
- Don't overwhelm them with all 5 skills + 10 agents
- Ask what they actually need

**When something doesn't work:**
- Check verification checklist
- Verify paths match their structure
- Test hooks manually
- Check for JSON syntax errors

**When user is unsure:**
- Recommend starting with just skill-activation hooks
- Add backend OR frontend skill (whichever they use)
- Add more later as needed

**Always explain what you're doing:**
- Show the commands you're running
- Explain why you're asking questions
- Provide clear next steps after integration

---

**Remember:** This is a reference library, not a working application. Your job is to help users cherry-pick and adapt components for THEIR specific project structure.
