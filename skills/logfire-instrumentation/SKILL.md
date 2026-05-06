---
name: logfire-instrumentation
description: 'Instrument Python code with Pydantic Logfire for observability. Use when: adding logging, tracing, spans, instrumentation, monitoring, observability, logfire, @instrument decorator, logfire.info, logfire.error, logfire.warning, logfire.debug.'
---

# Logfire Instrumentation

## When to Use

- Adding observability/tracing to new or existing functions
- Instrumenting service functions, external API clients, cache operations, or route handlers
- Adding structured logging (`info`, `error`, `warning`, `debug`) to code
- Setting up Logfire in the application entrypoint
- User asks to "add logfire", "add observability", "add tracing", or "add monitoring"
- User wants to monitor AI/LLM calls (PydanticAI, OpenAI, Anthropic)
- User asks to add observability to an AI agent or LLM pipeline

## Package

```
logfire[fastapi,psycopg2,pymongo]>=3.16.1
```

Install with extras matching detected frameworks. Each instrumented library needs its corresponding extra — without it, `instrument_*()` will fail at runtime with a missing dependency error.

```bash
uv add 'logfire[fastapi,httpx,asyncpg]'
```

Full list of available extras: `fastapi`, `starlette`, `django`, `flask`, `httpx`, `requests`, `asyncpg`, `psycopg`, `psycopg2`, `sqlalchemy`, `redis`, `pymongo`, `mysql`, `sqlite3`, `celery`, `aiohttp`, `aws-lambda`, `system-metrics`, `litellm`, `dspy`, `google-genai`.

## Import Convention

Import specific functions from `logfire` — do **not** import the module itself in service/utility files:

```python
from logfire import info, error, instrument, warning, debug
```

Only import `logfire` directly in the application entrypoint (`app/main.py`) where `logfire.configure()` and auto-instrumentation are called.

## How Logfire Works

Logfire is an observability platform built on OpenTelemetry. It captures traces, logs, and metrics from applications. Logfire has native SDKs for Python, JavaScript/TypeScript, and Rust, plus support for any language via OpenTelemetry.

The reason this skill exists is that Claude tends to get a few things subtly wrong with Logfire — especially the ordering of `configure()` vs `instrument_*()` calls, the structured logging syntax, and which extras to install. These matter because a misconfigured setup silently drops traces.

## Step 1: Detect Language and Frameworks

Identify the project language and instrumentable libraries:

- **Python**: Read `pyproject.toml` or `requirements.txt`. Common instrumentable libraries: FastAPI, httpx, asyncpg, SQLAlchemy, psycopg, Redis, Celery, Django, Flask, requests, PydanticAI.
- **JavaScript/TypeScript**: Read `package.json`. Common frameworks: Express, Next.js, Fastify. Also check for Cloudflare Workers or Deno.
- **Rust**: Read `Cargo.toml`.

Then follow the language-specific steps below.

---

## Python

### Application-Level Setup

`logfire.configure()` initializes the SDK and must come before everything else. The `instrument_*()` calls register hooks into each library. If you call `instrument_*()` before `configure()`, the hooks register but traces go nowhere.

Configure Logfire conditionally in `app/main.py` based on the `LOGFIRE_TOKEN` environment variable:

```python
import logfire

if settings.LOGFIRE_TOKEN:
    logfire.configure(
        service_name="marketmachine-api",
        token=settings.LOGFIRE_TOKEN,
        environment=settings.ENV,
    )
    logfire.instrument_fastapi(app)
    logfire.instrument_psycopg()
    logfire.instrument_pymongo()
```

Do **not** duplicate this configuration elsewhere. Auto-instrumentation for FastAPI, PostgreSQL, and MongoDB is handled here.

Placement rules:
- `logfire.configure()` goes in the application entry point — call it **once per process**
- `instrument_*()` calls go right after `configure()`
- Web framework instrumentors (`instrument_fastapi`, `instrument_flask`, `instrument_django`) need the app instance as an argument. HTTP client and database instrumentors (`instrument_httpx`, `instrument_asyncpg`) are global and take no arguments.
- In **Gunicorn** deployments, call `logfire.configure()` inside the `post_fork` hook, not at module level — each worker is a separate process

### The `@instrument()` Decorator

Use `@instrument()` on service functions, external client methods, cache operations, and route handlers to create named spans.

