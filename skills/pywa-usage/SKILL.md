---
name: pywa-usage
description: "Write code using the pywa or pywa_async WhatsApp Cloud API Python packages. Use when: building WhatsApp bots, sending messages/media/templates, handling webhooks, creating Flows, registering handlers/filters, using listeners, or working with callback data. Covers sync (Flask) and async (FastAPI) patterns."
argument-hint: "Describe what you want to build with pywa (e.g., 'echo bot with buttons', 'template sender', 'Flow handler')"
---

# PyWa Usage

Use this skill when writing code that imports from `pywa` or `pywa_async`. This covers the WhatsApp Cloud API Python framework for sending messages, handling webhooks, building Flows, and managing templates.

## Quick Reference

- **Sync**: `from pywa import WhatsApp` — use with Flask or no server
- **Async**: `from pywa_async import WhatsApp` — use with FastAPI or no server
- **Types**: `from pywa.types import Message, CallbackButton, ...` (or `from pywa_async.types import ...`)
- **Filters**: `from pywa import filters` (or `from pywa_async import filters`)
- **Templates module**: `from pywa.types import templates as t`
- **Errors**: `from pywa.errors import WhatsAppError, ...`
- **Listeners**: Exceptions from `from pywa.types import ListenerTimeout, ListenerCanceled, ListenerStopped`

## Initialization

See [client-init reference](./references/client-init.md) for constructor parameters.

### Sync (Flask)

```python
from flask import Flask
from pywa import WhatsApp

app = Flask(__name__)
wa = WhatsApp(
    phone_id="123456",
    token="your_token",
    server=app,
    verify_token="verify_token",
    app_secret="app_secret",
)
```

### Async (FastAPI)

```python
from fastapi import FastAPI
from pywa_async import WhatsApp

app = FastAPI()
wa = WhatsApp(
    phone_id="123456",
    token="your_token",
    server=app,
    verify_token="verify_token",
    app_secret="app_secret",
)
```

### Send-only (no webhook)

```python
from pywa import WhatsApp

wa = WhatsApp(phone_id="123456", token="your_token")
wa.send_message(to="1234567890", text="Hello!")
```

## Handlers & Filters

Register callbacks with decorators. The first argument is always the `WhatsApp` client, the second is the typed update.

```python
from pywa import WhatsApp, filters
from pywa.types import Message, CallbackButton, CallbackSelection, MessageStatus

@wa.on_message(filters.text)
def on_text(wa: WhatsApp, msg: Message):
    msg.reply(f"You said: {msg.text}")

@wa.on_callback_button
def on_btn(wa: WhatsApp, btn: CallbackButton):
    btn.reply(f"Clicked: {btn.data}")

@wa.on_callback_selection
def on_sel(wa: WhatsApp, sel: CallbackSelection):
    sel.reply(f"Selected: {sel.data}")

@wa.on_message_status(filters.message_status.failed)
def on_fail(wa: WhatsApp, status: MessageStatus):
    print(f"Failed: {status.error}")
```

### Async handlers

```python
@wa.on_message(filters.text)
async def on_text(wa: WhatsApp, msg: Message):
    await msg.reply(f"Echo: {msg.text}")
```

See [handlers reference](./references/handlers.md) for the full handler table and priority system.

### Filter composition

```python
filters.text & filters.startswith("hi")   # AND
filters.image | filters.video              # OR
~filters.forwarded                         # NOT
filters.from_users("123", "456")           # specific senders
filters.text.command("start")              # /start
filters.text.matches(r"^\d+$")            # regex
```

See [filters reference](./references/filters.md) for all built-in filters.

## Sending Messages

All `send_*` methods return a `SentMessage` (or variant). Media accepts URL strings, `pathlib.Path`, `bytes`, file objects, or media IDs.

