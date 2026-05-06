---
name: grandpython
description: "Python backend coding guidelines and conventions. Use when: writing Python backend code, creating services/routers/models/schemas, reviewing Python code style, organizing imports, naming functions or variables, writing docstrings or comments, structuring modules. Enforces GrandGale backend styleguide and FastAPI conventions."
argument-hint: "Describe what you're building or the file you need help with (e.g. 'a new contacts service', 'a route for campaign CRUD')"
---

# Python Backend Coding Guidelines

Coding standards for all Python backend code in `backend/app/`. Based on GrandGale Technologies styleguides with project-specific amendments.

## When to Use

- Writing or reviewing any Python file in `backend/app/`
- Creating new services, routers, models, schemas, CRUD classes, or formatters
- Organizing imports or resolving naming/aliasing questions
- Adding environment variables or configuring settings
- Writing docstrings, comments, or inline type annotations

## Imports

Three sections separated by blank lines: stdlib → third-party → codebase.

Key rules:
- **No `from __future__ import annotations`** unless strictly necessary for a forward ref or circular import
- **No function-level imports** — always import at the top of the file
- **Named imports (no alias)** for same-module usage: `from app.businesses import models, schemas`
- **Namespaced imports** for cross-module: `from app.auth import service as auth_service`
- **Schema aliases** use the `{module initial}{submodule initial}_schemas` pattern: `ba_schemas`, `bc_schemas`
- **Direct imports** for exceptions, constants, utilities: `from app.exceptions import NotFound`
- **`TYPE_CHECKING` guard** for type-hint-only imports that would cause circular dependencies

[Full import conventions with all examples and alias tables](./references/imports.md)

## Naming Conventions

### Variables and Functions

- Use `snake_case` for all function and variable names
- Be descriptive and specific

### Keyword Arguments

Always use keyword arguments when calling functions. Never use positional arguments.

**Yes:**
```python
user = await crud.get(id=user_id)
business = models.Business(name=name, phone=phone)
raise NotFound(msg="Business not found")
```

**No:**
```python
user = await crud.get(user_id)         # wrong: positional
business = models.Business(name, phone) # wrong: positional
raise NotFound("Business not found")   # wrong: positional
```

### Classes

- Use `PascalCase` for all classes (models, schemas, exceptions)

### Constants and Globals

Place constants and global instances at the top of the file after imports:

```python
# Globals
settings = get_settings()
router = APIRouter()

# Constants
MAX_RETRY_COUNT = 3
DEFAULT_PAGE_SIZE = 20
```

## Module Structure

Each module contains: `apis.py`, `crud.py`, `formatters.py`, `models.py`, `selectors.py` (optional), `service.py`, `utils.py` (optional), `routes/`, and `schemas/`.

Key rule: `service.py` contains **only orchestration logic** — no raw DB access, no utility functions. All DB operations go through the CRUD class. Read-only lookups with optional exception raising belong in `selectors.py`. Functions that don't orchestrate belong in `utils.py`.

[Full module layout, service vs utility distinction, and directory diagram](./references/module-structure.md)

## CRUD Classes

All DB operations go through a CRUD class. Naming: `{Model}CRUD` (e.g., `BusinessCRUD`, `UserCRUD`) — model name first.

Key rules:
- Each module has `crud.py` with a class extending `CRUDBase` from `app/db/crud.py`
- `__init__` hard-codes the model — callers only pass `db`
- Instantiate fresh inside each service function; never reuse across calls
- Never call `db.add()`, `db.commit()`, `db.execute()`, or `db.delete()` in `service.py` or routes
- Pass model objects to service functions when already in scope — avoid re-fetching by ID

[Full CRUD reference with CRUDBase definition, module classes, service function patterns, and pass-objects rule](./references/module-structure.md)

## Selectors

Each module has `selectors.py` with **read-only lookup functions** that wrap the CRUD class and optionally raise domain exceptions. They are the single authoritative place to look up an object and raise a 404-style exception.