**Import:**
```python
from logfire import instrument
```

**Naming convention:**
- Use a **descriptive, human-readable string** that describes the operation
- Use **Title Case**
- Include dynamic context with `{variable}` placeholders referencing function parameters
- The decorator MUST be the innermost decorator (closest to `def`) — under any `@router.*` decorator

**Service functions:**
```python
@instrument("Connect Organization Card")
async def connect_organization_card(user: user_models.User, db: Session):
    ...

@instrument("Generate Organization Invoice")
async def generate_organization_invoice(org: org_models.Organization, db: Session):
    ...
```

**Dynamic span names using function parameters:**
```python
@instrument("Get Shipping Rate - Source {data.source.source_type}")
async def get_shipping_rate(data: RateRequestData, db: Session):
    ...

@instrument("Void Label {id}")
async def void_label(id: int, db: Session):
    ...
```

**External API clients:**
```python
@instrument("Getting shipping rate from ShipStation")
async def get_rates(self, data: schemas.RateRequestSchema, ...):
    ...

@instrument("QuickBooks Client")
async def make_request(self, method: str, url: str, ...):
    ...
```

**Cache operations:**
```python
@instrument("Get cached data from Redis")
async def get(self, data: dict, cache_prefix: str) -> T | None:
    ...

@instrument("Set cached data in Redis")
async def set(self, data: dict, cache_prefix: str, value: Any) -> None:
    ...
```

**Route handlers:**
```python
@router.post("", ...)
@instrument("Logging user client error")
async def route_user_error_log(error_in: create.UserClientErrorLog, _: CurrentUser):
    ...
```

**Disable arg capture** (for large objects or sensitive data):
```python
@instrument("Export Report", extract_args=False)
def export_report(report: HugeReport):
    ...
```

### Structured Logging Calls

Use `info()`, `error()`, `warning()`, and `debug()` for structured log messages within functions.

```python
from logfire import info, error, warning, debug
```

**Guidelines:**
- Use **f-strings** for dynamic content
- Keep messages concise and descriptive
- Use `_tags` parameter for categorization when appropriate
- Place logging calls at meaningful points: before/after key operations, on errors, on cache hits/misses
- **Don't log success after an action log if no new information is gained.** If an `info()` says "Checking X" or "Verifying Y" and execution continues past the failure branch, success is implicit. Only add a follow-up log if it carries new information (e.g. a resolved ID or computed value that wasn't known before the call).

```python
# WRONG — redundant success log adds no new information
info(f"Verifying OTP: {phone}")
if not twilio_client.verify_otp(phone, otp):
    warning(f"Invalid OTP: {phone}")
    raise BadRequest(...)
info(f"OTP verified: {phone}")  # ❌ redundant — implied by not raising

# RIGHT — omit it, the absence of a warning is the signal
info(f"Verifying OTP: {phone}")
if not twilio_client.verify_otp(phone, otp):
    warning(f"Invalid OTP: {phone}")
    raise BadRequest(...)

# RIGHT — follow-up log is justified because it surfaces a newly resolved value
info(f"Loading user: {user_id}")
user = await crud_user.get(id=user_id)
if not user:
    warning(f"User not found: {user_id}")
    raise NotFound(...)
info(f"User loaded: id={user.id} business={user.business_id}")  # ✅ new info
```

```python
# Informational
info(f"Looking for cache key: {cache_key}")
info("Log successful")
info("Application in debug mode, printing to console...")

# With tags
info(f"Triggering error {trigger_error}", _tags=["deathnote"])

# Warnings
warning(f"Cache key {cache_key} not found")

# Errors (in exception handlers)
error(f"Stripe API error: {str(e)}")
error(f"Redis connection failed: {str(e)}")

# Debug (verbose/low-level details)
debug(f"Rate request payload: {data}")
```

For exceptions, use `logfire.exception()` which automatically captures the traceback:

```python
try:
    await process_order(order_id)
except Exception:
    logfire.exception("Failed to process order {order_id}", order_id=order_id)
    raise
```

### Spans - Full Reference

**Context manager** — any spans or logs created inside the `with` block become children of the span. The span records duration automatically.

```python
with logfire.span("Processing order {order_id}", order_id=order_id) as span:
    result = do_work()
    span.set_attribute("result", result)   # attach data after span starts
    span.message = f"Processed order {order_id}: {result}"  # override displayed message
```

