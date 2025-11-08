# Gigaverse AI - Claude Code Guidelines

This document contains project-specific guidelines and best practices for Claude Code when working on this codebase.

## Rules Summary

### 1. Meta - Maintaining This Document
See all rules in section **## META - MAINTAINING THIS DOCUMENT**
1. When adding new rules to detailed sections below, always update this summary section with a corresponding one-sentence summary
2. Each rule in this summary must reference its corresponding detailed section

### 2. Type Checking - Basedpyright
See all rules in section **## TYPE CHECKING WITH BASEDPYRIGHT**
1. Type checking should NOT change runtime behavior - keep original logic intact
2. Always run tests after adding type hints
3. Use `Field(default=...)` keyword syntax for Pydantic field defaults
4. Use `Optional[Type]` for parameters with `None` defaults
5. Use `# type: ignore` only for legitimate cases (function attributes, dynamic attributes, legacy patterns)
6. Hunt down root causes - never use `# type: ignore` as a first resort
7. When adding a module to type checking, also add its tests and fix ALL errors

### 3. Testing
See all rules in sections **## TESTING - IMPORT ORGANIZATION**, **## TESTING - MOCKING BEST PRACTICES**, **## TESTING - TEST DATA CREATION**, **## TESTING - COVERAGE AND ORGANIZATION**, **## TESTING - CONSTANTS AND LITERALS**
1. Always place imports at the top of the file, never inside functions
2. Always use `patch.object` rather than `patch` for mocking in pytest tests
3. Never mock common infrastructure (logger, db connections, etc.) - create testable wrappers instead
4. Never mock Pydantic model classes or data models
5. Always create test data using model classes (Pydantic models), not naked dicts or JSON strings
6. Always check tests/conftest.py and test file helpers before creating new fixtures/helpers
7. Never use naked literals in tests - extract constants or use helper defaults
8. All new code must have 100% test coverage before committing
9. Use descriptive test names that explain what is being tested

### 4. Code Style
See all rules in section **## CODE STYLE**
1. Never use long runs of `=` or other characters for section dividers in files

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
If you add a new rule about async testing in the detailed "TESTING - ASYNC PATTERNS" section, you must add:
- A new topic in Rules Summary: "8. Testing - Async Patterns - See all rules in section **## TESTING - ASYNC PATTERNS**"
- A one-sentence summary under that topic

---

## TYPE CHECKING WITH BASEDPYRIGHT

We use basedpyright for gradual type checking adoption. Currently, type checking is enabled for `src/gv/ai/common/models/` (see `pyrightconfig.json`).

### General Principles

1. **Minimize Logic Changes**: Type checking should NOT change runtime behavior. When adding type hints:
   - Keep original logic intact
   - Use `# type: ignore` comments when necessary
   - Only fix actual type errors, not "improvements"

2. **Test After Type Changes**: Always run tests after adding type hints:
   ```bash
   uv run basedpyright  # Type checking
   uv run pytest tests/  # Full test suite
   ```

### Common Pydantic + Basedpyright Patterns

#### 1. Field Defaults - Use Keyword Syntax

**Problem**: Basedpyright doesn't recognize positional defaults in `Field()`

❌ **Wrong** (causes type errors):
```python
class MyModel(BaseModel):
    role: CommunityRole = Field(CommunityRole.MEMBER, description="...")
    status: bool = Field(False, description="...")
```

✅ **Correct**:
```python
class MyModel(BaseModel):
    role: CommunityRole = Field(default=CommunityRole.MEMBER, description="...")
    status: bool = Field(default=False, description="...")
```

**Why**: Basedpyright requires the `default=` keyword to recognize the default value.

#### 2. Default Values with None - Use Optional

**Problem**: `Type = None` without `Optional` causes type errors

❌ **Wrong**:
```python
def my_function(channel_id: str = None):  # Type error: None not assignable to str
    pass
```

✅ **Correct**:
```python
def my_function(channel_id: Optional[str] = None):
    pass
```

#### 3. List Defaults - Match the Type Annotation

