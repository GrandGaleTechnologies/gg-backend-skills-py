# Formatters

Each module has a `formatters.py` that converts model instances into plain dicts for API responses. This separates serialization logic from service logic and makes it reusable across route functions.

Formatters must **never** perform database queries, call external services, or contain business logic — they only transform data. Formatters always return `dict`, never Pydantic schema instances. FastAPI validates the dict against `response_model` in the background.

## Naming Convention

| Function | Use for |
|----------|---------|
| `fmt_{resource}` | Single object |
| `fmt_{resource}_list` | Collection |

## Structure

```python
# app/businesses/formatters.py
from app.businesses import models


def fmt_business(business: models.Business) -> dict:
    """
    Format a Business model into a response dict
    """
    return {
        "id": business.id,
        "business_name": business.business_name,
        "slug": business.slug,
        "phone": business.phone,
        "business_type": business.business_type,
        "description": business.description,
        "default_language": business.default_language,
        "status": business.status,
        "is_active": business.is_active,
        "wa_configured": business.wa_configured,
        "created_at": business.created_at,
        "updated_at": business.updated_at,
    }


def fmt_business_list(businesses: list[models.Business]) -> list[dict]:
    """
    Format a list of Business models into response dicts
    """
    return [fmt_business(business=b) for b in businesses]
```

## Nested Formatters

Compose formatters together to build nested dicts:

```python
# app/auth/formatters.py
from app.auth import models
from app.businesses import formatters as biz_formatters
from app.businesses import models as biz_models


def fmt_user(user: models.User) -> dict:
    """
    Format a User model into a response dict
    """
    return {
        "id": user.id,
        "business_id": user.business_id,
        "phone": user.phone,
        "is_verified": user.is_verified,
        "is_active": user.is_active,
        "last_login_at": user.last_login_at,
        "created_at": user.created_at,
        "updated_at": user.updated_at,
    }


def fmt_token_response(
    access_token: str,
    refresh_token: str,
    user: models.User,
    business: biz_models.Business,
) -> dict:
    """
    Format JWT tokens, User, and Business into a token response dict
    """
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "user": fmt_user(user=user),
        "business": biz_formatters.fmt_business(business=business),
    }


def fmt_signup_response(phone: str, message: str) -> dict:
    """
    Format signup confirmation data into a response dict
    """
    return {
        "phone": phone,
        "message": message,
    }
```

## Usage in Routes

Formatters are called **only at the route level**. Service functions return raw model objects; the route calls the formatter and wraps the result under `{"data": ...}`:

```python
# app/businesses/routes/base.py
from app.businesses import formatters, service


@router.get(
    "/{business_id}",
    summary="Get business details",
    response_description="Business object",
    status_code=200,
    response_model=bb_schemas.BusinessDetailResponse,
)
async def route_business_details(business_id: uuid.UUID, db: DbSessionDep):
    """
    Get a single business by ID
    """
    business = await service.get_business(business_id=business_id, db=db)
    return {"data": formatters.fmt_business(business=business)}
```

For paginated routes, also include `"meta"`:

```python
businesses, meta = await service.list_businesses(db=db, page=page, size=size)
return {
    "data": formatters.fmt_business_list(businesses=businesses),
    "meta": meta,
}
```

**No:**
```python
# formatters.py — wrong: returns a Pydantic schema instance
def fmt_business(business: models.Business) -> bb_schemas.BusinessRead:
    return bb_schemas.BusinessRead.model_validate(business)  # wrong: return dict, not schema

# route — wrong: manual schema construction
return bb_schemas.BusinessRead.model_validate(business)  # wrong: use a formatter

# route — wrong: missing data wrapper
return formatters.fmt_business(business=business)  # wrong: must wrap in {"data": ...}
```

## Rules

- Formatters always return `dict` (or `list[dict]`) — never Pydantic schema instances
- Formatters return **only the data fields** — never include `status`, `msg`, or `data` wrapper keys
- Formatters are **only** called in route functions, never in service functions
- Service functions always return model instances, never dicts or response schemas
- Never call `model_validate()`, `from_orm()`, or schema constructors in routes, services, or formatters — build dicts directly
- Formatters must be pure functions: no DB access, no side effects, no async
- Cross-module formatters should live in the source module and be imported with an alias (e.g., `from app.businesses import formatters as biz_formatters`)