**Setting attributes after start:**
```python
with logfire.span("Calculating...") as span:
    result = expensive_computation()
    span.set_attribute("result", result)
```

**Setting span level dynamically** — useful when the outcome determines severity:
```python
with logfire.span("Sending notification") as span:
    success = send_notification()
    if not success:
        span.set_level("error")   # upgrades the span to error after the fact
```

**Span names should be low cardinality.** The first argument is both the `span_name` (indexed, filterable) and a format template for the displayed message. Never concatenate dynamic values directly:
```python
# Good - span_name is 'Processing {entity_type}', searchable
logfire.info("Processing {entity_type}", entity_type=entity_type)

# Bad - span_name is unique per entity, can't filter efficiently
logfire.info("Processing " + entity_type)
```

Use `_span_name` to decouple the span name from the message template:
```python
logfire.info("Hello {name}", name="world", _span_name="greeting")
# span_name = 'greeting', message = 'Hello world'
```

**f-strings** work in Python 3.11+ — Logfire uses AST inspection to extract attribute names:
```python
logfire.info(f"Hello {name}")  # equivalent to logfire.info("Hello {name}", name=name)
```
Avoid f-strings with expensive expressions or side effects. Disable with `logfire.configure(inspect_arguments=False)`.

### Exception and Error Handling

**Unhandled exceptions in a span** are automatically recorded with a traceback and the span level is set to `error`:
```python
with logfire.span("Risky operation"):
    raise ValueError("something went wrong")  # auto-recorded, span becomes error
```

**Handled exceptions** are not auto-recorded. Use `span.record_exception()` to capture them without re-raising:
```python
with logfire.span("Best-effort operation") as span:
    try:
        risky_call()
    except RecoverableError as e:
        span.record_exception(e)  # records traceback but does NOT set span to error level
```

**Logging exceptions without a span** — `logfire.exception()` is a shorthand for `logfire.error(..., _exc_info=True)`:
```python
try:
    process_payment(order_id)
except PaymentError:
    logfire.exception("Payment failed for order {order_id}", order_id=order_id)
    raise

# Equivalent explicit form - useful for non-error levels
logfire.warn("Retrying after error", _exc_info=True)

# Attach a specific exception object (not the one being handled)
logfire.error("Failed", _exc_info=caught_exception)
```

### `@instrument` Decorator — Additional Notes

See the full `@instrument()` usage guide above. Additional technical notes:

- By default adds all function arguments as attributes as span data
- The decorator must be applied **under** any other decorators (closest to `def`)
- The source code must be accessible at runtime (no `.pyc`-only deployments)
- Custom span name with argument formatting: `@instrument("Transform {x=} and {y=}")`

### Auto-Tracing

Automatically instruments every function call in specified modules — no manual span wrapping needed.

`install_auto_tracing` must be called **before** the target modules are imported.

```python
# main.py (outside the app package)
import logfire

logfire.configure()
logfire.install_auto_tracing(modules=["app"], min_duration=0.01)  # trace functions >10ms

from app.main import main
main()
```

**Key behaviours:**
- `min_duration=0` traces everything; `min_duration=0.01` only traces functions that took >10ms on a prior run
- A function starts being traced only after it first exceeds `min_duration` — the first slow call is not captured
- Generator functions are not traced

**Exclude specific functions:**
```python
@logfire.no_auto_trace   # zero runtime overhead, detected at import time
def hot_path_function():
    ...

@logfire.no_auto_trace   # all methods excluded
class InternalHelper:
    def method(self): ...
```

**Custom module filter:**
```python
import pathlib
PYTHON_LIB_ROOT = str(pathlib.Path(pathlib.__file__).parent)

logfire.install_auto_tracing(
    lambda m: not m.filename.startswith(PYTHON_LIB_ROOT),
    min_duration=0,
)
```

### Metrics

Create metrics once at module level, then record/increment in hot paths.

