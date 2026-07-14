# Conventions

## Services: functions, one per message type

Same default as every other context in this collection: a function, not a
class, unless there's a genuine need to hold configurable state across
calls (e.g. an injected downstream client).

```
services/
  handle_order_created.py     # handle_order_created(message: OrderCreatedMessage) -> None
  handle_order_cancelled.py     # handle_order_cancelled(message: OrderCancelledMessage) -> None
```

A service function receives an already-validated message object, never a
raw SQS record — parsing/validation happens one layer up, in the handler,
before the service is called.

## Messages: one Pydantic schema per message type

```python
# app/messages/order_created.py
class OrderCreatedMessage(BaseModel):
    order_id: str
    customer_id: str
    items: list[OrderItemPayload]
    created_at: datetime
```

Naming convention: `<EventName>Message`, matching the event/message name a
producer would use, not the internal domain model name — the message
schema documents the wire contract, which may reasonably differ from how
the domain models the same concept internally.

## Message type routing

The handler's dispatch logic maps a message type identifier (commonly a
field in the message body, or an SQS message attribute) to the
corresponding service function. Keep this mapping explicit and centralized
in one place — not a chain of `if/elif` scattered through the handler:

```python
# app/handler.py
MESSAGE_HANDLERS = {
    "OrderCreated": (OrderCreatedMessage, handle_order_created),
    "OrderCancelled": (OrderCancelledMessage, handle_order_cancelled),
}
```

## Exceptions: `RetryableError` vs `PermanentError`, not HTTP-style categories

Unlike an HTTP-facing context, exceptions here are not categorized by
"what status code to return" — there is no client waiting for a status
code. They're categorized by **what should happen to the message**:
retried, or given up on. See `patterns/error-handling.md` for the full
hierarchy.

## Environment variables

Same principle as every other context here: no `os.environ.get()`
scattered through the codebase. Centralize configuration access in a
single settings module with explicit types and defaults.
