# Filters Reference

All filters are in `pywa.filters` (or `pywa_async.filters`). Compose with `&` (AND), `|` (OR), `~` (NOT).

## Message Content Filters

| Filter | Matches |
|--------|---------|
| `filters.message` | Any message |
| `filters.text` | Text messages |
| `filters.text.command("cmd")` | `/cmd` command |
| `filters.text.is_command` | Any `/command` |
| `filters.text.startswith("x")` | Text starts with "x" |
| `filters.text.endswith("x")` | Text ends with "x" |
| `filters.text.contains("x")` | Text contains "x" |
| `filters.text.matches(r"regex")` | Text matches regex |
| `filters.image` | Image messages |
| `filters.video` | Video messages |
| `filters.audio` | Audio messages |
| `filters.voice` | Voice notes |
| `filters.document` | Documents |
| `filters.sticker` | Stickers |
| `filters.animated_sticker` | Animated stickers |
| `filters.static_sticker` | Static stickers |
| `filters.media` | Any media |
| `filters.has_caption` | Media with caption |
| `filters.location` | Location messages |
| `filters.location.current_location` | Current location share |
| `filters.location_in_radius(lat, lon, r)` | Within radius (km) |
| `filters.contacts` | Contact cards |
| `filters.contacts.has_wa` | Contacts with WhatsApp |
| `filters.order` | E-commerce orders |
| `filters.reaction` | Reactions |
| `filters.reaction.reaction_added` | Reaction added |
| `filters.reaction.reaction_removed` | Reaction removed |
| `filters.reaction.reaction_emojis(["👍"])` | Specific emoji(s) |
| `filters.unsupported` | Unknown message type |

## Callback & Status Filters

| Filter | Matches |
|--------|---------|
| `filters.callback_button` | Button click |
| `filters.callback_button.matches("id")` | Specific callback_data |
| `filters.callback_selection` | List selection |
| `filters.message_status` | Any status update |
| `filters.message_status.sent` | SENT |
| `filters.message_status.delivered` | DELIVERED |
| `filters.message_status.read` | READ |
| `filters.message_status.played` | PLAYED (voice) |
| `filters.message_status.failed` | FAILED |
| `filters.message_status.failed_with(code)` | Specific error code |
| `filters.message_status.with_tracker("x")` | Tracker match |

## Flow & Template Filters

| Filter | Matches |
|--------|---------|
| `filters.flow_completion` | Flow completed |
| `filters.template_status` | Template status change |
| `filters.template_status.approved` | APPROVED |
| `filters.template_status.rejected` | REJECTED |
| `filters.template_quality` | Quality score change |
| `filters.template_category` | Category change |
| `filters.template_components` | Components change |

## User & Context Filters

| Filter | Matches |
|--------|---------|
| `filters.from_users("wa_id", ...)` | Specific senders |
| `filters.from_countries("US", "IL")` | By country code |
| `filters.sent_to_me` | Sent to bot's phone |
| `filters.sent_to("phone_id")` | Sent to specific phone |
| `filters.forwarded` | Forwarded messages |
| `filters.forwarded_many_times` | Forwarded 5+ times |
| `filters.reply` | Replies |
| `filters.replays_to("msg_id")` | Reply to specific msg |
| `filters.has_referred_product` | Ad referral present |

## Call Filters

| Filter | Matches |
|--------|---------|
| `filters.call_connect` | Incoming call |
| `filters.call_terminate` | Call ended |
| `filters.call_status` | Call status update |
| `filters.call_status.answered` | Call answered |
| `filters.call_status.rejected` | Call rejected |
| `filters.call_status.ringing` | Call ringing |
| `filters.call_permission_update` | Permission response |
| `filters.call_permission_accepted` | Permission accepted |
| `filters.call_permission_rejected` | Permission rejected |

## Account Filters

| Filter | Matches |
|--------|---------|
| `filters.chat_opened` | Chat opened (first msg) |
| `filters.phone_number_change` | User changed phone |
| `filters.identity_change` | Identity changed |
| `filters.user_marketing_preferences` | Marketing pref change |
| `filters.user_marketing_preferences.stop` | Opted out |
| `filters.user_marketing_preferences.resume` | Opted back in |

## Custom Filters

```python
# Sync
def is_premium(wa: WhatsApp, msg: Message) -> bool:
    return msg.from_user.wa_id in PREMIUM_USERS

@wa.on_message(filters.text & is_premium)
def handle(wa, msg): ...

# Async (pywa_async only)
async def is_premium(wa: WhatsApp, msg: Message) -> bool:
    return await check_db(msg.from_user.wa_id)
```