```python
from pywa.types import Button, SectionList, Section, SectionRow

# Text with buttons (max 3)
wa.send_message(
    to="123",
    text="Pick one:",
    buttons=[
        Button(title="Option A", callback_data="a"),
        Button(title="Option B", callback_data="b"),
    ],
)

# Selection list
wa.send_message(
    to="123",
    text="Choose:",
    buttons=SectionList(
        button="Open list",
        sections=[Section(title="Colors", rows=[
            SectionRow(title="Red", callback_data="red", description="Warm"),
            SectionRow(title="Blue", callback_data="blue"),
        ])],
    ),
)

# Media
wa.send_image(to="123", image="https://example.com/img.jpg", caption="Check this out")
wa.send_document(to="123", document=pathlib.Path("report.pdf"), filename="report.pdf")

# Location
wa.send_location(to="123", latitude=37.7749, longitude=-122.4194, name="SF")

# Reaction
wa.send_reaction(to="123", emoji="👍", message_id="wamid.xxx")
```

## Callback Data (type-safe)

```python
from dataclasses import dataclass
from pywa.types import CallbackData, Button, CallbackButton

@dataclass(frozen=True, slots=True)
class Vote(CallbackData):
    option: str
    user_id: int

wa.send_message(
    to="123",
    text="Vote:",
    buttons=[Button(title="Yes", callback_data=Vote(option="yes", user_id=42))],
)

@wa.on_callback_button(factory=Vote)
def on_vote(wa: WhatsApp, btn: CallbackButton[Vote]):
    print(btn.data.option, btn.data.user_id)  # type-safe access
```

**Limits**: max 200 characters encoded. Supported field types: `str`, `int`, `bool`, `float`, `Enum`.

## Listeners (wait for reply)

```python
from pywa.types import ListenerTimeout, ListenerCanceled

sent = wa.send_message(to="123", text="Reply yes or no")
try:
    reply = sent.wait_for_reply(timeout=30, filters=filters.text)
    print(reply.text)
except ListenerTimeout:
    wa.send_message(to="123", text="Too slow!")
except ListenerCanceled:
    wa.send_message(to="123", text="Cancelled")
```

Other listener shortcuts: `sent.wait_for_click()`, `sent.wait_for_selection()`, `sent.wait_until_read()`.

## Templates

See [templates reference](./references/templates.md) for creation and sending.

```python
from pywa.types import templates as t

template = t.Template(
    name="order_update",
    category=t.TemplateCategory.UTILITY,
    language=t.TemplateLanguage.ENGLISH_US,
    parameter_format=t.ParamFormat.NAMED,
    components=[
        t.BodyText(text="Order {{order_id}} ships {{date}}", order_id="123", date="Tomorrow"),
        t.Buttons(buttons=[t.QuickReplyButton(text="Track")]),
    ],
)
created = wa.create_template(template, waba_id="waba_123")
```

## Flows

See [flows reference](./references/flows.md) for Flow lifecycle handling.

```python
wa.send_flow(
    to="123",
    flow_id="flow_abc",
    flow_action_data={"screen": "FIRST_SCREEN"},
)

@wa.on_flow_completion
def on_flow(wa: WhatsApp, fc: FlowCompletion):
    print(fc.response)
```

## Error Handling

```python
from pywa.errors import WhatsAppError, RateLimitError

try:
    wa.send_message(to="123", text="Hi")
except RateLimitError:
    # backoff and retry
    pass
except WhatsAppError as e:
    print(e.code, e.message, e.is_transient)
```

## Critical Rules

1. **Never mix imports**: use `from pywa import WhatsApp` OR `from pywa_async import WhatsApp`, not both.
2. **Async handlers require `async def`** when using `pywa_async`.
3. **Flask is sync-only**, FastAPI is async-only for webhook serving.
4. **Button callback_data max 200 chars** — use `CallbackData` subclass for structured data.
5. **Media URLs expire in 5 minutes** — always call `get_media_url()` fresh.
6. **Uploaded media expires in 30 days** (user media: 7 days).
7. **`filter_updates=True`** (default) filters webhooks to only the configured `phone_id`.
8. **`app_secret` is required** when `validate_updates=True` (default) for webhook signature verification.
9. **Handler priority**: higher number = executes first. Default is 0.
10. **`continue_handling=False`** (default) stops after the first matching handler.