Key rules:
- **Naming:** `get_{entity}_by_{field}` for single objects, `get_all_{entity}_{qualifier}` for lists
- **Signature:** always accepts `db: AsyncSession`; single-object selectors accept `raise_exc: bool = True` as the last parameter
- **CRUD only** — never call `db.execute()` directly; complex queries go as custom methods on the CRUD class
- **Read-only** — never write to the database
- **No auth deps** — `get_current_user` belongs in `dependencies.py`, not selectors
- **No business logic** — orchestration belongs in `service.py`
- Called from routes (to resolve path params) and service functions (as precondition checks)
- Wrap calls with `raise_exc=True` (default) in `cast(Model, await selectors.get_...())` for type narrowing and IDE autocomplete
- Cross-module: import as `from app.{module} import selectors as {alias}_selectors`

[Full selectors reference with signature pattern, multi-tenant scoped lookups, list selectors, cross-module usage, and all anti-patterns](./references/selectors.md)

## Formatters

Each module has `formatters.py` with pure functions that convert model instances to plain dicts. Always return `dict` (never Pydantic instances). Called **only in routes** — never in service functions. Routes wrap formatter output in `{"data": ...}`.

Naming: `fmt_{resource}` for single objects, `fmt_{resource}_list` for collections.

[Full formatter reference with structure, nested formatters, usage patterns, and all rules](./references/formatters.md)

## Routes

Route functions must start with `route_` prefix. Use multi-line decorator form with `summary`, `response_description`, `status_code`, and `response_model` as explicit keyword arguments. Never use return type annotations on the function — use `response_model` instead.

| Action | Pattern | Example |
|--------|---------|---------|
| List | `route_{resource}_list` | `route_contact_list` |
| Details | `route_{resource}_details` | `route_business_details` |
| Create | `route_{resource}_create` | `route_campaign_create` |
| Update | `route_{resource}_edit` | `route_contact_edit` |
| Delete | `route_{resource}_delete` | `route_campaign_delete` |
| Auth | `route_{module}_{action}` | `route_auth_login` |

[Full route reference with decorator structure, aggregation (apis.py), prefix patterns, and full examples](./references/routes.md)

## Dependency Annotations

FastAPI `Annotated` dependency shortcuts live in a module's `annotations.py` file — never in `dependencies.py`. `dependencies.py` contains the dependency function itself; `annotations.py` imports it and wraps it in `Annotated`.

```
app/
├── auth/
│   ├── dependencies.py   # get_current_user() function
│   └── annotations.py    # CurrentUserDep = Annotated[User, Depends(get_current_user)]
├── external/
│   └── redis/
│       ├── client.py     # get_redis() function
│       └── annotations.py  # RedisDep = Annotated[InternalRedisClient, Depends(get_redis)]
└── dependencies.py       # DbSessionDep = Annotated[AsyncSession, Depends(get_db)]
```

`app/dependencies.py` is the exception — it is the top-level home for cross-cutting session dependencies like `DbSessionDep`.

### Usage in Routes

Import the `Dep` alias from `annotations.py` directly:

```python
from app.auth.annotations import CurrentUserDep
from app.external.redis.annotations import RedisDep
from app.dependencies import DbSessionDep

async def route_campaign_list(db: DbSessionDep, current_user: CurrentUserDep):
    ...
```

## Comments

### `# Check:` Comments

Use `# Check: <description>` to mark validation checks, business rule validations, and conditional logic that raises exceptions or affects critical flow.

- Use lowercase descriptions
- Use present tense
- Be concise (3-10 words)
- Place directly before the check

```python
# Check: business exists
business = await service.get_business(business_id, db)
if not business:
    raise NotFound("Business not found")

# Check: user owns this business
if business.user_id != current_user.id:
    raise Forbidden("Access denied")

# Check: user has active subscription
if not current_user.subscription_active:
    raise BadRequest("Active subscription required")
```

**When to use:**
- Validation checks that raise exceptions
- Business logic validations
- Precondition checks (data exists, permissions)
- Error handling for external service responses

**When NOT to use:**
- Simple control flow (`if/else` routing)
- Data transformations
- Trivial optional value handling
- Loop control
- Debug conditionals

### `# NOTE:` Comments

Use `# NOTE:` to draw attention to important context:

```python
# NOTE: inactive users are filtered out by default
users = await service.list_users(db, active_only=True)
```

## Docstrings

Always add docstrings to **every function, class, Pydantic schema, and enum**. This includes route functions, service functions, CRUD classes, formatters, utility functions, exception classes, and enum classes. Do not add docstrings to modules, `__init__` methods, or standalone scripts.

