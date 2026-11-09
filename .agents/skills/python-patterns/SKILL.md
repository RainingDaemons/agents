---
name: python-patterns
description: Idiomatic Python, PEP 8, type hints, and best practices for building robust, efficient, maintainable Python. Language-level only.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated following PEP 8 and PEP 257 conventions.
---

# Python Development Patterns

Idiomatic Python patterns and best practices for building robust, efficient, and maintainable applications.

> Scope: the **Python language only**. Framework patterns live in `fastapi`; service architecture in `python-backend-design`; HTTP contract design in `api-design`. Formatting/linting/type-checking tooling is owned by the area's `conventions.md` and `verification.md` — run what they prescribe.

## When to use

Use this skill when working on Python code in `cdn/` or `scripts/`:

- Writing, reviewing, or refactoring Python at the language level
- Choosing types and typing (`Protocol`, `TypeVar`, dataclasses vs `NamedTuple`, generics)
- Designing error handling and exception hierarchies
- Working with context managers, decorators, comprehensions, or generators
- Deciding the concurrency model (threads vs processes vs `asyncio`) or memory/performance trade-offs
- Organizing packages, modules, exports, and imports
- Applying PEP 8 naming/formatting and PEP 257 docstrings

Do not use this skill when:

- You need FastAPI routes, dependencies, Pydantic models, or SSE → `fastapi`
- You need CDN service wiring (lifespan, psycopg pool, settings, uploads, rate limiting) → `python-backend-design`
- You need HTTP URL/status/pagination conventions → `api-design`
- You need the exact formatter, linter, type-checker, or test commands → the area's `conventions.md` / `verification.md`

## Core Principles

### 1. Readability Counts

Python prioritizes readability. Code should be obvious and easy to understand.

```python
# Good: clear and readable
def get_active_users(users: list[User]) -> list[User]:
    """Return only the active users from the provided list."""
    return [user for user in users if user.is_active]


# Bad: clever but confusing
def get_active_users(u):
    return [x for x in u if x.a]
```

### 2. Explicit is Better than Implicit

Avoid magic; be clear about what the code does.

```python
# Good: explicit configuration
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Bad: hidden side effects
import some_module
some_module.setup()  # What does this do?
```

### 3. EAFP - Easier to Ask Forgiveness than Permission

Python prefers exception handling over condition checks.

```python
# Good: EAFP style
def get_value(dictionary: dict, key: str) -> Any:
    try:
        return dictionary[key]
    except KeyError:
        return default_value

# Bad: LBYL (Look Before You Leap) style
def get_value(dictionary: dict, key: str) -> Any:
    if key in dictionary:
        return dictionary[key]
    else:
        return default_value
```

## Type Hints

### Basic Type Annotations

```python
from typing import Optional, List, Dict, Any

def process_user(
    user_id: str,
    data: Dict[str, Any],
    active: bool = True
) -> Optional[User]:
    """Process a user and return the updated User or None."""
    if not active:
        return None
    return User(user_id, data)
```

### Modern Type Hints (Python 3.9+)

```python
# Python 3.9+ - use built-in types
def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Python 3.8 and earlier - use the typing module
from typing import List, Dict

def process_items(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

### Type Aliases and TypeVar

```python
from typing import TypeVar, Union

# Type alias for complex types
JSON = Union[dict[str, Any], list[Any], str, int, float, bool, None]

def parse_json(data: str) -> JSON:
    return json.loads(data)

# Generic types
T = TypeVar('T')

def first(items: list[T]) -> T | None:
    """Return the first element, or None if the list is empty."""
    return items[0] if items else None
```

### Protocol-Based Duck Typing

```python
from typing import Protocol

class Renderable(Protocol):
    def render(self) -> str:
        """Render the object to a string."""

def render_all(items: list[Renderable]) -> str:
    """Render every element implementing the Renderable protocol."""
    return "\n".join(item.render() for item in items)
```

## Error Handling Patterns

### Catching Specific Exceptions

```python
# Good: catch specific exceptions
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in config: {path}") from e

# Bad: bare except
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except:
        return None  # Silent failure!
```

### Exception Chaining

```python
def process_data(data: str) -> Result:
    try:
        parsed = json.loads(data)
    except json.JSONDecodeError as e:
        # Chain exceptions to preserve the traceback
        raise ValueError(f"Failed to parse data: {data}") from e
```

### Custom Exception Hierarchy

```python
class AppError(Exception):
    """Base exception for all application errors."""
    pass

class ValidationError(AppError):
    """Raised when input validation fails."""
    pass

class NotFoundError(AppError):
    """Raised when a requested resource is not found."""
    pass

# Usage
def get_user(user_id: str) -> User:
    user = db.find_user(user_id)
    if not user:
        raise NotFoundError(f"User not found: {user_id}")
    return user
