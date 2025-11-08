# CLAUDE.md Template

```markdown
# [Your Project Name] - Claude Code Guidelines

This document contains project-specific guidelines and best practices for Claude Code when working on this codebase.

## Rules Summary

### 1. Meta - Maintaining This Document
See all rules in section **## META - MAINTAINING THIS DOCUMENT**
1. When adding new rules to detailed sections below, always update this summary section with a corresponding one-sentence summary
2. Each rule in this summary must reference its corresponding detailed section
3. Follow the writing guidelines when adding new rules

### 2. Code Organization
See all rules in section **## CODE ORGANIZATION**
1. Always place imports at the top of the file, never inside functions
2. [Add your rules here]

### 3. Testing
See all rules in section **## TESTING**
1. Always use `patch.object` rather than `patch` for mocking
2. Never use magic numbers or string literals in tests - extract constants
3. Always check for existing test helpers before creating new ones
4. [Add your rules here]

---

## META - MAINTAINING THIS DOCUMENT

### Keeping the Summary Section Up to Date

**Rule**: Whenever you add, modify, or remove rules in the detailed sections below, you MUST update the "Rules Summary" section at the top of this document.

**Process**:
1. Add the new rule to the appropriate detailed section below
2. Add a corresponding one-sentence summary to the Rules Summary section
3. Ensure the summary references the detailed section using the format: "See all rules in section **## SECTION NAME**"
4. If creating a new topic, add both a new numbered topic in the summary AND a new detailed section below

**Example**:
If you add a new rule about async patterns in the detailed "ASYNC PATTERNS" section, you must add:
- A new topic in Rules Summary: "4. Async Patterns - See all rules in section **## ASYNC PATTERNS**"
- A one-sentence summary under that topic

### Writing Effective Guidelines

When adding new rules to this document, follow these principles:

**Core Principles (Always Apply):**
1. **Use absolute directives** - Start with "NEVER" or "ALWAYS" for non-negotiable rules
2. **Lead with why** - Explain the problem/rationale before showing the solution (1-3 bullets max)
3. **Be concrete** - Include actual commands/code for project-specific patterns
4. **Minimize examples** - One clear point per code block
5. **Bullets over paragraphs** - Keep explanations concise
6. **Action before theory** - Put immediate takeaways first

**Optional Enhancements (Use Strategically):**
- **❌/✅ examples**: Only when the antipattern is subtle or common
- **"Why" or "Rationale" section**: Keep to 1-3 bullets explaining the underlying reason
- **"Warning Signs" section**: Only for gradual/easy-to-miss violations
- **"General Principle"**: Only when the abstraction is non-obvious  
- **Decision trees**: Only for 3+ factor decisions with multiple considerations

**Anti-Bloat Rules:**
- ❌ Don't add "Warning Signs" to obvious rules (e.g., "imports at top")
- ❌ Don't show bad examples for trivial mistakes
- ❌ Don't create decision trees for simple binary choices
- ❌ Don't add "General Principle" when the section title already generalizes
- ❌ Don't write paragraphs explaining what bullets can convey
- ❌ Don't write long "Why" explanations - 1-3 bullets maximum

---

## CODE ORGANIZATION

### Import Placement

**Always place imports at the top of the file, never inside functions.**

```python
# ✅ CORRECT
from datetime import datetime
import pytest

def test_something():
    # test code here
    pass

# ❌ INCORRECT
def test_something():
    from datetime import datetime  # Never inside functions
    pass
```

**Rationale**: Top-level imports make dependencies clear, allow static analysis tools to work correctly, and avoid repeated import overhead.

---

## TESTING

### Always Use patch.object

**Always use `patch.object` instead of `patch` for mocking.**

```python
# ✅ CORRECT
from unittest.mock import patch

@patch.object(MyClass, 'method_name')
def test_with_mock(mock_method):
    # test code
    pass

# ❌ INCORRECT
@patch('module.path.MyClass.method_name')  # Never use patch with string paths
def test_with_mock(mock_method):
    pass
```

**Rationale**: `patch.object` is more explicit, refactor-safe, and catches errors at test definition time rather than runtime.

### Never Use Magic Numbers or String Literals

**FORBIDDEN: Using literal values scattered throughout tests. Always extract constants and derive values from authoritative sources.**

```python
# ❌ INCORRECT - Magic numbers
def test_timeout():
    assert response.elapsed < 60  # What is 60?
    context = make_context(interval=30)  # What is 30?

# ✅ CORRECT - Named constants
DEFAULT_TIMEOUT_SECONDS = 60
DEFAULT_INTERVAL_SECONDS = 30

def test_timeout():
    assert response.elapsed < DEFAULT_TIMEOUT_SECONDS
    context = make_context(interval=DEFAULT_INTERVAL_SECONDS)
```

**Why It's Wrong**:
- Tests become brittle when constants change
- Unclear where values come from
- Obscures relationships between values

**Rules**:
1. Import constants from source code when possible
2. Define test-specific constants at module level
3. Calculate derived values explicitly to show relationships

### Check for Existing Test Helpers

**Before creating a new test helper or fixture, ALWAYS search for existing ones.**

```bash
# Search for similar helpers:
grep -r "def make_test" tests/ --include="*.py"
grep -r "def create_test" tests/ --include="*.py"
```

**Decision Tree**:
1. Helper exists and fits your needs? → Use it
2. Helper exists but needs extension? → Extend it with optional parameters
3. Helper doesn't exist? → Create it in the appropriate location

**Rationale**: Prevents code duplication, maintains consistency, and makes tests easier to maintain.

---

## [YOUR CUSTOM SECTIONS]

Add your project-specific sections here following the same pattern:
- Section header with ##
- Individual rules with ###
- Each rule should have ALWAYS/NEVER directive
- Include why/rationale
- Show ❌/✅ examples when the antipattern is subtle
- Reference this section in the Rules Summary at the top
```

**How to Use This Template:**

1. Copy this file to your repository as `CLAUDE.md`
2. Replace `[Your Project Name]` with your actual project name
3. Keep the META section intact - this is what makes the system work
4. Add your project-specific rules to the existing sections
5. Create new sections for topics specific to your codebase
6. When Claude makes a mistake, use the magic prompt: **"Reflect on this mistake. Abstract and generalize the learning. Write it to CLAUDE.md"**
7. Let the document grow organically through use

The template includes a few universally useful rules to get you started, but the real power comes from letting Claude add rules as it makes mistakes in your specific codebase.
