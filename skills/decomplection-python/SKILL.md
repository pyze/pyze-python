---
name: decomplection-python
description: "Python decomplection patterns: explicit dependencies, pure functions, dataclass composition, avoiding hidden state"
---

# Decomplection in Python

Python companion to the decomplection-first-design skill (in pyze-workflow). This skill provides Python-specific code examples for the general principles.

## Hidden State

Functions that read from global or ambient state are complected — their behavior depends on things not visible in their signature.

**WRONG — implicit dependency on environment:**

```python
import os

def get_api_url():
    env = os.environ.get("ENV", "production")
    if env == "production":
        return "https://api.example.com"
    return "https://staging.example.com"
```

**RIGHT — explicit parameter:**

```python
def get_api_url(env: str) -> str:
    if env == "production":
        return "https://api.example.com"
    return "https://staging.example.com"
```

The explicit version is testable without monkeypatching `os.environ`, composable with any config source, and its behavior is fully determined by its inputs.

**Detection:** `grep -rn 'os\.environ\|global ' src/` in non-entrypoint files.

---

## Pure Core + IO Boundary

Separate computation from I/O. The pure core is testable, cacheable, and composable. The thin IO wrapper at the boundary handles side effects.

**WRONG — fetching and transforming interleaved:**

```python
def get_user_report(user_id: int) -> str:
    user = db.query(User).get(user_id)          # IO
    orders = db.query(Order).filter_by(user_id=user_id).all()  # IO
    total = sum(o.amount for o in orders)        # Pure
    tier = "gold" if total > 1000 else "standard"  # Pure
    return f"{user.name}: {tier} (${total})"     # Pure
```

**RIGHT — pure transform + thin IO caller:**

```python
# Pure — testable without database
def format_user_report(name: str, order_amounts: list[float]) -> str:
    total = sum(order_amounts)
    tier = "gold" if total > 1000 else "standard"
    return f"{name}: {tier} (${total})"

# Thin IO boundary
def get_user_report(user_id: int) -> str:
    user = db.query(User).get(user_id)
    orders = db.query(Order).filter_by(user_id=user_id).all()
    return format_user_report(user.name, [o.amount for o in orders])
```

---

## Composition over Inheritance

Deep class hierarchies braid together concerns. Prefer composition with small, focused objects and Protocol for polymorphism.

**WRONG — inheritance hierarchy entangles concerns:**

```python
class BaseProcessor:
    def validate(self, data): ...
    def transform(self, data): ...
    def save(self, data): ...

class CSVProcessor(BaseProcessor):
    def validate(self, data): ...  # CSV-specific
    def transform(self, data): ...  # CSV-specific
    def save(self, data): ...  # same as base — inherited for free but entangled

class JSONProcessor(BaseProcessor):
    def validate(self, data): ...  # JSON-specific
    def transform(self, data): ...  # JSON-specific
    # save is inherited — changing BaseProcessor.save affects all subclasses
```

**RIGHT — composition with Protocol:**

```python
from typing import Protocol
from dataclasses import dataclass

class Validator(Protocol):
    def validate(self, data: bytes) -> dict: ...

class Transformer(Protocol):
    def transform(self, record: dict) -> dict: ...

class Storage(Protocol):
    def save(self, record: dict) -> None: ...

@dataclass
class Pipeline:
    validator: Validator
    transformer: Transformer
    storage: Storage

    def run(self, data: bytes) -> None:
        record = self.validator.validate(data)
        transformed = self.transformer.transform(record)
        self.storage.save(transformed)
```

Each concern is independent. Swap validators without touching storage. Test transformers without I/O.

---

## DDRY — Decomplected Don't Repeat Yourself

When extracting shared logic, compose small functions rather than adding mode parameters that branch internally.

**WRONG — mode parameter creates internal branching:**

```python
def process_data(records: list[dict], mode: str = "full") -> list[dict]:
    results = []
    for record in records:
        if mode == "full":
            validated = deep_validate(record)
            enriched = enrich_from_api(validated)
            results.append(enriched)
        elif mode == "quick":
            validated = shallow_validate(record)
            results.append(validated)
        elif mode == "dry_run":
            validated = deep_validate(record)
            results.append({"would_process": validated})
    return results
```

**RIGHT — composable functions:**

```python
def validate_records(records, validate_fn=deep_validate):
    return [validate_fn(r) for r in records]

def enrich_records(records, enrich_fn=enrich_from_api):
    return [enrich_fn(r) for r in records]

# Compose as needed at call sites:
# Full: enrich_records(validate_records(data))
# Quick: validate_records(data, validate_fn=shallow_validate)
# Dry run: validate_records(data)  # just validate, don't enrich
```

---

## Mutable State

Mutating state in place braids together "what happened before" with "what happens next." Returning new values keeps steps independent.

**WRONG — mutation entangles order of operations:**

```python
@dataclass
class Cart:
    items: list[str]
    total: float = 0.0

    def add_item(self, item: str, price: float):
        self.items.append(item)  # mutation
        self.total += price       # mutation — depends on prior state
```

**RIGHT — immutable updates return new state:**

```python
from dataclasses import dataclass, field, replace

@dataclass(frozen=True)
class Cart:
    items: tuple[str, ...] = ()
    total: float = 0.0

    def add_item(self, item: str, price: float) -> "Cart":
        return replace(self, items=self.items + (item,), total=self.total + price)
```

Each operation produces a new Cart. No mutation, no ordering dependency, easy to test.

---

## Explicit Dependencies via Constructor

Reach into global imports to get services (WRONG) vs receive them as constructor parameters (RIGHT).

**WRONG — hidden import dependency:**

```python
from app.database import db  # module-level global

class UserService:
    def get_user(self, user_id: int) -> User:
        return db.query(User).get(user_id)  # which db? can't test without patching
```

**RIGHT — constructor injection:**

```python
class UserService:
    def __init__(self, db: Database):
        self.db = db

    def get_user(self, user_id: int) -> User:
        return self.db.query(User).get(user_id)

# Production: UserService(db=production_db)
# Test: UserService(db=in_memory_db)
```

**Detection:** `grep -rn 'from.*import.*db\|from.*import.*cache\|from.*import.*client' src/` — look for service imports at module level in non-entrypoint files.
