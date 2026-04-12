---
name: testing-patterns-python
description: "This skill should be used when writing or reviewing pytest tests, debugging test/production divergence, auditing mock.patch usage, or checking test fixtures for configuration gaps in Python."
---

# Testing Patterns in Python

Python companion to the testing-patterns skill (in pyze-workflow). This skill shows how the six universal testing anti-patterns manifest in Python with pytest.

## Anti-Pattern 1: Private Implementation Access

Tests that reach into private methods or attributes are coupled to implementation, not behavior.

**WRONG — accessing private method:**

```python
def test_user_validation():
    service = UserService(db)
    # Reaching into private implementation
    result = service._validate_email("test@example.com")
    assert result is True
```

**RIGHT — test through public API:**

```python
def test_user_creation_validates_email():
    service = UserService(db)
    user = service.create_user(email="test@example.com", name="Test")
    assert user.email == "test@example.com"

def test_user_creation_rejects_invalid_email():
    service = UserService(db)
    with pytest.raises(ValidationError, match="invalid email"):
        service.create_user(email="not-an-email", name="Test")
```

**Detection:** `grep -rn '\._[a-z]' tests/` — find tests accessing underscore-prefixed members.

---

## Anti-Pattern 2: Configuration Divergence

When production calls a function with specific settings but tests use bare defaults, the test exercises a different code path.

**WRONG — tests use different defaults than production:**

```python
# Production code
def process_batch(records, workers=8, timeout=30, retry=True):
    ...

# Test — exercises default workers=8, timeout=30, retry=True
def test_process_batch():
    result = process_batch(test_records)
    assert len(result) == 5

# But production caller uses different settings:
# process_batch(records, workers=2, timeout=120, retry=False)
```

**RIGHT — test with production configuration:**

```python
PROD_CONFIG = {"workers": 2, "timeout": 120, "retry": False}

def test_process_batch_with_production_config():
    result = process_batch(test_records, **PROD_CONFIG)
    assert len(result) == 5
```

**Detection:** Compare function signatures with their call sites. Look for functions with many defaulted parameters where tests call with fewer arguments than production.

---

## Anti-Pattern 3: Manual Internal Data Construction

Hand-building dicts or objects that mirror internal data shapes. When the shape changes, the test silently diverges.

**WRONG — hand-built dict matching ORM shape:**

```python
def test_user_report():
    # Hand-built dict that mirrors User model internals
    user_data = {
        "id": 1,
        "name": "Alice",
        "email": "alice@example.com",
        "tier": "gold",
        "created_at": "2024-01-01",
    }
    report = generate_report(user_data)
    assert "Alice" in report
```

**RIGHT — use production factories:**

```python
def test_user_report():
    user = User.create(name="Alice", email="alice@example.com")
    # Or with Pydantic: User(name="Alice", email="alice@example.com")
    report = generate_report(user.model_dump())
    assert "Alice" in report
```

The factory/model ensures the data shape matches what production code actually produces, including any default fields, computed values, or validators.

---

## Anti-Pattern 4: Test Helper Passthrough Gaps

Helpers or fixtures that accept options but don't forward all of them to the system under test.

**WRONG — helper drops options:**

```python
@pytest.fixture
def api_client(request):
    """Create a test API client."""
    auth = request.param.get("auth", True)
    # Missing: timeout, retries, headers from request.param
    client = APIClient(authenticated=auth)
    return client
```

**RIGHT — forward all options:**

```python
@pytest.fixture
def api_client(request):
    """Create a test API client with full configuration."""
    defaults = {"auth": True, "timeout": 30, "retries": 3}
    config = {**defaults, **getattr(request, "param", {})}
    return APIClient(**config)
```

**Detection:** Compare fixture/helper parameters against the constructor or function they wrap. Every production parameter should be passable through the helper.

---

## Anti-Pattern 5: Excessive Mocking

`@patch` creates a test-only code path. The mock succeeds but the real integration may fail.

**WRONG — mocking away the thing you should test:**

```python
@patch("app.services.db.query")
def test_get_user(mock_query):
    mock_query.return_value.get.return_value = User(id=1, name="Alice")
    service = UserService()
    user = service.get_user(1)
    assert user.name == "Alice"
    # This test passes even if the real query is broken
```

**RIGHT — dependency injection with test double:**

```python
def test_get_user():
    test_db = InMemoryDatabase()
    test_db.add(User(id=1, name="Alice"))
    service = UserService(db=test_db)
    user = service.get_user(1)
    assert user.name == "Alice"
    # The real query path is exercised through InMemoryDatabase
```

**When mocking IS appropriate:**
- External HTTP APIs (use `responses` or `httpx_mock`)
- System clock (`freezegun`)
- File system operations in unit tests

**Detection:** `grep -rn '@patch\|mock\.patch' tests/ | wc -l` — high count suggests over-reliance on mocking.

---

## Anti-Pattern 6: Duplicated Production Logic

Test helpers that reimplement production logic. When production changes, tests still pass against the stale reimplementation.

**WRONG — test reimplements validation:**

```python
def assert_valid_email(email: str):
    """Test helper that checks email format."""
    # This reimplements production validation — may diverge
    assert "@" in email
    assert "." in email.split("@")[1]

def test_user_email():
    user = create_user(email="test@example.com")
    assert_valid_email(user.email)
```

**RIGHT — use production validation:**

```python
def test_user_email():
    user = create_user(email="test@example.com")
    # Call the actual production validator
    assert validate_email(user.email) is True

def test_invalid_email_rejected():
    with pytest.raises(ValidationError):
        create_user(email="not-valid")
```

**Detection:** Compare test helper function bodies against production functions. If they do the same thing, the test helper should call the production function instead.