**Problem**: Type annotation says `List[Enum]` but default uses strings

❌ **Wrong** (type mismatch):
```python
DEFAULT_TYPES = [MyEnum.VALUE1.value, MyEnum.VALUE2.value]  # List of strings

class MyModel(BaseModel):
    types: List[MyEnum] = Field(DEFAULT_TYPES, ...)  # Type error!
```

✅ **Correct**:
```python
DEFAULT_TYPES: List[MyEnum] = [MyEnum.VALUE1, MyEnum.VALUE2]  # List of enums

class MyModel(BaseModel):
    types: List[MyEnum] = Field(default=DEFAULT_TYPES, ...)
```

**Note**: If you have validators that convert strings to enums, you can keep string defaults BUT you need to use `# type: ignore` to suppress the error.

#### 4. Function Attributes - Use type: ignore

**Problem**: Python function attributes aren't type-checkable

❌ **Don't refactor to module-level variables** (changes state behavior):
```python
_index = 0  # Module-level state - different semantics!

def my_function():
    global _index
    result = _index
    _index += 1
    return result
```

✅ **Keep function attributes with type: ignore**:
```python
def my_function():
    if not hasattr(my_function, "_index"):
        my_function.__index = 0  # type: ignore[attr-defined]

    result = my_function.__index  # type: ignore[attr-defined]
    my_function.__index += 1  # type: ignore[attr-defined]
    return result
```

**Why**: Function attributes are a legitimate Python pattern. Refactoring to module-level variables changes the semantics (module state vs function state).

#### 5. Complex Property Logic - Preserve with type: ignore

**Problem**: Complex property logic that accesses dynamic attributes

❌ **Don't simplify/change the logic**:
```python
@property
def timestamp(self) -> datetime:
    # Original checks message existence before accessing
    if self.items:
        if self.items[0].message:  # Important check!
            return self.items[0].message.timestamp
    return None
```

Don't remove the `if self.items[0].message:` check just to satisfy the type checker!

✅ **Keep original logic with type: ignore**:
```python
@property
def timestamp(self) -> Optional[datetime]:
    if self.items:
        if self.items[0].message:  # type: ignore[attr-defined]
            return self.items[0].message.timestamp  # type: ignore[attr-defined]
    return None
```

**Why**: The logic might be defending against edge cases or dynamic attributes that the type system can't understand. Changing it could introduce bugs.

#### 6. Optional[bool] Properties - Use bool() for Conversion

**Problem**: Expression returns `Optional[bool]` but property declared as `bool`

✅ **Use bool() for safe conversion**:
```python
@property
def is_valid(self) -> bool:
    return bool(self.check1 and self.check2)  # Safely converts None/Optional to bool
```

**Why**: `bool(None)` returns `False`, which is usually the desired behavior for boolean properties.

#### 7. Type Narrowing - Use Assertions

**Problem**: Optional type needs to be narrowed to non-None

✅ **Use assertions to narrow types**:
```python
@property
def message_id(self) -> str:
    assert self.message is not None, "message must be set"
    assert self.message.message_id is not None, "message_id must be set"
    return self.message.message_id
```

**Why**: Assertions document assumptions and help the type checker understand the guarantees.

#### 8. Decorator Return Types - Fix the Decorator

**Problem**: Decorator that transforms a class but type checker doesn't know the return type

❌ **Wrong** (lazy approach):
```python
# Using type: ignore to cover up the issue
GLOBAL_OPTIONS.values[OPT_MY_SETTING] = value  # type: ignore[index]
```

✅ **Correct** (fix the decorator):
```python
# In the decorator definition file:
from typing import Type, TypeVar

_T = TypeVar("_T")

def my_decorator(default=None, ...):
    def decorator(cls: Type[_T]) -> str:  # Explicitly declare return type
        # ... decorator logic ...
        return option_name  # Returns string, not class
    return decorator

# Now usage works without type: ignore:
GLOBAL_OPTIONS.values[OPT_MY_SETTING] = value  # No error!
```

