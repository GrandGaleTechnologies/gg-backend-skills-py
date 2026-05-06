# Pydantic Schemas

## Schema Layout

Every module's schemas live in a `schemas/` package (never a single `schemas.py` file):

```
{module}/schemas/
├── __init__.py   # Re-exports all public schemas for backward-compatible imports
├── base.py       # Read / shared schemas (e.g. UserRead, BusinessRead)
├── create.py     # Create & request schemas (e.g. SignupRequest, BusinessCreate)
├── edit.py       # Edit / update schemas (e.g. BusinessUpdate)
└── response.py   # Composite response schemas (e.g. TokenResponse, SignupResponse)
```

## Which File Does Each Schema Live In?

| Schema kind | File | Examples |
|-------------|------|---------|
| DB read representation | `base.py` | `UserRead`, `BusinessRead`, `ContactRead` |
| Create/POST request body | `create.py` | `BusinessCreate`, `SignupRequest`, `OTPVerify` |
| Edit/PUT/PATCH request body | `edit.py` | `BusinessUpdate`, `ContactEdit` |
| Composite API response | `response.py` | `TokenResponse`, `SignupResponse` |

`create.py` owns **all request body schemas**, not just "create" ones — it is the natural home for any schema that arrives in an inbound HTTP request body.

`response.py` owns schemas that compose multiple pieces together for a response. Simple response shapes that mirror a single DB object go in `base.py`.

## `__init__.py` Re-Exports

Re-export every public schema so both import styles remain valid:

```python
# schemas/__init__.py
from app.businesses.schemas.base import BusinessRead
from app.businesses.schemas.create import BusinessCreate, WhatsAppCredentials
from app.businesses.schemas.edit import BusinessUpdate

__all__ = ["BusinessRead", "BusinessCreate", "WhatsAppCredentials", "BusinessUpdate"]
```

This means callers can use either style:

```python
# Style 1 — package-level (used in same-module files via `from app.businesses import schemas`)
schemas.BusinessRead

# Style 2 — submodule-level (preferred for cross-module imports)
from app.businesses.schemas import base as bb_schemas
bb_schemas.BusinessRead
```

## Inheritance Rules

Never use inheritance between Pydantic schemas — **with one explicit exception**: schemas in `response.py` must inherit from `Response` or `PaginatedResponse`. All other schemas (`base.py`, `create.py`, `edit.py`) must be standalone flat classes that inherit only from `BaseModel`. Duplicating fields across schemas is preferred over sharing them via a custom base class.

**Yes:**
```python
# base.py / create.py / edit.py — flat, no inheritance
class BusinessCreate(BaseModel):
    business_name: str
    phone: str
    description: str | None = None

class BusinessRead(BaseModel):
    id: uuid.UUID
    business_name: str
    phone: str
    description: str | None
    created_at: datetime

# response.py — MUST inherit from Response or PaginatedResponse
class BusinessDetailResponse(Response):
    data: BusinessRead = Field(description="The business object")
```

**No:**
```python
class BusinessBase(BaseModel):  # wrong: no custom base schemas for inheritance
    business_name: str
    phone: str

class BusinessCreate(BusinessBase):  # wrong: inheriting from a custom base
    description: str | None = None

class BusinessDetailResponse(BaseModel):  # wrong: response.py schema must inherit from Response
    business: BusinessRead
```

## Response & PaginatedResponse

All schemas in `response.py` **must** inherit from `Response` or `PaginatedResponse`. These base classes live in `app/common/schemas.py`:

```python
# app/common/schemas.py
class Response(BaseModel):
    status: str = Field(default="success")
    msg: str = Field(default="Request Successful")
    data: Any = Field(description="The response data")

class PaginationMeta(BaseModel):
    total_no_items: int
    total_no_pages: int
    page: int
    size: int
    count: int
    has_next_page: bool
    has_prev_page: bool

class PaginatedResponse(Response):
    meta: PaginationMeta = Field(description="The pagination metadata")
```