```

## Context Managers

### Resource Management

```python
# Good: use context managers
def process_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()

# Bad: manual resource management
def process_file(path: str) -> str:
    f = open(path, 'r')
    try:
        return f.read()
    finally:
        f.close()
```

### Custom Context Managers

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    """Context manager to time a block of code."""
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{name} took {elapsed:.4f} seconds")

# Usage
with timer("data processing"):
    process_large_dataset()
```

### Context Manager Classes

```python
class DatabaseTransaction:
    def __init__(self, connection):
        self.connection = connection

    def __enter__(self):
        self.connection.begin_transaction()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()
        return False  # Do not suppress exceptions

# Usage
with DatabaseTransaction(conn):
    user = conn.create_user(user_data)
    conn.create_profile(user.id, profile_data)
```

## Comprehensions and Generators

### List Comprehensions

```python
# Good: list comprehension for simple transformations
names = [user.name for user in users if user.is_active]

# Bad: manual loop
names = []
for user in users:
    if user.is_active:
        names.append(user.name)

# Complex comprehensions should be expanded
# Bad: too complex
result = [x * 2 for x in items if x > 0 if x % 2 == 0]

# Good: use a generator function
def filter_and_transform(items: Iterable[int]) -> list[int]:
    result = []
    for x in items:
        if x > 0 and x % 2 == 0:
            result.append(x * 2)
    return result
```

### Generator Expressions

```python
# Good: generator for lazy evaluation
total = sum(x * x for x in range(1_000_000))

# Bad: creates a large intermediate list
total = sum([x * x for x in range(1_000_000)])
```

### Generator Functions

```python
def read_large_file(path: str) -> Iterator[str]:
    """Read a large file line by line."""
    with open(path) as f:
        for line in f:
            yield line.strip()

# Usage
for line in read_large_file("huge.txt"):
    process(line)
```

## Data Classes and Named Tuples

### Data Classes

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    """User entity with automatic __init__, __repr__, and __eq__."""
    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True

# Usage
user = User(
    id="123",
    name="Alice",
    email="alice@example.com"
)
```

### Data Classes with Validation

```python
@dataclass
class User:
    email: str
    age: int

    def __post_init__(self):
        # Validate email format
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")
        # Validate age range
        if self.age < 0 or self.age > 150:
            raise ValueError(f"Invalid age: {self.age}")
```

### Named Tuples

```python
from typing import NamedTuple

class Point(NamedTuple):
    """Immutable 2D point."""
    x: float
    y: float

    def distance(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# Usage
p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance(p2))  # 5.0
```

## Decorators

### Function Decorators

```python
import functools
import time

def timer(func: Callable) -> Callable:
    """Decorator to measure a function's execution time."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)

# slow_function() prints: slow_function took 1.0012s
```

### Parameterized Decorators

```python
def repeat(times: int):
    """Decorator to repeat a function multiple times."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(times=3)
def greet(name: str) -> str:
    return f"Hello, {name}!"

# greet("Alice") returns ["Hello, Alice!", "Hello, Alice!", "Hello, Alice!"]
```

### Class-Based Decorators

```python
class CountCalls:
    """Decorator that counts how many times a function is called."""
    def __init__(self, func: Callable):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} has been called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def process():
    pass

# Each call to process() prints the call count
```

## Concurrency Patterns

### Threading for I/O-Bound Tasks

```python
import concurrent.futures

def fetch_url(url: str) -> str:
    """Fetch a URL (I/O-bound operation)."""
    import urllib.request
    with urllib.request.urlopen(url) as response:
        return response.read().decode()

def fetch_all_urls(urls: list[str]) -> dict[str, str]:
    """Fetch multiple URLs concurrently using threads."""
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {executor.submit(fetch_url, url): url for url in urls}
        results = {}
        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                results[url] = future.result()
            except Exception as e:
                results[url] = f"Error: {e}"
    return results
```

### Multiprocessing for CPU-Bound Tasks

```python
def process_data(data: list[int]) -> int:
    """CPU-intensive computation."""
    return sum(x ** 2 for x in data)

def process_all(datasets: list[list[int]]) -> list[int]:
    """Process multiple datasets using multiple processes."""
    with concurrent.futures.ProcessPoolExecutor() as executor:
        results = list(executor.map(process_data, datasets))
    return results
```

### Async/Await for Concurrent I/O

```python
import asyncio

