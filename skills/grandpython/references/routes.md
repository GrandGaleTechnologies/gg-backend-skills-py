# Routes

## Route Aggregation (apis.py)

Routes are aggregated in `{module}/apis.py`:

```python
from app.{module}.routes.base import router as base_router

router = APIRouter()

router.include_router(base_router, prefix="/{module}")
```

For modules with admin or feature-specific routes:

```python
from app.{module}.routes.base import router as base_router
from app.{module}.routes.admin import router as admin_router

router = APIRouter()

router.include_router(base_router, prefix="/{module}")
router.include_router(admin_router, prefix="/admin/{module}")
```

## Prefix Patterns

| Route type | Prefix | Example |
|-----------|--------|---------|
| Base routes | `/{module}` | `/businesses`, `/campaigns` |
| Admin routes | `/admin/{module}` | `/admin/businesses` |
| Nested routes | `/{module}/{subresource}` | `/businesses/members` |

## Route Function Naming

All route functions MUST start with the `route_` prefix and use `snake_case`.

Pattern: `route_{module}_{resource}_{action}`

| Action | Pattern | Example |
|--------|---------|---------|
| List | `route_{resource}_list` | `route_contact_list` |
| Details | `route_{resource}_details` | `route_business_details` |
| Create | `route_{resource}_create` | `route_campaign_create` |
| Update | `route_{resource}_edit` | `route_contact_edit` |
| Delete | `route_{resource}_delete` | `route_campaign_delete` |
| Auth | `route_{module}_{action}` | `route_auth_login` |

## Route Decorator Structure

All route decorators must be written in multi-line form with `summary`, `response_description`, `status_code`, and `response_model` as explicit keyword arguments. Never use the shorthand single-line decorator form. Never put return type annotations on the function — use `response_model` instead.

**Yes:**
```python
@router.post(
    "/login",
    summary="Login user",
    response_description="JWT token response",
    status_code=200,
    response_model=schemas.TokenResponse,
)
async def route_auth_login(payload: schemas.LoginRequest, db: DbSessionDep):
    """
    Authenticate user and return a JWT
    """
```

**No:**
```python
@router.post("/login", status_code=200)  # wrong: shorthand, missing required fields
async def route_auth_login(...) -> schemas.TokenResponse:  # wrong: return type on function
```

## Full Examples

```python
@router.post(
    "/login",
    summary="Login user",
    response_description="JWT token response",
    status_code=200,
    response_model=schemas.TokenResponse,
)
async def route_auth_login(...):
    """
    Authenticate user and return a JWT
    """


@router.get(
    "",
    summary="List businesses",
    response_description="Paginated list of businesses",
    status_code=200,
    response_model=schemas.BusinessListResponse,
)
async def route_business_list(...):
    """
    Return all businesses
    """


@router.get(
    "/{business_id}",
    summary="Get business details",
    response_description="Business object",
    status_code=200,
    response_model=schemas.BusinessRead,
)
async def route_business_details(business_id: uuid.UUID, ...):
    """
    Get a single business by ID
    """


@router.post(
    "",
    summary="Create campaign",
    response_description="Created campaign object",
    status_code=201,
    response_model=schemas.CampaignRead,
)
async def route_campaign_create(...):
    """
    Create a new campaign
    """


@router.put(
    "/{contact_id}",
    summary="Edit contact",
    response_description="Updated contact object",
    status_code=200,
    response_model=schemas.ContactRead,
)
async def route_contact_edit(contact_id: uuid.UUID, ...):
    """
    Update an existing contact
    """


@router.delete(
    "/{campaign_id}",
    summary="Delete campaign",
    response_description="Deleted campaign confirmation",
    status_code=200,
    response_model=schemas.DeleteResponse,
)
async def route_campaign_delete(campaign_id: uuid.UUID, ...):
    """
    Delete a campaign by ID
    """
```

## Admin and Nested Resource Routes

```python
# Admin routes include admin_ prefix
@router.get(
    "",
    summary="List businesses (admin)",
    response_description="Paginated list of all businesses",
    status_code=200,
    response_model=schemas.AdminBusinessListResponse,
)
async def route_admin_business_list(...):
    """
    Return all businesses for admin
    """


# Nested resources include full hierarchy
@router.post(
    "/invite",
    summary="Invite business member",
    response_description="Invitation confirmation",
    status_code=200,
    response_model=schemas.InviteResponse,
)
async def route_business_member_invite(...):
    """
    Send an invitation to a new business member
    """
```
