# pyze-python

Python companion skills for [pyze-workflow](https://github.com/pyze/pyze-workflow). Provides concrete Python code examples for pyze-workflow's language-agnostic principles — pytest patterns, `@lru_cache` correctness, exception chaining, structured logging, and more.

## Installation

Install alongside pyze-workflow for the full experience:

```bash
claude plugin add pyze/pyze-workflow
claude plugin add pyze/pyze-python
```

## Skills

Each skill is a companion to a general principle in pyze-workflow, grounding it with Python-specific WRONG/RIGHT code examples.

| Skill | Companion to | What it covers |
|-------|-------------|----------------|
| decomplection-python | decomplection-first-design | Explicit dependencies, pure core + IO boundary, dataclass composition, avoiding hidden state |
| testing-patterns-python | testing-patterns | Six pytest anti-patterns: `mock.patch` pitfalls, fixture gaps, `_private` access, config divergence |
| error-handling-python | error-handling-patterns | `raise ... from e` chaining, `assert` vs `raise` for production, structured logging, custom exceptions |
| caching-and-purity-python | caching-and-purity | `@lru_cache` on impure functions, mutable return gotchas, hidden time dependencies |
| bdd-scenarios-python | bdd-scenarios | Given/When/Then in pytest, `@pytest.mark.parametrize`, fixtures as preconditions |
| tool-selection-python | tool-selection | pyright/pylance LSP, IPython REPL, grep false positives with dynamic imports |

## Example

From **error-handling-python** — why `assert` is unsafe in production:

```python
# WRONG — stripped by python -O
assert api_key, "API_KEY required"

# RIGHT — always runs
if not api_key:
    raise ValueError("API_KEY environment variable required")
```

From **caching-and-purity-python** — hidden dependency bug:

```python
# WRONG — os.environ not in cache key, returns stale config
@lru_cache(maxsize=128)
def get_api_config(service: str) -> dict:
    env = os.environ.get("ENV", "production")  # hidden input!
    return {"url": URLS[env][service]}

# RIGHT — all dependencies explicit
@lru_cache(maxsize=128)
def get_api_config(service: str, env: str) -> dict:
    return {"url": URLS[env][service]}
```

## License

MIT
