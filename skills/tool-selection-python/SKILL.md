---
name: tool-selection-python
description: "This skill should be used when deciding between pyright LSP, IPython REPL, and grep for Python code navigation, or when grep produces false positives from dynamic imports and string matching."
---

# Tool Selection for Python

Python companion to the tool-selection skill (in pyze-workflow). This skill maps the general tool hierarchy (LSP > REPL > Grep) to Python-specific tools and patterns.

## LSP: pyright / pylance

Python's LSP servers (pyright, pylance) understand the type system and import graph. Use for structural queries.

**What LSP handles well:**
- **Find all references** — understands `from app.models import User as U` and tracks both `User` and `U`
- **Go to definition** — follows through re-exports, `__init__.py` imports, decorators
- **Rename symbol** — renames across all files that import it
- **Find implementations** — shows all classes implementing a Protocol or ABC

**Example — finding callers of a function:**

LSP "Find References" on `validate_email` returns:
```
app/services/user.py:45 — validate_email(data["email"])
app/api/registration.py:23 — validate_email(request.email)
tests/test_validation.py:12 — validate_email("test@example.com")
```

Grep for `validate_email` also returns false positives:
```
app/errors.py:8 — "validate_email failed"  (string literal)
docs/api.md:34 — validate_email endpoint   (documentation)
app/utils.py:1 — # TODO: refactor validate_email  (comment)
```

**When LSP struggles:**
- `getattr(obj, method_name)` — dynamic attribute access
- `**kwargs` forwarding — can't trace which keys are used downstream
- Metaclass-generated methods — `type()` or `__init_subclass__` creating methods at runtime
- Plugin systems using `importlib.import_module()` — dynamic imports invisible to static analysis

---

## REPL: IPython / python -i

The Python REPL excels at runtime exploration — understanding what objects look like, what methods they have, and how libraries behave.

**When REPL wins over reading source:**

```python
# Understanding a third-party return type:
>>> import requests
>>> r = requests.get("https://httpbin.org/json")
>>> type(r)
<class 'requests.models.Response'>
>>> dir(r)  # see all attributes
>>> r.json()  # test behavior directly

# Exploring an ORM model:
>>> from app.models import User
>>> u = User.query.first()
>>> u.__dict__  # see actual fields and values
>>> type(u.created_at)  # datetime? string? None?
```

**When to use REPL vs LSP:**
- "What type does this function return?" → REPL (`type(fn())`)
- "Where is this function called?" → LSP (Find References)
- "What happens when I pass None?" → REPL (try it)
- "What implements this Protocol?" → LSP (Find Implementations)

---

## Grep: Text Pattern Matching

Use grep for literal text search — config values, error messages, TODO comments. Don't use it for code structure queries.

**Grep false positives in Python:**

| Search | False Positives |
|--------|----------------|
| `validate` | Matches `"validation error"`, `# validate later`, `class Validator` |
| `User` | Matches `UserError`, `user_name`, `"User not found"`, `# User model` |
| `import db` | Matches `# import db later`, `"failed to import db"` |
| `process` | Matches `subprocess`, `process_id`, `"processing complete"` |

**When grep IS the right tool:**
- Finding error message text: `grep -rn "Invalid API key" src/`
- Finding config keys: `grep -rn "DATABASE_URL" .`
- Finding TODOs: `grep -rn "TODO\|FIXME\|HACK" src/`
- Searching non-Python files: YAML configs, JSON fixtures, Dockerfiles

---

## Python-Specific Decision Tree

```
What are you looking for?
│
├─ Code relationships (who calls X, who implements Y)
│   └─ LSP: Find References, Find Implementations
│
├─ Runtime behavior (what does X return, what type is Y)
│   └─ REPL: import and call it
│
├─ Dynamic code (getattr, importlib, metaclass)
│   └─ REPL: instantiate and inspect with dir(), type()
│
├─ Text patterns (error messages, config keys, TODOs)
│   └─ Grep: literal string search
│
├─ Type information (what's the signature of X)
│   ├─ Static types annotated? → LSP: Hover
│   └─ No annotations? → REPL: help(fn), inspect.signature(fn)
│
└─ Dead code (is X used anywhere?)
    └─ LSP: Find References (0 results = likely dead)
    └─ NOT grep (catches comments, strings, documentation)
```

---

## Tool Sequencing

Sometimes you need multiple tools in sequence:

1. **Grep → LSP**: `grep -rn "deprecated" src/` finds deprecated functions, then LSP "Find References" on each to see if they're still called
2. **LSP → REPL**: LSP shows a function signature with complex types, REPL call reveals actual runtime shape
3. **REPL → Grep**: REPL reveals an object uses `getattr` for method dispatch, grep finds the string keys used across the codebase
