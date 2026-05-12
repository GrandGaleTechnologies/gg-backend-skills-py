# Module Structure & Service Layer

## Directory Layout

Each module follows a standard layout:

```
app/
├── {module}/
│   ├── __init__.py
│   ├── apis.py          # Aggregates all route files
│   ├── crud.py          # CRUD class for all DB operations
│   ├── formatters.py    # Serialization: model → response schema
│   ├── models.py
│   ├── selectors.py     # Read-only lookup functions (optional)
│   ├── services.py
│   ├── utils.py         # Module-scoped utility functions (optional)
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── base.py      # Base routes for the module
│   │   └── {feature}.py # Feature-specific routes (optional)
│   └── schemas/
│       ├── __init__.py  # Re-exports all public schemas
│       ├── base.py      # Read / shared schemas
│       ├── create.py    # Create & request schemas
│       ├── edit.py      # Edit / update schemas
│       └── response.py  # Composite response schemas
```

## File Responsibilities

| File | Responsibility |
|------|---------------|
| `crud.py` | All DB operations: `get`, `get_all`, `create`, `update`, `delete`, and custom query methods |
| `selectors.py` | Read-only lookup functions that wrap CRUD with optional `raise_exc`; called from routes and services |
| `services.py` | Orchestration: multi-step business logic, external API calls, coordinating CRUD and selectors |
| `utils.py` | Module-scoped helpers that don't fit the above (no DB access, no orchestration) |

## Service vs Utility Functions

`services.py` must only contain **service functions** (business logic that orchestrates operations, calls external APIs, or coordinates between subsystems). Direct database access must never appear in `services.py` — all DB operations must go through the module's **CRUD class**.

Any function that does not fit that definition is a **utility function** and must live in:

- `app/utils/` — if it is generic and reusable across modules (e.g., crypto helpers, string manipulation)
- `app/{module}/utils.py` — if it is specific to that module but not a service function

**Yes:**
```python
# crud.py — all DB operations
class BusinessCRUD(CRUDBase[models.Business]):
    pass

# services.py — orchestration only, delegates DB work to CRUD
async def create_business(data: schemas.BusinessCreate, db: AsyncSession) -> models.Business:
    """
    Create a new business
    """
    crud = BusinessCRUD(db=db)
    return await crud.create(data=data.model_dump())

# utils.py — module-specific utility functions
def build_business_display_name(name: str, suffix: str) -> str:
    """
    Build the formatted display name for a business
    """
    return f"{name.strip()} {suffix}".strip()
```

**No:**
```python
# services.py — wrong: direct DB access without CRUD
async def create_business(data: schemas.BusinessCreate, db: AsyncSession) -> models.Business:
    business = models.Business(**data.model_dump())  # wrong: direct model instantiation
    db.add(business)                                  # wrong: direct db.add
    await db.commit()                                 # wrong: direct db.commit
    return business

# services.py — wrong: selector query without CRUD
async def get_business(business_id: uuid.UUID, db: AsyncSession):
    return await db.scalar(select(models.Business).where(...))  # wrong: raw SQLAlchemy in service

# services.py — wrong: utility logic mixed in
def build_business_display_name(name: str, suffix: str) -> str:  # wrong: not a service function
    return f"{name.strip()} {suffix}".strip()
```

## CRUD Base Class

The base `CRUDBase` lives in `app/db/crud.py`:

```python
from typing import Generic, Type, TypeVar

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

T = TypeVar("T")


class CRUDBase(Generic[T]):
    """
    Base CRUD class with default Create, Read, Update, Delete operations
    """

    def __init__(self, model: Type[T], db: AsyncSession):
        """
        Initialize with the SQLAlchemy model class and an async DB session
        """
        self.model = model
        self.db = db

    async def create(self, *, data: dict) -> T:
        """
        Create and persist a new model instance
        """
        db_obj = self.model(**data)
        self.db.add(db_obj)
        await self.db.commit()
        await self.db.refresh(db_obj)
        return db_obj

    async def get(self, **kwargs) -> T | None:
        """
        Retrieve the first object matching the given keyword filters
        """
        result = await self.db.execute(select(self.model).filter_by(**kwargs))
        return result.scalars().first()

    async def get_all(self, **kwargs) -> list[T]:
        """
        Retrieve all objects matching the given keyword filters
        """
        result = await self.db.execute(select(self.model).filter_by(**kwargs))
        return result.scalars().all()

    async def update(self, obj: T, *, data: dict) -> T:
        """
        Update an existing model instance with the provided fields
        """
        for field, value in data.items():
            setattr(obj, field, value)
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def delete(self, obj: T) -> None:
        """
        Delete a model instance from the database
        """
        await self.db.delete(obj)
        await self.db.commit()
```

