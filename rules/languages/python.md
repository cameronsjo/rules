---
globs: ["**/*.py"]
alwaysApply: false
---

# Python Standards

**Runtime:** Python 3.11+ | **Tooling:** ruff, black, isort, mypy/pyright

## Patterns

```python
# Type hints required (minimize Any)
def process_user(user_id: str, options: dict[str, Any] | None = None) -> User: ...

# Dataclasses/Pydantic for data structures
@dataclass
class User:
    id: str
    email: str
    created_at: datetime = field(default_factory=datetime.utcnow)

# Context managers for resources
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()

# Lazy logging (not f-strings)
logger.debug("Processing user %s", user_id)
```

## Anti-patterns

```python
# ❌ hasattr, f-string logging, magic strings
if hasattr(obj, "status"):
    return obj.status == "active"
logger.debug(f"Processing user {user_id}")

# ✅ getattr with default, lazy logging, constants
STATUS_ACTIVE = "active"
status = getattr(obj, "status", None)
logger.debug("Processing user %s", user_id)
```

## Testing

- pytest with fixtures and parametrize
- 90%+ coverage target
- Use `pytest-asyncio` for async tests
