---
paths:
  - "**/*.py"
  - "**/pyproject.toml"
  - "**/requirements*.txt"
  - "**/setup.py"
  - "**/setup.cfg"
  - "**/.python-version"
---

# Python Standards

- **Runtime**: Python 3.13+ (3.14 released Oct 2025, 3.15 Oct 2026)
- **Version Management**: mise (replaces pyenv)
- **Package Management**: uv (10x+ faster than pip, written in Rust)
- **Linting/Formatting**: Ruff (10-100x faster, replaces black, isort, flake8)
- **Type Checking**: ty (Astral, 10-60x faster than mypy/Pyright) or Pyright
- **Validation**: Pydantic v2
- **Testing**: pytest + pytest-asyncio
- **AI/LLM**: pydanticai (type-safe LLM interactions)
- **Observability**: OpenTelemetry + structlog

## Core Requirements

- **MUST** use uv for package management (not pip)
- **MUST** include type hints - untyped code is technical debt
- **MUST** use Ruff for linting and formatting
- **MUST** use lazy logging: `logger.debug("val=%s", val)` not f-strings
- **MUST** use Pydantic for validation (config, API bodies, schemas)
- **MUST** use `TaskGroups` (3.11+) for structured concurrency - no dangling futures
- **MUST** explicitly return `T | None` rather than implicit `None`
- **MUST NOT** use magic strings/numbers
- **MUST NOT** use `*args`/`**kwargs` unless necessary - destroys type safety and autocomplete
- **MUST NOT** use mutable defaults in function args (`def foo(items=[])`)
- **SHOULD** use mise for Python version management
- **SHOULD** use type guards and `typing.Self` for OOP patterns
- **SHOULD** avoid `hasattr()` - use `getattr()` with default or try/except
- **SHOULD NOT** use pandas in production APIs (heavy memory) - prefer Polars or dataclasses

## Project Setup

```bash
# Initialize with uv
uv init myproject
cd myproject

# Add dependencies
uv add fastapi pydantic structlog

# Add dev dependencies
uv add --dev pytest ruff pyright

# Run commands
uv run python main.py
uv run pytest
uv run ruff check .
```

## pyproject.toml

```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []

[tool.uv]
dev-dependencies = [
    "pytest>=8.0",
    "ruff>=0.8",
    "pyright>=1.1",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "ASYNC"]

[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "strict"
```

## Python 3.13+ Features

```python
# Type parameter syntax (3.12+) - no more TypeVar
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

# Lazy annotations (3.14) - no more forward reference issues
# Annotations evaluated only when explicitly requested
class Node:
    def __init__(self, child: Node | None = None):  # Just works now
        self.child = child

# Free-threaded mode (3.13+, improved 3.14)
# 5-10% single-threaded penalty, true parallelism
# Build with --disable-gil or use python3.14t

# Exception groups with TaskGroups (3.11+)
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(task1())
        tg.create_task(task2())
except* ValueError as eg:
    for exc in eg.exceptions:
        handle(exc)
```

## Modern Patterns

```python
# Pydantic v2 for validation
from pydantic import BaseModel, Field

class User(BaseModel):
    model_config = {"strict": True}

    id: str = Field(..., pattern=r"^[0-9A-Z]{26}$")  # ULID
    email: str = Field(..., pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$")
    role: Literal["admin", "user"]

# Structured logging with structlog
import structlog

logger = structlog.get_logger()
logger.info("user_created", user_id=user.id, email=user.email)

# Async context managers
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_connection():
    conn = await create_connection()
    try:
        yield conn
    finally:
        await conn.close()

# Result pattern (no exceptions for expected failures)
from dataclasses import dataclass

@dataclass
class Ok[T]:
    value: T

@dataclass
class Err[E]:
    error: E

type Result[T, E] = Ok[T] | Err[E]
```

## Anti-patterns

```python
# ❌ Bad
import os
api_key = os.environ["API_KEY"]  # Crashes if missing
logger.debug(f"Processing {user_id}")  # Eager evaluation
data: Any = response.json()

# ✅ Good
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_key: str

settings = Settings()  # Validates at startup
logger.debug("Processing %s", user_id)  # Lazy evaluation
data = ResponseModel.model_validate(response.json())
```

## FastAPI Patterns

```python
from fastapi import FastAPI, HTTPException, Depends
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await init_db()
    yield
    # Shutdown
    await close_db()

app = FastAPI(lifespan=lifespan)

@app.get("/users/{user_id}")
async def get_user(user_id: str) -> User:
    user = await db.get_user(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user
```