Use imperative-style docstrings. Always use the multi-line format — text must start on a new line after the opening `"""` and the closing `"""` must be on its own line. Never use inline `"""Text"""` format.

**Yes:**
```python
class BusinessStatus(str, enum.Enum):
    """
    Possible lifecycle states of a business
    """

    active = "active"
    suspended = "suspended"


class BusinessCRUD(CRUDBase[models.Business]):
    """
    CRUD operations for the Business model
    """


class BusinessCreate(BaseModel):
    """
    Payload for creating a new business
    """

    business_name: str = Field(description="The display name of the business")
    phone: str = Field(description="Primary contact phone number")


def get_user_by_id(user_id: uuid.UUID, db: Session):
    """
    Get user by ID

    Args:
        user_id (uuid.UUID): The user's ID
        db (Session): The database session

    Returns:
        models.User | None: The user object

    Raises:
        NotFound
    """


async def route_auth_me(current_user: CurrentUserDep):
    """
    Get current user details
    """
```

**No:**
```python
"""This module handles user operations."""  # wrong: module-level docstring

class BusinessStatus(str, enum.Enum):  # wrong: missing docstring
    active = "active"

class BusinessCreate(BaseModel):  # wrong: missing docstring
    business_name: str = Field(description="The display name of the business")

def get_user_by_id(user_id: uuid.UUID, db: Session):
    """Gets user by their ID"""  # wrong: inline format; descriptive, not imperative

def encrypt(plaintext: str) -> str:
    """Encrypt a plaintext string."""  # wrong: inline format

class UserService:
    def __init__(self):
        """Initialize the service."""  # wrong: don't docstring __init__
```

## Exceptions

Name HTTP exceptions after their reason phrase in PascalCase:

```python
class NotFound(CustomHTTPException):
    def __init__(self, msg: str = "Not Found", loc: list | None = None):
        super().__init__(msg, status_code=404, loc=loc)

class BadRequest(CustomHTTPException):
    def __init__(self, msg: str = "Bad Request", loc: list | None = None):
        super().__init__(msg, status_code=400, loc=loc)

class Unauthorized(CustomHTTPException):
    def __init__(self, msg: str = "Unauthorized", loc: list | None = None):
        super().__init__(msg, status_code=401, loc=loc)

class Forbidden(CustomHTTPException):
    def __init__(self, msg: str = "Forbidden", loc: list | None = None):
        super().__init__(msg, status_code=403, loc=loc)
```

Usage:
```python
if not obj:
    raise NotFound(msg="Business not found")
```

## Type Hints

- Use modern Python type hints (`str | None` instead of `Optional[str]`)
- Use `Annotated` for FastAPI parameter and dependency declarations
- Use `Mapped` and `mapped_column` for SQLAlchemy model columns
- Use `Literal[...]` for fields or parameters where only specific values are valid

### SQLAlchemy Models

Always declare columns with `Mapped[T] = mapped_column(...)` (SQLAlchemy 2.0 style). This is what makes attribute assignment (`org.credit_limit = 5000`) type-safe and avoids the need for `setattr`/`cast` workarounds at call sites.

**Yes:**
```python
from datetime import datetime
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship


class Organization(DBBase):
    """
    Database model for organizations
    """

    __tablename__ = "organizations"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(255))
    email: Mapped[str] = mapped_column(String(255))
    credit_limit: Mapped[float] = mapped_column(Numeric(12, 2), default=0)
    is_active: Mapped[bool] = mapped_column(default=True)
    parent_id: Mapped[int | None] = mapped_column(ForeignKey("organizations.id"))
    created_at: Mapped[datetime] = mapped_column(default=datetime.now)
```

**No:**
```python
class Organization(DBBase):
    id = Column(Integer, primary_key=True, autoincrement=True)         # wrong: legacy form
    name = Column(String(255), nullable=False)                         # wrong: untyped
    credit_limit = Column(Numeric(12, 2), default=0, nullable=False)   # wrong: untyped
```

Nullability is inferred from the type: `Mapped[int]` → `NOT NULL`, `Mapped[int | None]` → `NULL`. Do not pass `nullable=...` to `mapped_column` — let the type drive it.

### `setattr` on Model Instances

**Never use `setattr` with a constant attribute name on a model instance.** With `Mapped[T]` columns, attribute assignment is fully typed and `setattr` only obscures intent (and triggers `B010`).