```python
# Counter — monotonically increasing (requests, errors, events)
requests_total = logfire.metric_counter("requests_total", unit="1", description="HTTP requests received")
requests_total.add(1)

# Histogram — distribution of values (latency, file sizes)
request_duration = logfire.metric_histogram("request_duration", unit="ms", description="Request latency")
request_duration.record(duration_ms)

# Up-Down Counter — can go up or down (queue depth, active connections)
active_connections = logfire.metric_up_down_counter("active_connections", unit="1")
active_connections.add(1)   # on connect
active_connections.add(-1)  # on disconnect

# Gauge — current snapshot value (temperature, memory usage)
memory_usage = logfire.metric_gauge("memory_bytes", unit="By")
memory_usage.set(get_current_memory())
```

**Callback metrics** — automatically reported every 60 seconds:
```python
from opentelemetry.metrics import CallbackOptions, Observation

def active_user_count(options: CallbackOptions):
    yield Observation(query_active_users(), {})

logfire.metric_gauge_callback("active_users", unit="1", callbacks=[active_user_count])
```

**System metrics** (CPU, memory, disk) — just install and call:
```python
# uv add 'logfire[system-metrics]'
logfire.instrument_system_metrics()
```

### Log Levels

Available log methods: `trace`, `debug`, `info`, `notice`, `warn`, `error`, `fatal`.

```python
logfire.trace("Verbose detail {detail}", detail=x)
logfire.debug("Debug info {val}", val=v)
logfire.info("User {user_id} logged in", user_id=uid)
logfire.warn("Slow query took {ms}ms", ms=elapsed)
logfire.error("Payment declined for {order_id}", order_id=oid)
logfire.fatal("Database unreachable")

# Variable level
logfire.log("warn", "Rate limit approaching {pct}%", pct=usage)

# Spans are info level by default; override with _level
with logfire.span("Background sync", _level="debug"):
    ...
```

**Filter minimum level** to reduce noise/cost:
```python
logfire.configure(min_level="info")  # drops trace and debug spans/logs
logfire.configure(console=logfire.ConsoleOptions(min_log_level="debug"))  # console only
```

### Best Practices

**Span design:**
- Create spans at operation boundaries (handle request, process job, call external API) — not around every small function
- Use `@logfire.instrument` for service/business-logic functions; use `with logfire.span(...)` for inline blocks
- Attach outcome data as attributes (`span.set_attribute`) rather than in the message
- Keep span names low-cardinality — use `{placeholder}` for variable parts, not string concatenation

**Error logging:**
- Use `logfire.exception(...)` in `except` blocks — it attaches the full traceback automatically
- Re-raise after logging unless you genuinely handle the error
- Use `span.record_exception(e)` for caught-and-recovered errors inside spans
- Upgrade span level with `span.set_level("error")` when a business-logic failure occurs even without an exception

**Structured attributes:**
- Always pass dynamic data as keyword args: `logfire.info("msg {key}", key=val)` not `logfire.info(f"msg {val}")`
- Names must not start with `_` (reserved for logfire control args like `_level`, `_span_name`, `_exc_info`)
- Group related context into a span rather than repeating attributes on every log inside it

**Performance:**
- Create metric instruments once at module level, not inside request handlers
- Use `min_duration` with `install_auto_tracing` to avoid tracing every tiny function
- Decorate hot-path functions with `@logfire.no_auto_trace` when using auto-tracing
- Disable `inspect_arguments` for `.pyc`-only deployments or extreme performance sensitivity

**Sensitive data:**
- Never log passwords, tokens, PII as attributes
- Use `logfire.configure(scrubbing=...)` or an OTel collector scrubbing rule to strip sensitive fields automatically

### AI/LLM Instrumentation

Logfire auto-instruments AI libraries to capture LLM calls, token usage, tool invocations, and agent runs.

```bash
uv add 'logfire[pydantic-ai]'
# or: uv add 'logfire[openai]' / uv add 'logfire[anthropic]'
```

Available AI extras: `pydantic-ai`, `openai`, `anthropic`, `litellm`, `dspy`, `google-genai`.

```python
logfire.configure()
logfire.instrument_pydantic_ai()  # captures agent runs, tool calls, LLM request/response
# or:
logfire.instrument_openai()       # captures chat completions, embeddings, token counts
logfire.instrument_anthropic()    # captures messages, token usage
```

For PydanticAI, each agent run becomes a parent span containing child spans for every tool call and LLM request.

---

## Patterns Summary

