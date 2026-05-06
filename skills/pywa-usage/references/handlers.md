# Handlers Reference

## All Handler Types

| Decorator | Callback Signature | Update Type |
|-----------|--------------------|-------------|
| `@wa.on_message()` | `(wa, msg: Message)` | Message |
| `@wa.on_callback_button()` | `(wa, btn: CallbackButton)` | CallbackButton |
| `@wa.on_callback_selection()` | `(wa, sel: CallbackSelection)` | CallbackSelection |
| `@wa.on_message_status()` | `(wa, status: MessageStatus)` | MessageStatus |
| `@wa.on_chat_opened()` | `(wa, co: ChatOpened)` | ChatOpened |
| `@wa.on_flow_completion()` | `(wa, fc: FlowCompletion)` | FlowCompletion |
| `@wa.on_template_status()` | `(wa, tu: TemplateStatusUpdate)` | TemplateStatusUpdate |
| `@wa.on_template_category()` | `(wa, tu: TemplateCategoryUpdate)` | TemplateCategoryUpdate |
| `@wa.on_template_quality()` | `(wa, tu: TemplateQualityUpdate)` | TemplateQualityUpdate |
| `@wa.on_template_components()` | `(wa, tu: TemplateComponentsUpdate)` | TemplateComponentsUpdate |
| `@wa.on_call_connect()` | `(wa, call: CallConnect)` | CallConnect |
| `@wa.on_call_terminate()` | `(wa, call: CallTerminate)` | CallTerminate |
| `@wa.on_call_status()` | `(wa, call: CallStatus)` | CallStatus |
| `@wa.on_call_permission_update()` | `(wa, cpu: CallPermissionUpdate)` | CallPermissionUpdate |
| `@wa.on_phone_number_change()` | `(wa, pnc: PhoneNumberChange)` | PhoneNumberChange |
| `@wa.on_identity_change()` | `(wa, ic: IdentityChange)` | IdentityChange |
| `@wa.on_user_marketing_preferences()` | `(wa, ump: UserMarketingPreferences)` | UserMarketingPreferences |
| `@wa.on_raw_update()` | `(wa, raw: RawUpdate)` | RawUpdate |

## Handler Priority

Higher number = executes first. Default is 0.

```python
@wa.on_message(filters.text, priority=10)  # Runs first
def log_all(wa, msg): print(msg)

@wa.on_message(filters.text.command("help"), priority=5)  # Runs second
def help_cmd(wa, msg): msg.reply("Help!")

@wa.on_message(filters.text)  # Runs last (priority=0)
def fallback(wa, msg): msg.reply("Unknown command")
```

## Execution Flow

1. Webhook received → signature validated → duplicates filtered
2. Handlers sorted by priority (descending)
3. For each handler: check filter → if pass, execute callback
4. If `continue_handling=False` (default): stop after first match
5. Exceptions are logged; processing continues to next handler

## Programmatic Registration

```python
from pywa.handlers import MessageHandler

handler = MessageHandler(
    callback=my_func,
    filters=filters.text,
    priority=10,
)
wa.add_handlers(handler)
```

## Flow Request Handler

For handling Flow lifecycle events (INIT, DATA_EXCHANGE, etc.):

```python
from pywa.handlers import FlowRequestHandler

wa.add_flow_request_handler(
    FlowRequestHandler(
        endpoint="/flow",
        callback=handle_flow,
    )
)
```

## Type-safe Callback Data with factory

```python
@wa.on_callback_button(factory=MyCallbackData)
def on_btn(wa: WhatsApp, btn: CallbackButton[MyCallbackData]):
    btn.data.my_field  # type-safe
```

## Key Update Properties

### Message

- `msg.id`, `msg.type`, `msg.text`, `msg.from_user` (User with `.wa_id`, `.name`)
- `msg.image`, `msg.video`, `msg.audio`, `msg.document`, `msg.sticker`, `msg.location`, `msg.contacts`
- `msg.caption`, `msg.reaction`, `msg.order`, `msg.referral`
- `msg.reply_to_message`, `msg.forwarded`, `msg.forwarded_many_times`
- `msg.timestamp`, `msg.metadata`
- Methods: `msg.reply()`, `msg.react(emoji)`, `msg.unreact()`, `msg.mark_as_read()`

### CallbackButton / CallbackSelection

- `btn.data` (str or CallbackData subclass), `btn.title`, `btn.reply_to_message`
- `btn.from_user`, `btn.timestamp`
- Methods: `btn.reply()`, `btn.react()`, `btn.mark_as_read()`

### MessageStatus

- `status.id`, `status.status` (SENT/DELIVERED/READ/PLAYED/FAILED)
- `status.from_user`, `status.timestamp`
- `status.error`, `status.tracker`, `status.conversation`, `status.pricing`

### FlowCompletion

- `fc.flow_token`, `fc.status` (COMPLETED/DECLINED), `fc.response`

### ChatOpened

- First-time message from a user. Same properties as base update.
