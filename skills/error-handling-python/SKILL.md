---
name: error-handling-python
description: "Python error handling: exception chaining with 'from', assert vs raise for production, structured logging, custom exception hierarchies"
---

# Error Handling in Python

Python companion to the error-handling-patterns skill (in pyze-workflow). This skill provides Python-specific implementation patterns for fail-fast error handling.

## Assert vs Raise

Python's `assert` statements are **stripped entirely** when running with the `-O` (optimize) flag. Many deployment tools enable this by default. Never use `assert` for checks that must run in production.

**WRONG — assert for production validation:**

```python
def transfer_funds(from_account, to_account, amount):
    assert amount > 0, "Amount must be positive"
    assert from_account.balance >= amount, "Insufficient funds"
    # With python -O, these checks disappear silently
```

**RIGHT — explicit raise for production validation:**

```python
def transfer_funds(from_account, to_account, amount):
    if amount <= 0:
        raise ValueError(f"Amount must be positive, got {amount}")
    if from_account.balance < amount:
        raise InsufficientFundsError(
            f"Account {from_account.id} has {from_account.balance}, needs {amount}"
        )
```

**When assert IS safe:**
- Development-time invariants you want to catch during testing
- Test assertions (`assert result == expected` in pytest)
- Type narrowing hints for static analysis

---

## Exception Chaining

Always use `raise ... from e` to preserve the original exception. Without `from`, the original traceback and context are lost.

**WRONG — losing the original exception:**

```python
try:
    result = db.execute(query)
except DatabaseError:
    raise ServiceError("Query failed")
    # Original DatabaseError traceback is gone
```

**RIGHT — chaining with `from`:**

```python
try:
    result = db.execute(query)
except DatabaseError as e:
    raise ServiceError(f"Query failed: {query[:100]}") from e
    # Original traceback preserved, both exceptions visible in logs
```

**Suppressing the chain** (rare, intentional): Use `raise ... from None` only when the original exception would leak internal details to an external caller.

---

## Custom Exception Hierarchy

Define domain-specific exceptions so callers can handle errors precisely. Catching bare `Exception` hides bugs.

**WRONG — generic exceptions:**

```python
def authenticate(email, password):
    user = db.get_user(email)
    if not user:
        raise Exception("User not found")  # caller can't distinguish this
    if not verify_password(password, user.hash):
        raise Exception("Wrong password")  # from any other Exception
```

**RIGHT — domain exception hierarchy:**

```python
class AppError(Exception):
    """Base for all application errors."""

class AuthError(AppError):
    """Authentication failures."""

class NotFoundError(AppError):
    """Resource not found."""

class ValidationError(AppError):
    """Input validation failures."""

def authenticate(email, password):
    user = db.get_user(email)
    if not user:
        raise NotFoundError(f"No user with email {email}")
    if not verify_password(password, user.hash):
        raise AuthError("Invalid credentials")

# Callers handle specific cases:
try:
    token = authenticate(email, password)
except NotFoundError:
    return {"error": "Unknown email"}, 404
except AuthError:
    return {"error": "Invalid credentials"}, 401
```

---

## Structured Logging

Attach exceptions as objects, not strings. Use `extra` for structured context. Use `logger.exception()` to automatically include the traceback.

**WRONG — string formatting loses context:**

```python
except Exception as e:
    logger.error(f"Failed to process order: {e}")
    # Loses: traceback, exception type, structured fields
```

**RIGHT — structured logging with context:**

```python
except Exception as e:
    logger.exception(
        "Failed to process order",
        extra={
            "order_id": order.id,
            "user_id": user.id,
            "amount": order.total,
        },
    )
    # Preserves: full traceback, exception type, structured fields
```

**Severity levels:**
- `logger.error()` — failures requiring attention (failed payment, data corruption)
- `logger.warning()` — recoverable issues (retry succeeded, deprecated API used)
- `logger.info()` — operational events (order placed, user logged in)
- `logger.debug()` — development tracing (function entry/exit, intermediate values)

**For production services**, consider `structlog` for JSON-structured output that integrates with log aggregation tools.

---

## Boundary Validation

Validate inputs at system boundaries (API endpoints, CLI entry points, message consumers). Use Pydantic for declarative validation.

**WRONG — manual validation scattered through code:**

```python
def create_order(data: dict):
    if "items" not in data:
        raise ValueError("items required")
    if not isinstance(data["items"], list):
        raise TypeError("items must be a list")
    if len(data["items"]) == 0:
        raise ValueError("at least one item required")
    # Validation logic tangled with business logic
```

**RIGHT — Pydantic model at the boundary:**

```python
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    items: list[OrderItem] = Field(..., min_length=1)
    customer_id: int
    shipping_address: str

# At the API boundary:
def create_order_endpoint(raw_data: dict):
    request = CreateOrderRequest(**raw_data)  # validates or raises
    return order_service.create(request)       # business logic gets clean data
```

---

## Retryable vs Permanent Errors

Classify errors to decide whether to retry or fail fast.

```python
import time
from functools import wraps

class RetryableError(Exception):
    """Transient errors that may succeed on retry."""

class PermanentError(Exception):
    """Errors that will not resolve with retry."""

def with_retry(max_attempts=3, backoff=1.0):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return fn(*args, **kwargs)
                except RetryableError as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(backoff * (2 ** attempt))
                # PermanentError propagates immediately — no retry
        return wrapper
    return decorator

# Usage:
@with_retry(max_attempts=3)
def call_api(endpoint, payload):
    response = requests.post(endpoint, json=payload)
    if response.status_code == 429:
        raise RetryableError("Rate limited")
    if response.status_code == 401:
        raise PermanentError("Invalid API key")  # don't retry auth failures
    response.raise_for_status()
    return response.json()
```