| Context | What to Do |
|---------|----------|
| New service function | Add `@instrument("Descriptive Name")` decorator |
| New external API client method | Add `@instrument("Service Name - Operation")` decorator |
| New cache operation | Add `@instrument("Cache verb + target")` decorator |
| Error handling / except blocks | Add `error(f"what failed: {str(e)}")` |
| Key operation milestones | Add `info("what happened")` |
| Conditional/debug paths | Add `debug(f"details: {value}")` |
| Middleware or tagged contexts | Use `_tags=["category"]` parameter |

## Do NOT

- Import `logfire` as a module in service files — import specific functions instead
- Call `logfire.configure()` outside of `app/main.py`
- Add `@instrument()` to trivial one-liner utility functions
- Use `print()` for logging in production code — use logfire calls instead
- Add instrumentation to CRUD/database query functions — PostgreSQL is auto-instrumented via `instrument_psycopg()`

---

## JavaScript / TypeScript

### Install

```bash
# Node.js
npm install @pydantic/logfire-node

# Cloudflare Workers
npm install @pydantic/logfire-cf-workers logfire

# Next.js / generic
npm install logfire
```

### Configure

**Node.js (Express, Fastify, etc.)** - create an `instrumentation.ts` loaded before your app:

```typescript
import * as logfire from '@pydantic/logfire-node'
logfire.configure()
```

Launch with: `node --require ./instrumentation.js app.js`

The SDK auto-instruments common libraries when loaded before the app. Set `LOGFIRE_TOKEN` in your environment or pass `token` to `configure()`.

**Cloudflare Workers** - wrap your handler with `instrument()`:

```typescript
import { instrument } from '@pydantic/logfire-cf-workers'

export default instrument(handler, {
  service: { name: 'my-worker', version: '1.0.0' }
})
```

**Next.js** - set environment variables for OpenTelemetry export:

```
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://logfire-api.pydantic.dev/v1/traces
OTEL_EXPORTER_OTLP_HEADERS=Authorization=<your-write-token>
```

### Structured Logging (JS/TS)

```typescript
// Structured attributes as second argument
logfire.info('Created user', { user_id: uid })
logfire.error('Payment failed', { amount: 100, currency: 'USD' })

// Spans
logfire.span('Processing order', { order_id }, {}, async () => {
  logfire.info('Processing step completed')
})

// Error reporting
logfire.reportError('order processing', error)
```

Log levels: `trace`, `debug`, `info`, `notice`, `warn`, `error`, `fatal`.

---

## Rust

### Install

```toml
[dependencies]
logfire = "0.6"
```

### Configure

```rust
let shutdown_handler = logfire::configure()
    .install_panic_handler()
    .finish()?;
```

Set `LOGFIRE_TOKEN` in your environment or use the Logfire CLI to select a project.

### Structured Logging (Rust)

The Rust SDK is built on `tracing` and `opentelemetry` - existing `tracing` macros work automatically.

```rust
// Spans
logfire::span!("processing order", order_id = order_id).in_scope(|| {
    // traced code
});

// Events
logfire::info!("Created user {user_id}", user_id = uid);
```

Always call `shutdown_handler.shutdown()` before program exit to flush data.

---

## Verify

After instrumentation, verify the setup works:

1. Run `logfire auth` to check authentication (or set `LOGFIRE_TOKEN`)
2. Start the app and trigger a request
3. Check https://logfire.pydantic.dev/ for traces

If traces aren't appearing: check that `configure()` is called before `instrument_*()` (Python), check that `LOGFIRE_TOKEN` is set, and check that the correct packages/extras are installed.

## References

Detailed patterns and integration tables, organized by language:

- **Python**: `${CLAUDE_PLUGIN_ROOT}/skills/instrumentation/references/python/logging-patterns.md` (log levels, spans, stdlib integration, metrics, capfire testing) and `${CLAUDE_PLUGIN_ROOT}/skills/instrumentation/references/python/integrations.md` (full instrumentor table with extras)
- **JavaScript/TypeScript**: `${CLAUDE_PLUGIN_ROOT}/skills/instrumentation/references/javascript/patterns.md` (log levels, spans, error handling, config) and `${CLAUDE_PLUGIN_ROOT}/skills/instrumentation/references/javascript/frameworks.md` (Node.js, Cloudflare Workers, Next.js, Deno setup)
- **Rust**: `${CLAUDE_PLUGIN_ROOT}/skills/instrumentation/references/rust/patterns.md` (macros, spans, tracing/log create integration, async, shutdown)
