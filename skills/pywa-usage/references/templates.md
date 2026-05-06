# Templates Reference

## Creating Templates

```python
from pywa.types import templates as t

template = t.Template(
    name="my_template",                        # Max 512 chars, lowercase + underscores
    category=t.TemplateCategory.UTILITY,        # MARKETING, AUTHENTICATION, UTILITY
    language=t.TemplateLanguage.ENGLISH_US,
    parameter_format=t.ParamFormat.NAMED,       # or POSITIONAL
    components=[...],                           # List of template components
)

created = wa.create_template(template, waba_id="waba_123")
# created.id, created.status (PENDING or APPROVED)
```

## Component Types

### Header

```python
t.HeaderText(text="Hello {{name}}!", name="Default Name")
t.HeaderImage(image="https://example.com/img.jpg")
t.HeaderVideo(video="https://example.com/vid.mp4")
t.HeaderDocument(document="https://example.com/doc.pdf")
t.HeaderLocation()
```

### Body (required for most categories)

```python
t.BodyText(
    text="Order {{order_id}} ships on {{date}}.",
    order_id="123",
    date="Tomorrow",
)
```

### Footer

```python
t.FooterText(text="Reply STOP to unsubscribe")
```

### Buttons (max 3 CTA or 10 quick-reply)

```python
t.Buttons(buttons=[
    t.URLButton(text="View", url="https://example.com/{{id}}"),
    t.PhoneNumberButton(text="Call", phone_number="+1234567890"),
    t.QuickReplyButton(text="Confirm"),
    t.CopyCodeButton(code="DISCOUNT10"),
])
```

## Sending Templates

```python
sent = wa.send_template(
    to="1234567890",
    template_name="my_template",
    template_language=t.TemplateLanguage.ENGLISH_US,
    parameters={"order_id": "ORD-123", "date": "June 15"},
)
```

## Template Statuses

`APPROVED`, `PENDING`, `REJECTED`, `DISABLED`, `IN_APPEAL`, `PAUSED`, `FLAGGED`, `ARCHIVED`, `DELETED`

## Management

```python
wa.get_template(template_id="123")
wa.get_templates()                          # Returns paginated Result
wa.update_template(template_id="123", template=updated_obj)
wa.delete_template(template_id="123")
wa.migrate_templates(source_waba_id="old", target_waba_id="new")
```

## Listening for Status Updates

```python
@wa.on_template_status(filters.template_status.approved)
def on_approved(wa, update: TemplateStatusUpdate):
    print(f"Template {update.template_id} approved!")

# Or use listener
created = wa.create_template(template, waba_id="waba_id")
approved = created.wait_until_approved(timeout=3600)
```