**Yes:**
```python
org.credit_limit = new_credit_limit
org.is_credit_enabled = True
```

**No:**
```python
setattr(org, "credit_limit", new_credit_limit)   # wrong: triggers B010
setattr(org, "is_credit_enabled", True)          # wrong: same
```

`setattr` is only acceptable when the attribute name is a runtime value — e.g. inside a generic `crud.update(obj, data)` loop:

```python
for field, value in data.items():
    setattr(obj, field, value)   # ok: dynamic field name
```

### `Literal[]` for Fixed Values

Use `Literal[...]` whenever a field or parameter is constrained to a known set of values — in schemas, function signatures, and formatters.

**Yes:**
```python
from typing import Literal

class TokenResponse(BaseModel):
    access_token: str = Field(description="JWT access token")
    token_type: Literal["bearer"] = Field(default="bearer", description="Token type")

class NotificationCreate(BaseModel):
    channel: Literal["sms", "whatsapp", "email"] = Field(description="Delivery channel")

def fmt_token_response(token: str, token_type: Literal["bearer"] = "bearer") -> dict:
    return {"access_token": token, "token_type": token_type}
```

**No:**
```python
class TokenResponse(BaseModel):
    token_type: str = Field(default="bearer", description="Token type")  # wrong: str is too wide

class NotificationCreate(BaseModel):
    channel: str = Field(description="Delivery channel")  # wrong: use Literal for fixed options
```

## Pydantic Schemas

Schemas live in a `schemas/` package with exactly 5 files: `__init__.py`, `base.py`, `create.py`, `edit.py`, `response.py`.

| Schema kind | File | Examples |
|-------------|------|---------|
| DB read representation | `base.py` | `UserRead`, `BusinessRead` |
| Create/POST request body | `create.py` | `BusinessCreate`, `SignupRequest` |
| Edit/PUT/PATCH request body | `edit.py` | `BusinessUpdate`, `ContactEdit` |
| Composite API response | `response.py` | `TokenResponse`, `SignupResponse` |

Key rules:
- `__init__.py` re-exports all public schemas for both import styles
- `base.py`/`create.py`/`edit.py` schemas inherit only from `BaseModel` — no custom base classes
- All `response.py` schemas **must** inherit from `Response` or `PaginatedResponse` (from `app.common.schemas`)
- Every field must use `Field(description="...")` — bare annotations and bare `Field()` are wrong
- `@field_validator` methods require a multi-line docstring with `Tasks:` bullet format

[Full schema reference with layout, inheritance rules, Response/PaginatedResponse, field validators, and examples](./references/schemas.md)

## Logfire Tracing

Logfire is the mandatory observability layer. Use `@logfire.instrument()` on every service function, CRUD method, and background task. Use `{key}` placeholders with keyword args — never f-strings.

```python
@logfire.instrument("get_business {business_id}")
async def get_business(business_id: uuid.UUID, db: AsyncSession) -> models.Business:
    ...

logfire.info("Created business {business_id}", business_id=business.id)
```

Never use `print()`, `logging.*()`, or swallow exceptions silently. Always re-raise after `logfire.exception()`.

For full setup instructions, span naming conventions, log levels, extras, and exception logging patterns — invoke the [logfire-instrumentation skill](../logfire-instrumentation/SKILL.md).

## Indentation

Use 4 spaces, never tabs.

## Environment Variables

- Use Pydantic `BaseSettings` with `SettingsConfigDict` for configuration
- **Whenever a new env var is added**, update all of the following in order:
  1. `backend/app/config.py` — add to `Settings` with the correct type and validation (e.g. `Field(min_length=32)`), grouped under the relevant comment block
  2. `backend/.env.example` — add with a safe placeholder value (e.g. `MY_VAR=change-me`), mirroring the group from `config.py`
  3. `backend/.env.test` — add with a plausible fake value for CI (e.g. matching any regex pattern, or a clearly fake sentinel like `ci-test-my-var`)
  4. `docs/deployment.md` — add to the "Required Environment Variables (Railway)" section with a one-line description

## Gotchas

- **Always update all sample env files when adding a new env var.** Adding a field to `Settings` in `config.py` without updating `backend/.env.example`, `backend/.env.test`, and `docs/deployment.md` breaks local dev setup for teammates and CI runs. Follow the 4-step checklist in [Environment Variables](#environment-variables) every time.