async def fetch_async(url: str) -> str:
    """Fetch a URL asynchronously."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def fetch_all(urls: list[str]) -> dict[str, str]:
    """Fetch multiple URLs concurrently."""
    tasks = [fetch_async(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(urls, results))
```

> Only reach for these when the workload warrants it. For async web-framework integrations, see `fastapi`.

## Package Organization

### Standard Project Layout

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── main.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_models.py
├── pyproject.toml
├── README.md
└── .gitignore
```

### Import Conventions

```python
# Good: import order - stdlib, third-party, local
import os
import sys
from pathlib import Path

import requests

from mypackage.models import User
from mypackage.utils import format_name

# Sort imports automatically with a tool configured in the area's conventions.
```

### __init__.py for Package Exports

```python
# mypackage/__init__.py
"""mypackage - an example Python package."""

__version__ = "1.0.0"

# Export main classes/functions at the package level
from mypackage.models import User, Post
from mypackage.utils import format_name

__all__ = ["User", "Post", "format_name"]
```

## Memory and Performance

### __slots__ for Memory Efficiency

```python
# Bad: a regular class uses __dict__ (more memory)
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

# Good: __slots__ reduces memory usage
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
```

### Generators for Large Data

```python
# Bad: returns the whole list in memory
def read_lines(path: str) -> list[str]:
    with open(path) as f:
        return [line.strip() for line in f]

# Good: yields lines one at a time
def read_lines(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()
```

### Avoid String Concatenation in Loops

```python
# Bad: O(n²) due to string immutability
result = ""
for item in items:
    result += str(item)

# Good: O(n) using join
result = "".join(str(item) for item in items)
```

## Style (PEP 8)

PEP 8 is the reference for Python style. Full guide: `references/python_style_guide.md`. A project's own conventions take precedence on conflict. The essentials, with examples:

- **Consistency hierarchy**: guide < project < module < function. Code is read more than written; when a rule makes code worse, follow your judgment instead — but never break backwards compatibility for style.
- Code is UTF-8 without an encoding declaration; identifiers are ASCII; comments and names are English.

### Layout

- 4 spaces per indent; spaces, never tabs (never mix). Max 79 chars per line (docstrings/comments 72); a team may agree up to 99 for code. Prefer implicit continuation inside parentheses over backslashes.
- Compound statements (`if x: do()`, `a(); b()`) are discouraged; never put a multi-clause statement on one line.

```python
# Good: hanging indent adds a level
foo = long_function_name(
    var_one, var_two,
    var_three, var_four)

# Bad: arguments on the first line without vertical alignment
foo = long_function_name(var_one, var_two,
    var_three, var_four)
```

- Break **before** binary operators (easier to match operands):

```python
income = (gross_wages
          + taxable_interest
          - ira_deduction)
```

- Two blank lines around top-level functions/classes; one around methods. Use blank lines sparingly inside functions to separate logical sections.
- Module dunders (`__all__`, `__version__`, …) go after the module docstring and before imports — only `from __future__` imports may precede them.
- Trailing commas: mandatory for a 1-tuple (`FILES = ('setup.cfg',)`), never on the same line as the closing delimiter; otherwise use the one-per-line + trailing comma pattern when the collection is expected to grow.

### Imports

- One import per line; top of file; groups separated by a blank line: stdlib → third-party → local.
- Prefer absolute imports; relative imports are fine for complex layouts. Avoid wildcard imports.

```python
import os
import sys

import requests

from mypackage.models import User
```

### Strings, whitespace & annotations

```python
# Good
spam(ham[1], {eggs: 2})
foo = (0,)
ham[lower+offset : upper+offset]
x = 1

# Bad
spam( ham[ 1 ], { eggs: 2 } )
bar = (0, )
ham[lower + offset:upper + offset]
x             = 1
```

- Pick one quote character for ordinary strings and stick to it; triple-quoted strings always use `"""`. Use the other quote to avoid escapes.
- No trailing whitespace anywhere, and never rely on significant trailing whitespace in string literals.
- Binary operators get one space each side; around mixed priorities, space the lowest ones (`x = x*2 - 1`). Slices treat `:` like a binary operator (`ham[lower:upper]`, `ham[lower+offset : upper+offset]`, no space when a bound is omitted).
- No spaces around `=` for keyword args / unannotated defaults; **do** use spaces when a default is combined with an annotation:

```python
def complex(real, imag=0.0): ...          # no spaces
def munge(input: AnyStr = None): ...      # spaces (annotated)
```

- Annotations: no space before `:`, one space after; spaces around `->`. Variables follow the same rule (`code: int`, `label: str = '<unknown>'`).

### Naming

- `snake_case` for functions, variables, modules; `CapWords` for classes; `UPPER_CASE` for module-level constants.
- Short all-lowercase module names (underscores only if they help); underscores in package names are discouraged.
- Exceptions: `CapWords` + `Error` suffix (e.g. `NotFoundError`). Derive from `Exception`, never `BaseException`; non-error signal exceptions need no suffix.
- Type variables: short `CapWords` (`T`, `AnyStr`), suffix `_co`/`_contra` for covariance/contravariance.
- `self` / `cls` for method first arguments; a single trailing underscore to dodge keywords (`class_`); `_single_leading_underscore` for internal; `__double_leading` invokes name mangling (use it to avoid subclass collisions, not as "private").
- Avoid `l`, `O`, `I` as single-char names.

### Comments & docstrings

- Keep comments current; a stale comment is worse than none. Write complete, capitalized English sentences ending in a period.
- Block comments indent with the code, each line starting `# `; separate paragraphs with a bare `#`.
- Inline comments: at least two spaces before `#`, then one space — only when they add information (`# Compensate for border`, not `# Increment x`).
- Write docstrings for all public modules, functions, classes, methods; give non-public ones a comment after `def` instead.
- Triple-quoted strings use double quotes; a one-line docstring keeps the closing `"""` on the same line; a multi-line one puts it alone:

```python
def parrot():
    """Return an ex-parrot."""

def frobnicate():
    """Return a foobang.

    Optional plotz says to frobnicate the bizbaz first.
    """
```

### Designing classes & public interfaces

- Decide explicitly what is public (no leading underscore) vs non-public (`_name`) vs subclass API. When in doubt, start non-public.
- Expose plain data attributes directly; add properties later if behavior is needed. Avoid properties for expensive work — attribute syntax implies cheap access.
- Documented interfaces are public; everything else is internal. Declare the public API with `__all__` (empty list = no public API) and prefix internal names with `_`. Imported names are implementation details.

### Programming recommendations

```python
# Good
if foo is not None: ...
if isinstance(obj, int): ...
if foo.startswith('bar'): ...
if not seq: ...
result = "".join(parts)
def f(x): return 2*x

# Bad
if not foo is None: ...
if type(obj) is type(1): ...
if foo[:3] == 'bar': ...
if len(seq): ...
result = ""  # then result += part in a loop
f = lambda x: 2*x
```

- Catch specific exceptions; `except Exception:` when you mean all program errors (bare `except:` also catches `KeyboardInterrupt`/`SystemExit`); chain with `raise X from Y`, or `raise X from None` only after transferring the original details.
- Keep the `try` block minimal; use `else` for code that should run only on success; prefer `with`/`try...finally` for cleanup.
- Only invoke a context manager directly when it just acquires/releases a resource — name the operation otherwise (`with conn.begin_transaction():`).
- Return consistently: all returns return a value, or none do; write an explicit `return None` where a value-returning function falls through.
- Use `functools.total_ordering()` when implementing rich comparisons instead of hand-writing all six methods.
- Never `return`/`break`/`continue` out of a `finally` block — it swallows the active exception.
- Prefer `''.join()` over `+=` in performance-sensitive loops; don't rely on CPython-only optimizations.
- Don't compare booleans with `== True` or `is True` — test truthiness.
- OS errors: catch the Python 3.3+ exception hierarchy (`FileNotFoundError`, …), not `errno` integers.
- Use `# type: ignore` at file top only when intentionally opting out of type checking.

## Tooling

Formatting, linting, type-checking, and test-runner choices are **not** defined here — they belong to the area's `conventions.md` (e.g. line length, import ordering) and `verification.md` (the exact commands to run). Follow what the area prescribes; do not introduce a second toolchain.

## Quick Reference: Python Patterns

| Pattern | Description |
|--------|-------------|
| EAFP | Easier to Ask Forgiveness than Permission |
| Context managers | Use `with` for resource management |
| List comprehensions | For simple transformations |
| Generators | For lazy evaluation and large datasets |
| Type hints | Annotate function signatures |
| Dataclasses | For data containers with auto-generated methods |
| `__slots__` | For memory optimization |
| f-strings | For string formatting (Python 3.6+) |
| `pathlib.Path` | For path operations (Python 3.4+) |
| `enumerate` | For index-element pairs in loops |

## Anti-Patterns to Avoid

```python
# Bad: mutable default arguments
def append_to(item, items=[]):
    items.append(item)
    return items

# Good: use None and create a new list
def append_to(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# Bad: checking type with type()
if type(obj) == list:
    process(obj)

# Good: use isinstance
if isinstance(obj, list):
    process(obj)

# Bad: comparing to None with ==
if value == None:
    process()

# Good: use is
if value is None:
    process()

# Bad: from module import *
from os.path import *

# Good: explicit imports
from os.path import join, exists

# Bad: bare except
try:
    risky_operation()
except:
    pass

# Good: specific exception
try:
    risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
```

__Remember__: Python code should be readable, explicit, and follow the principle of least surprise. When in doubt, prioritize clarity over cleverness.

## References

- `references/python_style_guide.md` — *PEP 8 – Style Guide for Python Code* (van Rossum, Warsaw, Coghlan) — code layout, imports, whitespace, comments/docstrings, naming, annotations, and programming recommendations.
- *PEP 257 – Docstring Conventions* (Goodger, van Rossum) — docstring content and closing-quote placement rules referenced above.