## Module CRUD Classes

CRUD classes follow the `{Model}CRUD` naming convention — the model name first, followed by `CRUD`.

**Yes:** `BusinessCRUD`, `UserCRUD`, `CampaignCRUD`, `ContactCRUD`
**No:** `CRUDBusiness`, `CRUDUser` — wrong order

Each module defines its own CRUD class in `{module}/crud.py`, extending `CRUDBase`. Each CRUD class defines its own `__init__` that hard-codes the model, so callers only ever pass `db`. Add custom query methods here when the base methods are insufficient.

```python
# app/businesses/crud.py
from sqlalchemy.ext.asyncio import AsyncSession

from app.businesses import models
from app.db.crud import CRUDBase


class BusinessCRUD(CRUDBase[models.Business]):
    """
    CRUD operations for the Business model
    """

    def __init__(self, db: AsyncSession):
        super().__init__(model=models.Business, db=db)

    async def get_by_slug(self, slug: str) -> models.Business | None:
        """
        Retrieve a business by its slug
        """
        return await self.get(slug=slug)
```

## Service Function Patterns

Instantiate the CRUD class inside the service function, passing only `db`. Service functions always return model objects — never response schemas:

```python
# app/businesses/services.py
import logfire
from sqlalchemy.ext.asyncio import AsyncSession

from app.businesses import models, schemas
from app.businesses.crud import BusinessCRUD
from app.exceptions import NotFound


@logfire.instrument("get_business {business_id}")
async def get_business(business_id: uuid.UUID, db: AsyncSession) -> models.Business:
    """
    Get a business by ID
    """
    crud = BusinessCRUD(db=db)

    # Check: business exists
    business = await crud.get(id=business_id)
    if not business:
        raise NotFound(msg="Business not found")

    return business


@logfire.instrument("create_business")
async def create_business(data: schemas.BusinessCreate, db: AsyncSession) -> models.Business:
    """
    Create a new business
    """
    crud = BusinessCRUD(db=db)
    business = await crud.create(data=data.model_dump())
    logfire.info("Created business {business_id}", business_id=business.id)
    return business


@logfire.instrument("update_business {business_id}")
async def update_business(
    business: models.Business,
    data: schemas.BusinessUpdate,
    db: AsyncSession,
) -> models.Business:
    """
    Update an existing business
    """
    crud = BusinessCRUD(db=db)
    updated = await crud.update(obj=business, data=data.model_dump(exclude_unset=True))
    logfire.info("Updated business {business_id}", business_id=updated.id)
    return updated
```

## Pass Objects, Not IDs

When the caller already has the model object, pass the object directly rather than its ID. Fetching an object only to pass its ID forces an unnecessary re-fetch inside the callee.

**Yes:**
```python
business = await service.get_business(business_id=business_id, db=db)
updated = await service.update_business(business=business, data=payload, db=db)
```

**No:**
```python
# Wrong: caller has `business` but passes its ID, causing a redundant lookup
updated = await service.update_business(business_id=business.id, data=payload, db=db)
```

Only accept a bare ID when the object is genuinely not available at the call site (e.g., a background worker that only has an ID from a message queue).

## Rules

- Never call `db.add()`, `db.commit()`, `db.execute(select(...))`, or `db.delete()` directly in `services.py` or route files — always go through the CRUD class
- Never reuse a CRUD instance across service functions; instantiate it fresh each time
- Add custom query methods to the module CRUD class, not to `services.py`
- If a query is needed across modules, add a method to that module's CRUD class and call it via aliased import
- Pass model objects to service functions whenever the object is already in scope — avoid passing only an ID when the full object is available
