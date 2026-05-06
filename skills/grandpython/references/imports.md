# Import Conventions

## Import Order

Separate each section with a blank line:

1. Standard library imports
2. Third-party package imports
3. Codebase imports

```python
import os
import uuid
from datetime import datetime
from typing import TYPE_CHECKING

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from app.auth import service
from app.businesses import models, schemas
```

## `from __future__ import annotations`

Do **not** add `from __future__ import annotations` unless it is strictly required to resolve a forward reference or circular import that cannot be solved another way (e.g. a `TYPE_CHECKING` guard). Never add it by default or out of habit.

## No Function-Level Imports

Always import at the top of the file. Never place imports inside functions or methods unless strictly required to break a circular import.

**Yes:**
```python
from app.businesses import models
from app.businesses.crud import BusinessCRUD

async def get_business(business_id: uuid.UUID, db: AsyncSession) -> models.Business:
    crud = BusinessCRUD(model=models.Business, db=db)
    ...
```

**No:**
```python
async def get_business(business_id: uuid.UUID, db: AsyncSession) -> models.Business:
    from app.businesses.crud import BusinessCRUD  # wrong: function-level import
    crud = BusinessCRUD(model=models.Business, db=db)
    ...
```

## Namespaced Imports

Use namespaced imports to avoid long import sections and name clashes. This makes the source of every function immediately clear.

**Yes:**
```python
from app.auth import service as auth_service
from app.businesses import service as biz_service

result = await auth_service.verify_token(token)
biz = await biz_service.get_business(biz_id)
```

**No:**
```python
from app.auth.service import verify_token
from app.businesses.service import get_business

result = await verify_token(token)  # Which module?
biz = await get_business(biz_id)
```

## Same-Module Named Imports

When importing core components from **within the same module**, use named imports without aliases:

```python
# In app/businesses/router.py
from app.businesses import models, schemas, service
```

Do NOT alias same-module imports:
```python
# Wrong
from app.businesses import models as biz_models  # unnecessary in same module
```

## Cross-Module Aliased Imports

When importing from a **different module**, always use aliases with descriptive prefixes:

```python
from app.auth import models as auth_models
from app.auth import service as auth_service
from app.businesses import models as biz_models
from app.campaigns import schemas as campaign_schemas
from app.contacts import service as contact_service
```

| Module | Prefix | Example |
|--------|--------|---------|
| `app.auth` | `auth_` | `auth_models`, `auth_service` |
| `app.businesses` | `biz_` | `biz_models`, `biz_service` |
| `app.campaigns` | `campaign_` | `campaign_models` |
| `app.contacts` | `contact_` | `contact_service` |
| `app.billing` | `billing_` | `billing_models` |
| `app.whatsapp` | `wa_` | `wa_sender`, `wa_models` |

## Schema Imports

### Same-Module Schemas

When importing schemas from the **same module**, use named imports without aliases:

```python
# In app/businesses/service.py
from app.businesses.schemas import create, base
```

### Cross-Module Schemas

When importing schemas from a **different module**, use aliases with abbreviated descriptive names:

```python
from app.auth.schemas import base as ba_schemas
from app.businesses.schemas import create as bc_schemas
from app.campaigns.schemas import base as cb_schemas
```

Schema alias naming convention — pattern is `{first letter of module}{first letter of schema submodule}_schemas`:

| Schema Path | Alias | Description |
|-------------|-------|-------------|
| `app.auth.schemas.base` | `ba_schemas` | base auth schemas |
| `app.auth.schemas.create` | `ac_schemas` | auth create schemas |
| `app.businesses.schemas.base` | `bb_schemas` | base business schemas |
| `app.businesses.schemas.create` | `bc_schemas` | business create schemas |
| `app.campaigns.schemas.base` | `cb_schemas` | campaign base schemas |
| `app.contacts.schemas.create` | `cc_schemas` | contact create schemas |

If the abbreviation is ambiguous, use a longer prefix (e.g., `biz_base_schemas`).

## Direct Imports

Use direct imports for specific classes, constants, exceptions, and utility functions:

```python
from app.exceptions import NotFound, BadRequest
from app.config import get_settings
from app.utils.crypto import encrypt_value
```

## TYPE_CHECKING Guard

Use `TYPE_CHECKING` for imports needed only for type hints to avoid circular dependencies:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from app.businesses.models import Business
```
