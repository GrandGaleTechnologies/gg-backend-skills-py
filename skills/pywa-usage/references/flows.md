# Flows Reference

## Sending a Flow

```python
wa.send_flow(
    to="1234567890",
    flow_id="flow_abc",
    flow_action="NAVIGATE",          # or DATA_EXCHANGE
    flow_action_data={
        "screen": "FIRST_SCREEN",
        "data": {"key": "value"},
    },
)
```

## Handling Flow Completion

```python
from pywa.types import FlowCompletion, FlowStatus

@wa.on_flow_completion
def on_flow(wa: WhatsApp, fc: FlowCompletion):
    if fc.status == FlowStatus.COMPLETED:
        user_data = fc.response  # dict of submitted data
    elif fc.status == FlowStatus.DECLINED:
        print("User dismissed the flow")
```

## Flow Request Handler (Lifecycle)

Handle INIT, REFRESH, DATA_EXCHANGE, and PING events:

```python
from pywa.types import FlowRequest, FlowRequestActionType
from pywa.handlers import FlowRequestHandler

@wa.on_flow_request(endpoint="/flow")
def handle(wa: WhatsApp, flow: FlowRequest):
    if flow.action == FlowRequestActionType.INIT:
        return {"data": {"welcome": "Hello!"}}
    elif flow.action == FlowRequestActionType.DATA_EXCHANGE:
        return {"data": {"result": process(flow.data)}}
```

Requires `business_private_key` for encryption/decryption.

## Flow Management

```python
# Create
flow = wa.create_flow(name="signup", category=FlowCategory.SIGN_UP, json=flow_json)

# Update JSON
wa.update_flow_json(flow_id=flow.id, json=updated_json)

# Get details
details = wa.get_flow(flow_id="flow_abc")

# List all
flows = wa.get_flows()  # Paginated Result

# Metrics
metrics = wa.get_flow_metrics(flow_id="flow_abc", granularity=FlowMetricGranularity.DAILY)

# Delete
wa.delete_flow(flow_id="flow_abc")

# Migrate between WABAs
wa.migrate_flows(source_waba_id="old", target_waba_id="new")
```

## FlowJSON Type

The `FlowJSON` type represents the JSON definition of a Flow. Use it when creating or updating Flows programmatically. See the `tests/data/flows/` directory for version-specific examples.