Import directly from `app.common.schemas` — do not import via `app.common`:

```python
# Yes
from app.common.schemas import Response, PaginatedResponse

# No
from app.common import schemas  # wrong: use direct import
```

Each response schema overrides `data` with a concrete inner schema. Inner schemas live in `base.py`:

```python
# app/businesses/schemas/base.py
class BusinessRead(BaseModel):
    """
    Business read schema
    """

    id: uuid.UUID = Field(description="Unique business identifier")
    business_name: str = Field(description="The display name of the business")
    ...
```

```python
# app/businesses/schemas/response.py
from app.businesses.schemas.base import BusinessRead
from app.common.schemas import PaginatedResponse, Response


class BusinessDetailResponse(Response):
    """
    Business detail response schema
    """

    data: BusinessRead = Field(description="The business object")


class BusinessListResponse(PaginatedResponse):
    """
    Business list response schema
    """

    data: list[BusinessRead] = Field(description="List of businesses")
```

## Wrapping Data in Routes

Formatters return a plain dict of the data fields only. Routes wrap the formatter output under the `"data"` key before returning — the `Response`/`PaginatedResponse` base class handles `status` and `msg` via defaults:

```python
# route — wraps in {"data": ...}
return {"data": formatters.fmt_business(business=business)}

# paginated route — also includes "meta"
businesses, meta = await service.list_businesses(db=db, page=page, size=size)
return {
    "data": formatters.fmt_business_list(businesses=businesses),
    "meta": meta,
}
```

Never construct `Response` or `PaginatedResponse` instances manually.

## Field Validators

Always add a docstring to every `@field_validator` method using the `Tasks:` bullet format:

```python
@field_validator("phone", mode="before")
@classmethod
def normalize_phone(cls, v: str) -> str:
    """
    Tasks:
        - Normalize the phone number to E.164 format (+234XXXXXXXXX)
    """
    return normalize_nigerian_phone(v)

@field_validator("slug", mode="before")
@classmethod
def slugify_name(cls, v: str) -> str:
    """
    Tasks:
        - Convert the name to a URL-safe lowercase hyphen-separated slug
    """
    return v.lower().replace(" ", "-")
```

**No:**
```python
@field_validator("phone", mode="before")
@classmethod
def normalize_phone(cls, v: str) -> str:  # wrong: missing docstring
    return normalize_nigerian_phone(v)

@field_validator("phone", mode="before")
@classmethod
def normalize_phone(cls, v: str) -> str:
    """Normalize phone."""  # wrong: inline format
    return normalize_nigerian_phone(v)
```

## Schema Fields

Always annotate every schema attribute with `Field()`. Every `Field()` must include a `description`.

**Yes:**
```python
class BusinessCreate(BaseModel):
    business_name: str = Field(description="The display name of the business")
    phone: str = Field(description="Primary contact phone number")
    description: str | None = Field(default=None, description="Optional description of the business")

class BusinessRead(BaseModel):
    id: uuid.UUID = Field(description="Unique business identifier")
    business_name: str = Field(description="The display name of the business")
    phone: str = Field(description="Primary contact phone number")
    created_at: datetime = Field(description="Timestamp when the business was created")
```

**No:**
```python
class BusinessCreate(BaseModel):
    business_name: str  # wrong: no Field()
    phone: str = Field()  # wrong: Field() has no description
    description: str | None = None  # wrong: no Field()
```

## Rules

- Every schema in `response.py` **must** inherit from `Response` or `PaginatedResponse` — never from `BaseModel` directly
- Inner data schemas (`*Read`) live in `base.py`, not in `response.py`
- Formatters return a flat dict of data fields only — never include `status`, `msg`, or `data` keys
- Routes wrap formatter output with `{"data": formatters.fmt_...(...)}` — never construct `Response` instances manually
- For paginated responses, include `{"meta": meta_dict}` alongside `"data"` in the route return dict
