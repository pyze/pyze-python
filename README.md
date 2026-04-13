# pyze-python

Python companion skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Grounds [pyze-workflow](https://github.com/pyze/pyze-workflow)'s language-agnostic principles with concrete Python code examples.

## Why install this

- **Claude avoids Python-specific gotchas.** Wrong/right code examples teach Claude about [`@lru_cache`](https://docs.python.org/3/library/functools.html#functools.lru_cache) on impure functions, [`assert`](https://docs.python.org/3/reference/simple_stmts.html#the-assert-statement) in production (stripped by `python -O`), bare `except`, and missing `raise ... from e` chains.
- **Idiomatic test patterns.** [pytest](https://docs.pytest.org/) anti-patterns — `mock.patch` pitfalls, fixture gaps, `_private` access, config divergence — so generated tests actually exercise production code paths.
- **Principled, not prescriptive.** Each skill is a companion to a pyze-workflow principle, not a standalone rulebook. The workflow plugin provides the "why"; this plugin provides the Python "how".

## Installation

Install alongside pyze-workflow for the full experience:

```bash
claude plugin add pyze/pyze-workflow
claude plugin add pyze/pyze-python
```

See the [Claude Code plugin documentation](https://docs.anthropic.com/en/docs/claude-code/plugins) for details on plugin management.

## Relationship to pyze-workflow

[pyze-workflow](https://github.com/pyze/pyze-workflow) provides language-agnostic principles (decomplection, testing patterns, error handling, caching/purity). pyze-python grounds those principles with Python-specific WRONG/RIGHT code examples.

You can use pyze-python standalone, but pairing it with pyze-workflow gives you PDCA cycle management, risk assessment, and quality gate hooks.

## Skills

Each skill is a companion to a general principle in pyze-workflow:

| Skill | Companion to | What it covers |
|-------|-------------|----------------|
| decomplection-python | [decomplection-first-design](https://github.com/pyze/pyze-workflow) | Explicit dependencies, pure core + IO boundary, [dataclass](https://docs.python.org/3/library/dataclasses.html) composition, avoiding hidden state |
| testing-patterns-python | [testing-patterns](https://github.com/pyze/pyze-workflow) | Six [pytest](https://docs.pytest.org/) anti-patterns: [`mock.patch`](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.patch) pitfalls, fixture gaps, `_private` access, config divergence |
| error-handling-python | [error-handling-patterns](https://github.com/pyze/pyze-workflow) | [`raise ... from e`](https://docs.python.org/3/tutorial/errors.html#exception-chaining) chaining, `assert` vs `raise` for production, [structured logging](https://docs.python.org/3/library/logging.html), custom exceptions |
| caching-and-purity-python | [caching-and-purity](https://github.com/pyze/pyze-workflow) | [`@lru_cache`](https://docs.python.org/3/library/functools.html#functools.lru_cache) on impure functions, mutable return gotchas, hidden time dependencies |
| bdd-scenarios-python | [bdd-scenarios](https://github.com/pyze/pyze-workflow) | [Given/When/Then](https://martinfowler.com/bliki/GivenWhenThen.html) in pytest, [`@pytest.mark.parametrize`](https://docs.pytest.org/en/stable/how-to/parametrize.html), fixtures as preconditions |
| tool-selection-python | [tool-selection](https://github.com/pyze/pyze-workflow) | [Pyright](https://github.com/microsoft/pyright)/[Pylance](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance) LSP, [IPython](https://ipython.org/) REPL, grep false positives with dynamic imports |

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

## Further Reading

- [Python Documentation](https://docs.python.org/3/) — standard library reference
- [pytest Documentation](https://docs.pytest.org/) — testing framework
- [Simple Made Easy](https://www.infoq.com/presentations/Simple-Made-Easy/) — Rich Hickey on decomplection (the principle behind decomplection-python)
- [Claude Code Plugins](https://docs.anthropic.com/en/docs/claude-code/plugins) — how to install and manage plugins

## License

MIT
