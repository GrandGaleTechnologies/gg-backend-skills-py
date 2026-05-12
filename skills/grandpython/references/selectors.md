# Selectors Convention

## Purpose

Each module has a `selectors.py` with **read-only lookup functions** that retrieve model objects by ID or other filters, wrapping the module's CRUD class and optionally raising domain-specific exceptions.

Selectors are the single authoritative place to look up an object and raise a 404-style exception if it is not found. They prevent the same `crud.get() → raise` pattern from being duplicated across routes and service functions.

---

## Naming Convention

| Function | Use for |
|----------|---------|
| `get_{entity}_by_{field}` | Single object lookup by one field |
| `get_{entity}_{qualifier}` | Single object with compound filter |
| `get_all_{entity}_{qualifier}` | List of objects |

---

## Signature Pattern

Every selector must follow this shape:

```python
async def get_{entity}_by_{field}(
    {field}: type,
    db: AsyncSession,
    raise_exc: bool = True,
) -> models.Entity | None:
```

Rules:
- `db: AsyncSession` is always present
- `raise_exc: bool = True` is always the last parameter and defaults to `True`
- Return type is `Model | None` for single objects; `list[Model]` for collections (list selectors never accept `raise_exc`)
- Always use keyword arguments at call sites: `get_business_by_id(business_id=business_id, db=db)`

---

## Minimal Example

```python
# app/{module}/selectors.py
import uuid

from sqlalchemy.ext.asyncio import AsyncSession

from app.{module} import models
from app.{module}.crud import {Entity}CRUD
from app.{module}.exceptions import {Entity}NotFound


async def get_{entity}_by_id(
    {entity}_id: uuid.UUID,
    db: AsyncSession,
    raise_exc: bool = True,
) -> models.{Entity} | None:
    """
    Get {entity} by ID
    """
    crud = {Entity}CRUD(db=db)

    # Check: {entity} exists
    obj = await crud.get(id={entity}_id)
    if not obj and raise_exc:
        raise {Entity}NotFound()

    return obj
```

---

## Multi-Tenant Scoped Lookup

When a resource is scoped to a tenant (e.g. business), put the scoping ID as the first parameter:

```python
async def get_contact_by_id(
    business_id: uuid.UUID,
    contact_id: uuid.UUID,
    db: AsyncSession,
    raise_exc: bool = True,
) -> models.Contact | None:
    """
    Get contact by ID, scoped to the given business
    """
    crud = ContactCRUD(db=db)

    # Check: contact exists
    contact = await crud.get(id=contact_id, business_id=business_id, is_deleted=False)
    if not contact and raise_exc:
        raise ContactNotFound()

    return contact
```

---

## List Selectors

List selectors return all matching objects and never accept `raise_exc`. When complex filtering is needed (joins, `ILIKE`, ordering), add a custom method to the CRUD class and call it from the selector — never use `db.execute()` directly in selectors.

```python
async def get_all_campaigns_by_business(
    business_id: uuid.UUID,
    db: AsyncSession,
) -> list[models.Campaign]:
    """
    Return all campaigns for a business
    """
    crud = CampaignCRUD(db=db)
    return await crud.get_all(business_id=business_id)
```

---

## `cast()` for Type Narrowing

When calling a selector with `raise_exc=True` (the default), the call either returns the object or stops execution — it will never return `None`. Wrap the call in `cast()` from `typing` so type checkers and IDE autocomplete treat the result as the concrete model type.

```python
from typing import cast

# Yes — typed as models.Contact, full autocomplete
contact = cast(models.Contact, await selectors.get_contact_by_id(
    business_id=current_user.business_id,
    contact_id=contact_id,
    db=db,
))

# No — typed as models.Contact | None, type checker will complain on attribute access
contact = await selectors.get_contact_by_id(...)
```

Only omit `cast()` when you pass `raise_exc=False` and genuinely need to handle the `None` case.

---

## Usage in Routes and Services

```python
# route — resolve path param, then format and return
async def route_contact_details(
    contact_id: uuid.UUID,
    db: DbSessionDep,
    current_user: CurrentUserDep,
):
    """
    Get a single contact
    """
    contact = cast(models.Contact, await selectors.get_contact_by_id(
        business_id=current_user.business_id,
        contact_id=contact_id,
        db=db,
    ))
    return {"data": formatters.fmt_contact(contact=contact)}


# service — precondition check before an operation
async def create_campaign(
    contact_id: uuid.UUID,
    data: schemas.CampaignCreate,
    business_id: uuid.UUID,
    db: AsyncSession,
) -> models.Campaign:
    """
    Create a new campaign for a contact
    """
    # Check: contact exists
    cast(models.Contact, await contact_selectors.get_contact_by_id(
        business_id=business_id,
        contact_id=contact_id,
        db=db,
    ))

    crud = CampaignCRUD(db=db)
    return await crud.create(data={**data.model_dump(), "contact_id": contact_id})
```

---

## Cross-Module Usage

Import the selectors module with an aliased namespace import:

```python
from app.contacts import selectors as contact_selectors

contact = cast(models.Contact, await contact_selectors.get_contact_by_id(
    business_id=business_id,
    contact_id=contact_id,
    db=db,
))
```

**No:**
```python
from app.contacts.selectors import get_contact_by_id  # wrong: direct import across modules
```

---

## Rules

- **Read-only only** — selectors never write to the database
- **CRUD always** — never call `db.execute()` directly; complex queries go as custom CRUD class methods
- **No business logic** — orchestration belongs in `services.py`
- **No auth dependencies** — `get_current_user` and WebSocket auth functions belong in `dependencies.py`
- **No utility logic** — helpers without DB access belong in `utils.py`
- **`cast()` always** — wrap every `raise_exc=True` call site with `cast(Model, await ...)`
- **Docstring required** — every selector function must have a docstring

---

## Anti-Pattern Table

| What | Where it belongs |
|------|-----------------|
| `get_current_user()` / `get_current_user_ws()` | `dependencies.py` + `annotations.py` |
| Writing / updating records | `services.py` via CRUD |
| Business logic orchestration | `services.py` |
| Raw `db.execute()` queries | Custom method in `crud.py` |
| Utility / transformation functions | `utils.py` |