**Why**: When decorators transform types (e.g., `@register_option` turns a class into a string), the decorator itself needs proper type annotations. Don't use `# type: ignore` to paper over missing annotations - fix the root cause by annotating the decorator's return type.

**Real Example**: In `options_registry.py`, the `@register_option` decorator returns a string (the option name), but without the `-> str` annotation, the type checker thought `OPT_*` constants were class types instead of strings.

### When to Use type: ignore

Use `# type: ignore` for:

1. **Function attributes** (`# type: ignore[attr-defined]`)
2. **Dynamic/runtime attributes** not in the type system
3. **Legacy code patterns** that would require significant refactoring
4. **Protobuf/external library dynamic attributes** (`getstream_webhooks_models.py`, `livekit_webhooks_models.py`)

DO NOT use `# type: ignore` for:

1. **Simple type annotation fixes** (add `Optional`, fix return types)
2. **Field default syntax** (use `Field(default=...)` instead)
3. **Missing imports** (add the import)
4. **Decorator return types** (annotate the decorator properly instead)

### IMPORTANT: Hunt Down Root Causes

**Never use `# type: ignore` as a first resort.** When you encounter a type error:

1. **Investigate the root cause** - Why is the type checker confused?
2. **Fix the source** - Can you add proper type annotations to the original code?
3. **Only then consider type: ignore** - And only for the legitimate cases listed above

Using blanket `# type: ignore` comments without understanding the issue is lazy and pollutes the codebase. Always prefer fixing the actual problem.

### Configuration

See `pyrightconfig.json` for current settings:
- `typeCheckingMode: "standard"` - Balanced strictness
- `reportMissingImports: true` - Catch import errors
- `reportUnusedImport: "warning"` - Catch unused imports
- `reportUnnecessaryTypeIgnoreComment: "warning"` - Catch unnecessary type: ignore

### CI Integration

Type checking runs in CI via `.github/workflows/type-check.yaml`:
- Runs on all branches matching `eng-*`
- Runs on PRs to `main`
- Uses `uv run basedpyright`

### Code Review Checklist

When reviewing type checking changes:

- [ ] Does it change runtime behavior? (If yes, justify or revert)
- [ ] Are tests still passing?
- [ ] Are `type: ignore` comments justified?
- [ ] Could `Field(default=...)` be used instead of `type: ignore`?
- [ ] Are there unnecessary parameter values (that have defaults)?

### Expanding Type Coverage

**CRITICAL RULE: The job is NOT done until tests are also under pyright, WITHOUT ERRORS.**

When adding a module to type checking, you MUST also add its directly connected test files AND fix all type errors in both the module and its tests. This is NON-NEGOTIABLE. This ensures that:
1. Tests are type-checked alongside the code they test
2. Type errors in test code are caught early
3. The module's public API is validated by type-checked tests
4. The entire module (source + tests) is fully type-safe

**DO NOT consider a module "done" if:**
- ❌ Tests are not included in pyrightconfig.json
- ❌ Tests have type errors (even if the source code passes)
- ❌ You're deferring test fixes "for later"

This is not optional. Fix ALL errors before marking the work complete.

To add more modules to type checking:

1. **Identify the module and its tests**:
   - Find tests that directly import from the module: `find tests -name "*.py" -exec grep -l "from gv.ai.module.path" {} \;`
   - Focus on tests in `tests/test_common/test_<module_name>/` for common modules
   - Include API-level tests that directly test the module's functionality

2. **Add both module and tests to `pyrightconfig.json`**:
   ```json
   "include": [
     "src/gv/ai/common/my_module",
     "tests/test_common/test_my_module/"
   ]
   ```

3. **Run type checking and fix errors**:
   ```bash
   uv run basedpyright src/gv/ai/common/my_module tests/test_common/test_my_module/
   ```

