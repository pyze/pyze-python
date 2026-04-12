---
name: caching-and-purity-python
description: "This skill should be used when using @lru_cache or functools caching in Python, diagnosing stale cache results, or evaluating whether a Python function is pure enough to cache safely."
---

# Caching and Purity in Python

Python companion to the caching-and-purity skill (in pyze-workflow). This skill shows how caching bugs manifest in Python and how to fix them.

## @lru_cache on Impure Functions

`@lru_cache` memoizes based on arguments. If function behavior depends on something NOT in the arguments, the cache returns stale results.

**WRONG — hidden dependency on environment:**

```python
from functools import lru_cache
import os

@lru_cache(maxsize=128)
def get_api_config(service_name: str) -> dict:
    env = os.environ.get("ENV", "production")  # NOT in cache key
    base_url = URLS[env][service_name]
    return {"url": base_url, "timeout": 30}

# First call with ENV=staging: returns staging config, cached
# ENV changes to production: still returns staging config!
```

**RIGHT — make all dependencies explicit:**

```python
@lru_cache(maxsize=128)
def get_api_config(service_name: str, env: str) -> dict:
    base_url = URLS[env][service_name]
    return {"url": base_url, "timeout": 30}

# Caller provides env explicitly:
config = get_api_config("users", env=os.environ.get("ENV", "production"))
```

---

## Hidden Dependency: Wall-Clock Time

Functions that read `datetime.now()` are impure — their output changes even with identical arguments.

**WRONG — time-dependent, cache returns stale result:**

```python
@lru_cache(maxsize=256)
def is_token_valid(token_id: str) -> bool:
    token = token_store.get(token_id)
    return datetime.now() < token.expires_at  # time not in cache key
    # Cached True result persists even after token expires
```

**RIGHT — pass time as parameter:**

```python
def is_token_valid(token_id: str, now: datetime) -> bool:
    token = token_store.get(token_id)
    return now < token.expires_at
    # Don't cache this — result depends on continuously changing input
```

---

## Mutable Return Gotcha

`@lru_cache` caches the actual object. If callers mutate it, the cached value is corrupted for all future callers.

**WRONG — mutable dict returned from cache:**

```python
@lru_cache(maxsize=1)
def get_defaults() -> dict:
    return {"debug": False, "workers": 4, "timeout": 30}

# Caller 1:
config = get_defaults()
config["debug"] = True  # MUTATES THE CACHED DICT

# Caller 2:
config = get_defaults()
print(config["debug"])  # True — corrupted by Caller 1!
```

**RIGHT — return immutable or copy:**

```python
from types import MappingProxyType
from dataclasses import dataclass

# Option 1: Immutable proxy
@lru_cache(maxsize=1)
def get_defaults() -> MappingProxyType:
    return MappingProxyType({"debug": False, "workers": 4, "timeout": 30})
# Callers get TypeError if they try to mutate

# Option 2: Frozen dataclass
@dataclass(frozen=True)
class AppConfig:
    debug: bool = False
    workers: int = 4
    timeout: int = 30

@lru_cache(maxsize=1)
def get_defaults() -> AppConfig:
    return AppConfig()
# Callers get FrozenInstanceError if they try to mutate
```

---

## Cache Key Completeness

Ask: does the function's output depend on anything not in its parameters? If yes, either add that dependency as a parameter or don't cache.

**Diagnostic checklist before adding `@lru_cache`:**

1. Does the function read from `self`, globals, environment, or config? → Include them in parameters or don't cache
2. Does the function call I/O (database, HTTP, filesystem)? → Don't cache the function; cache at a higher level with explicit invalidation
3. Does the function return a mutable object? → Return frozen/immutable version
4. Does the function depend on time? → Don't cache, or include a time bucket as key

---

## When NOT to Cache

**Don't use `@lru_cache` on:**
- Functions with side effects (writing to DB, sending emails)
- Functions reading mutable external state (DB queries, HTTP calls)
- Functions returning mutable objects without freezing
- Methods on mutable objects (`self` changes between calls)
- Functions where correctness depends on wall-clock time

**Do use `@lru_cache` on:**
- Pure computation (math, string formatting, parsing)
- Configuration loading at startup (call once, cache forever)
- Expensive pure transformations (compiled regex, parsed templates)
