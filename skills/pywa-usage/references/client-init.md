# Client Initialization

## Constructor Parameters

```python
WhatsApp(
    phone_id: str | int | None = None,       # WhatsApp phone number ID
    token: str = None,                         # Graph API access token
    *,
    session: httpx.Client | None = None,       # Custom httpx session
    server: Flask | FastAPI | None = MISSING,  # Web framework app (auto-detect)
    webhook_endpoint: str = "/",               # POST/GET endpoint path
    verify_token: str | None = None,           # Webhook verification token
    filter_updates: bool = True,               # Filter updates to this phone_id
    continue_handling: bool = False,            # Continue to next handler after match
    skip_duplicate_updates: bool = True,       # Deduplicate webhook updates
    validate_updates: bool = True,             # Validate X-Hub-Signature-256 (needs app_secret)
    business_account_id: str | int | None = None,  # WABA ID
    callback_url: str | None = None,           # Auto-register webhook URL with Meta
    callback_url_scope: CallbackURLScope = APP,    # APP, WABA, or PHONE scope
    webhook_fields: Iterable[str] | None = None,   # Subscribed fields (reduce traffic)
    app_id: int | str | None = None,           # Meta App ID (for callback_url registration)
    app_secret: str | None = None,             # Meta App Secret (for signature validation)
    webhook_challenge_delay: int = 3,          # Seconds to wait for verification challenge
    business_private_key: str | None = None,   # RSA key for Flow encryption
    business_private_key_password: str | None = None,
    flows_request_decryptor: FlowRequestDecryptor | None = default,
    flows_response_encryptor: FlowResponseEncryptor | None = default,
    api_version: str | int | float | Version.GRAPH_API = Version.GRAPH_API,
    handlers_modules: Iterable[ModuleType] | None = None,  # Auto-load handler modules
)
```

## Initialization Patterns

### Send-only (no webhook)

```python
wa = WhatsApp(phone_id="123456", token="token")
```

### With Flask

```python
from flask import Flask
from pywa import WhatsApp

app = Flask(__name__)
wa = WhatsApp(
    phone_id="123456",
    token="token",
    server=app,
    verify_token="my_verify_token",
    app_secret="my_app_secret",
)
app.run(port=5000)
```

### With FastAPI (async)

```python
from fastapi import FastAPI
from pywa_async import WhatsApp

app = FastAPI()
wa = WhatsApp(
    phone_id="123456",
    token="token",
    server=app,
    verify_token="my_verify_token",
    app_secret="my_app_secret",
)
# Run with: uvicorn main:app
```

### Auto-register webhook URL

```python
wa = WhatsApp(
    phone_id="123456",
    token="token",
    server=app,
    verify_token="my_verify_token",
    app_secret="my_app_secret",
    callback_url="https://my-domain.com",
    app_id="app_id",
)
```

### Manual webhook handling (no server)

```python
wa = WhatsApp(phone_id="123456", token="token", server=None)

# In your custom route handler:
wa.webhook_update_handler(
    body=request_json,
    hmac_header=request.headers.get("X-Hub-Signature-256"),
)
```

### Override phone_id per request

```python
wa = WhatsApp(token="token")  # No default phone_id
wa.send_message(to="123", text="Hi", sender="phone_id_1")
wa.send_message(to="456", text="Hi", sender="phone_id_2")
```

## Key Behaviors

- `filter_updates=True`: Only processes updates destined for the configured `phone_id` / `business_account_id`.
- `validate_updates=True`: Requires `app_secret` to verify webhook signatures. Set to `False` only in development.
- `skip_duplicate_updates=True`: Prevents processing the same webhook update twice.
- `continue_handling=False`: Stops after the first matching handler. Set `True` to allow multiple handlers per update.
- `handlers_modules`: Pass module objects to auto-discover and load handlers defined in other files.

## All Public Methods

### Message Sending

| Method | Returns | Purpose |
|--------|---------|---------|
| `send_message()` | `SentMessage` | Text with optional buttons/list |
| `send_image()` | `SentMediaMessage` | Image (URL/path/bytes) |
| `send_video()` | `SentMediaMessage` | Video |
| `send_document()` | `SentMediaMessage` | Document with filename |
| `send_audio()` | `SentMediaMessage` | Audio file |
| `send_voice()` | `SentVoiceMessage` | Voice note (OGG/OPUS) |
| `send_sticker()` | `SentMediaMessage` | Sticker (512x512 PNG/WEBP) |
| `send_reaction()` | `SentReaction` | Emoji reaction |
| `remove_reaction()` | `SentReaction` | Remove reaction |
| `send_location()` | `SentMessage` | GPS coordinates |
| `request_location()` | `SentLocationRequest` | Request user's location |
| `send_contact()` | `SentMessage` | Contact card(s) |
| `send_catalog()` | `SentMessage` | Product catalog |
| `send_product()` | `SentMessage` | Single product |
| `send_products()` | `SentMessage` | Multi-product list |
| `send_template()` | `SentTemplate` | Pre-approved template |
| `send_flow()` | `SentMessage` | WhatsApp Flow |

### Media Management

| Method | Purpose |
|--------|---------|
| `upload_media()` | Upload media (30-day expiration) |
| `get_media_url()` | Get download URL (5-min expiration) |
| `download_media()` | Download to file |
| `get_media_bytes()` | Get as bytes |
| `stream_media()` | Stream as byte chunks |
| `delete_media()` | Delete from servers |

### Template Management

| Method | Purpose |
|--------|---------|
| `create_template()` | Create new template |
| `get_template()` | Fetch by ID |
| `get_templates()` | List all (paginated `Result`) |
| `update_template()` | Update content |
| `delete_template()` | Delete |
| `migrate_templates()` | Migrate between WABAs |
| `compare_templates()` | Compare versions |

### Flow Management

| Method | Purpose |
|--------|---------|
| `create_flow()` | Create new Flow |
| `update_flow_json()` | Update Flow JSON |
| `get_flow()` | Fetch by ID |
| `get_flows()` | List all (paginated) |
| `delete_flow()` | Delete |
| `get_flow_metrics()` | Analytics |
| `migrate_flows()` | Migrate between WABAs |

### Account & Profile

| Method | Purpose |
|--------|---------|
| `get_business_account()` | WABA info |
| `get_business_phone_number()` | Phone number details |
| `get_business_phone_numbers()` | List phone numbers |
| `update_display_name()` | Change display name |
| `get_business_profile()` | Profile info |
| `update_business_profile()` | Update profile |
| `update_conversational_automation()` | Ice-breakers/commands |
| `get_commerce_settings()` | Catalog settings |
| `update_commerce_settings()` | Enable/disable catalog |

### Webhook & Handlers

| Method | Purpose |
|--------|---------|
| `mark_message_as_read()` | Mark message read |
| `indicate_typing()` | Show typing indicator |
| `webhook_update_handler()` | Manually process update |
| `register_callback_url()` | Register webhook URL |
| `add_handlers()` | Register handlers programmatically |
| `remove_handlers()` | Remove handlers |
| `remove_callbacks()` | Remove by function reference |
| `load_handlers_modules()` | Load handlers from modules |