4. **Fix errors following these guidelines**:
   - Hunt down root causes (don't use `# type: ignore` as a first resort)
   - Fix decorator return types, add proper annotations
   - Only use `# type: ignore` for legitimate cases (see "When to Use type: ignore" section)

5. **Run tests to verify no behavior changes**:
   ```bash
   uv run pytest tests/test_common/test_my_module/ -v
   ```

6. **Create PR with descriptive commit messages**

**Example**: When adding `common/api` to type checking, we also added:
- `tests/test_common/test_url_friendly_conversions.py` (tests `common/api/formats.py`)
- `tests/test_persistence_service/test_api_base_functions.py` (tests `common/client/persistence_service/api.py` which uses `common/api`)

**Note**: If the tests have many pre-existing type errors, you have two options:
1. **Fix the errors** - Preferred if errors are straightforward (missing annotations, wrong types)
2. **Defer adding tests** - If fixing would require significant refactoring, add the tests in a follow-up PR after cleaning them up

### Resources

- [Basedpyright Documentation](https://docs.basedpyright.com/)
- [Pydantic Type Checking](https://docs.pydantic.dev/latest/concepts/type_adapter/)
- [Mypy Common Issues](https://mypy.readthedocs.io/en/stable/common_issues.html) (many apply to basedpyright)

---

## TESTING - IMPORT ORGANIZATION

### Import Placement Rule

**Always place imports at the top of the file, never inside functions.**

```python
# ✅ CORRECT
from datetime import datetime
import pytest
from unittest.mock import patch

def test_something():
    # test code here
    pass

# ❌ INCORRECT
def test_something():
    from datetime import datetime  # Never do this
    import pytest  # Never do this
    pass
```

**Rationale**: Top-level imports make dependencies clear, allow static analysis tools to work correctly, and avoid repeated import overhead.

---

## TESTING - MOCKING BEST PRACTICES

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

### Never Mock Widely-Used Infrastructure

**Never mock widely-used infrastructure (logger, datetime, random, etc.). Instead, create dedicated helper functions that contain the logic, then patch those helpers.**

```python
# ❌ INCORRECT - Patching infrastructure
@patch("loguru.logger.warning")
def test_staleness_logging(mock_warning):
    convert_to_video_moderation_state(result)
    assert mock_warning.called

# ✅ CORRECT - Patch the business logic helper
# In production code:
def log_participant_staleness_warning(user_id: str, channel_id: str, age_seconds: float, ...) -> None:
    """Helper that encapsulates logging logic."""
    logger.warning(
        f"Participant result became stale: {user_id}",
        extra={
            "event": "participant_result_stale",
            "channel_id": channel_id,
            "user_id": user_id,
            "age_seconds": age_seconds,
            ...
        }
    )

# In test code:
@patch.object(conversion_module, "log_participant_staleness_warning")
def test_staleness_logging(mock_log_staleness):
    convert_to_video_moderation_state(result)
    assert mock_log_staleness.called
    assert mock_log_staleness.call_args.kwargs["user_id"] == "test_user_456"
```

**Why It's Wrong to Mock Infrastructure**:
- **Tests implementation, not behavior**: Testing "did we call logger.warning" instead of actual functionality
- **Brittle and coupled**: Changes to logging don't represent functionality changes
- **Infrastructure is everywhere**: Logger used across all files - patching it is fragile
- **Obscures intent**: What behavior are you actually testing?

**General Principle**:

If you find yourself patching widely-used capabilities (logger, datetime, random, database clients), ask: **"Is there a helper/wrapper I should create instead?"**

**Benefits of Helper Functions**:
- Tests verify business logic, not infrastructure calls
- Helper is reusable across the codebase
- Refactoring implementation doesn't break tests
- Type-safe function signature documents what's being done
- Can add additional logic in one place

### Never Mock Model Classes

**Never mock Pydantic model classes or data models.**

```python
# ✅ CORRECT - Create real instances
from gv.ai.common.models.video_moderation_models import VideoModerationResult

def test_with_real_model():
    result = VideoModerationResult(
        channel_id="test_channel",
        user_id="test_user",
        timestamp=datetime.now(UTC),
        details=VideoModerationDetails(
            is_appropriate=True,
            classification="SAFE"
        )
    )
    # test code using real model instance

# ❌ INCORRECT - Don't mock models
from unittest.mock import MagicMock

def test_with_mocked_model():
    result = MagicMock(spec=VideoModerationResult)  # Never do this
    result.channel_id = "test_channel"
```

**Rationale**: Real model instances ensure validation logic is tested and catch schema issues early.

---

## TESTING - TEST DATA CREATION

### Use Model Classes for Test Data

**Always create test data using model classes, not naked dicts or JSON.**

```python
# ✅ CORRECT - Use model classes
from gv.ai.common.models.video_moderation_models import VideoModerationResult, VideoModerationDetails

def create_test_moderation_result(channel_id: str, user_id: str) -> VideoModerationResult:
    return VideoModerationResult(
        channel_id=channel_id,
        user_id=user_id,
        timestamp=datetime.now(UTC),
        details=VideoModerationDetails(
            is_appropriate=True,
            classification="SAFE",
            severity="low"
        )
    )

# ❌ INCORRECT - Don't use naked dicts
def create_test_data_bad():
    return {  # Never do this
        "channel_id": "test",
        "user_id": "user123",
        "timestamp": "2024-01-01T00:00:00Z"
    }

# ❌ INCORRECT - Don't use JSON strings
def create_test_data_worse():
    return json.loads('{"channel_id": "test"}')  # Never do this
```

**Rationale**: Using model classes ensures type safety, validation, and catches breaking changes in schemas.

### Always Check for Existing Test Helpers First

**Before creating a new test helper or fixture, ALWAYS search for existing ones.**

**Why It Matters**:
- Duplicates code and creates inconsistency
- Different tests might have different defaults for the same data
- Harder to maintain when defaults need to change
- Wastes time reinventing the wheel

**The Search Process**:
```bash
# Before creating a new helper, search for similar ones:
grep -r "def make_test.*context" tests/ --include="*.py"
grep -r "def create_test.*policy" tests/ --include="*.py"
grep -r "def build_test.*result" tests/ --include="*.py"

# Check conftest.py specifically
grep -A 10 "def make_test" tests/conftest.py
```

**Decision Tree**:
1. **Helper exists and fits your needs** → Use it
2. **Helper exists but needs extension** → Extend it with optional parameters
3. **Helper doesn't exist** → Create it in appropriate conftest.py or test module
4. **Helper is module-specific** → Create in module's test file, document it with clear docstring

**Common Fixtures Available**:
- `clean_db`: Cleans the in-memory test database between tests
- `mock_db_client`: Provides a test MongoDB client
- `persistence_app`: FastAPI test app for persistence service

```python
# ✅ CORRECT - Use existing fixture
@pytest.mark.asyncio
async def test_database_operation(clean_db, mock_db_client):
    # Test will have a clean database
    pass

# ❌ INCORRECT - Creating redundant fixture
@pytest.fixture
def clean_video_moderation_collection():  # Don't recreate what exists
    # Reinventing the wheel
    pass
```

---

## TESTING - COVERAGE AND ORGANIZATION

### 100% Coverage for New Code

**All new code must have 100% test coverage before committing.**

This includes:
1. **Unit tests** for individual functions and methods
2. **Integration tests** for API endpoints
3. **Edge case tests** for error handling

### Test Organization

```
tests/
├── test_persistence_service/
│   ├── test_apis/
│   │   └── test_video_moderation_api.py  # API endpoint tests
│   └── test_models/
│       └── test_video_moderation_models.py  # Model validation tests
├── test_video_moderation_service/
│   ├── test_sampling_integration.py  # Integration tests
│   └── test_video_moderator_sampling.py  # Unit tests for VideoModerator
└── conftest.py  # Shared fixtures
```

### pytest Best Practices

#### Async Tests
```python
# ✅ CORRECT
@pytest.mark.asyncio
async def test_async_function():
    result = await my_async_function()
    assert result is not None
```

#### Parametrized Tests
```python
# ✅ CORRECT - Test multiple cases efficiently
@pytest.mark.parametrize("input_val,expected", [
    (10, 20),
    (5, 10),
    (0, 0),
])
def test_doubling(input_val, expected):
    assert double(input_val) == expected
```

#### Test Naming
```python
# ✅ CORRECT - Descriptive test names
def test_latest_per_user_returns_newest_result_per_user():
    pass

def test_policy_reload_after_60_seconds():
    pass

# ❌ INCORRECT - Vague test names
def test_endpoint():
    pass

def test_policy():
    pass
```

### Common Testing Patterns

#### Testing API Endpoints
```python
@pytest.mark.asyncio
async def test_endpoint_returns_correct_data(persistence_app, mock_db_client, clean_db):
    # Arrange: Create test data using model classes
    test_result = VideoModerationResult(
        channel_id="channel_123",
        user_id="user_456",
        timestamp=datetime.now(UTC),
        details=VideoModerationDetails(is_appropriate=True, classification="SAFE")
    )

    collection = mock_db_client[DATABASE_NAME][COLLECTION_NAME]
    await collection.insert_one(test_result.model_dump(by_alias=True))

    # Act: Call endpoint
    async with AsyncClient(app=persistence_app, base_url="http://test") as client:
        response = await client.get("/channels/channel_123/video-moderation")

    # Assert: Verify response
    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) == 1
    assert data["items"][0]["channel_id"] == "channel_123"
```

#### Testing Business Logic with Mocks
```python
from unittest.mock import patch

@pytest.mark.asyncio
@patch.object(LaunchDarklyWrapper, 'get_instance')
async def test_policy_reload(mock_ld_instance):
    # Arrange: Setup mock
    mock_ld = AsyncMock()
    mock_ld.variation.return_value = {"enabled": True, "placements": {}}
    mock_ld_instance.return_value = mock_ld

    # Act: Test logic
    moderator = VideoModerator(channel_id="test", shared_producer=None)
    policy = await moderator._ensure_policy_loaded()

    # Assert: Verify behavior
    assert policy.enabled is True
    mock_ld_instance.assert_called_once()
```

### Running Tests

```bash
# Run all tests with coverage
uv run pytest --cov=src --cov-report=term-missing

# Run specific test file
uv run pytest tests/test_video_moderation_service/test_sampling_integration.py -v

# Run specific test class
uv run pytest tests/test_persistence_service/test_apis/test_video_moderation_api.py::TestLatestPerUserEndpoint -v

# Run with print statements visible
uv run pytest -s
```

---

## TESTING - CONSTANTS AND LITERALS

### NEVER Use Magic Numbers or String Literals

**FORBIDDEN: Using literal values scattered throughout tests. Always extract constants and derive values from authoritative sources.**

**Why It's Wrong**:
- Tests become brittle when constants change
- Unclear where values come from
- Hard to maintain consistency
- Obscures the relationship between values (e.g., 75 = 60 + 15)

```python
# ❌ INCORRECT - Magic numbers everywhere
def test_staleness():
    context = make_test_context(recheck_interval_seconds=60)
    timestamp = datetime.now(UTC) - timedelta(seconds=120)  # Why 120?

    assert extra["age_seconds"] > 75  # Why 75? Unclear!
    assert extra["staleness_threshold_seconds"] == 60  # Repeated magic number

# ✅ CORRECT - Constants from authoritative sources
from gv.ai.common.models.video_moderation_models import MODERATION_DURATION_ESTIMATE

# Test file constants that match helper defaults
DEFAULT_TEST_RECHECK_INTERVAL = 60
STALE_RESULT_AGE_SECONDS = 120  # Must be > recheck_interval + grace_period

def make_test_context(recheck_interval_seconds: int = DEFAULT_TEST_RECHECK_INTERVAL):
    return VideoModerationContext(...)

def test_staleness():
    context = make_test_context()  # Uses default
    timestamp = datetime.now(UTC) - timedelta(seconds=STALE_RESULT_AGE_SECONDS)

    # Derive values from constants - relationship is clear
    threshold_with_grace = (
        DEFAULT_TEST_RECHECK_INTERVAL + MODERATION_DURATION_ESTIMATE.total_seconds()
    )

    assert extra["age_seconds"] > threshold_with_grace
    assert extra["staleness_threshold_seconds"] == DEFAULT_TEST_RECHECK_INTERVAL
    assert extra["grace_period_seconds"] == MODERATION_DURATION_ESTIMATE.total_seconds()
```

### Rules for Test Data

**1. Import from source code**: Use actual constants from the module under test
```python
from gv.ai.common.models.video_moderation_models import (
    VIDEO_MODERATION_STALENESS_THRESHOLD,
    MODERATION_DURATION_ESTIMATE
)
```

**2. Define test-specific constants**: At module level for test defaults
```python
# Top of test file
DEFAULT_TEST_RECHECK_INTERVAL = 60  # Matches helper default
STALE_RESULT_AGE_SECONDS = 120  # Past threshold + grace
TEST_CHANNEL_ID = "test_channel_123"
TEST_USER_ID = "test_user_456"
```

**3. Use helper defaults**: Get values from the helpers that create test data
```python
# ❌ INCORRECT - Hardcoded value
expected_threshold = 60

# ✅ CORRECT - Use helper's default
context = make_test_context()
expected_threshold = DEFAULT_TEST_RECHECK_INTERVAL  # Same value helper uses
```

**4. Calculate derived values**: Show the relationship explicitly
```python
# ✅ CORRECT - Relationship is obvious
threshold_with_grace = (
    DEFAULT_TEST_RECHECK_INTERVAL + MODERATION_DURATION_ESTIMATE.total_seconds()
)
stale_age = threshold_with_grace + 10  # Clearly 10s past threshold
```

**5. Put defaults in helper signatures**: Literals only in helper parameter defaults
```python
def make_test_context(
    recheck_interval_seconds: int = DEFAULT_TEST_RECHECK_INTERVAL,
    user_id: str = TEST_USER_ID
) -> VideoModerationContext:
    ...
```

### When Literals Are Acceptable vs Forbidden

**✅ Acceptable**:
- Test IDs: `"test_user_123"`, `"test_channel_456"` (unique per test)
- Obvious values: `True`, `False`, `0`, `1`, `None`
- String messages: `"Test error message"`

**❌ FORBIDDEN**:
- Timing values: `60`, `75`, `120` (use constants)
- Thresholds: `0.7`, `0.5` (extract from config/constants)
- Computed values: `75` (should be `60 + 15`)
- Repeated values: Same number appears multiple times

### Warning Signs You're Doing It Wrong

- Seeing the same number repeated across multiple tests
- Comments explaining "why this specific value"
- Tests break when unrelated constants change
- Unclear where test values come from

### Benefits

- Tests adapt automatically when constants change
- Relationships between values are explicit
- Single source of truth for test defaults
- Easy to understand test intentions
- Refactoring is safer

---

## CODE STYLE

### File Organization - No Section Dividers

**Never use long runs of `=` characters for section dividers in files.**

```python
# ❌ INCORRECT - Don't use long section dividers
# ============================================================================
# Fixtures
# ============================================================================

# ✅ CORRECT - Use simple comments
# Fixtures
```

**Rationale**: Long character runs are visual noise that don't add value and make code harder to read.

---

## TESTING - SUMMARY CHECKLIST

Before committing tests, verify:
- [ ] All imports are at the top of the file
- [ ] Using `patch.object` instead of `patch`
- [ ] No mocking of common infrastructure (using testable wrappers instead)
- [ ] No mocking of model classes
- [ ] All test data created using model classes (not dicts/JSON)
- [ ] Checked tests/conftest.py for existing fixtures before creating new ones
- [ ] No naked literals - all timing values, thresholds use constants
- [ ] 100% coverage of new code
- [ ] All tests pass
- [ ] Descriptive test names
- [ ] Edge cases covered
- [ ] No long runs of `=` or other characters for section dividers
