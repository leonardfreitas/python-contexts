# Adding a New Message Type

Step-by-step for adding support for a new message type within an existing
worker. If the new message type actually belongs to a different bounded
context, don't add it here — start a new worker project instead (see
`architecture.md`'s bounded-context rule).

## 1. Message schema

```python
# app/messages/order_cancelled.py
class OrderCancelledMessage(BaseModel):
    order_id: str
    reason: str | None = None
    cancelled_at: datetime
```

## 2. Domain exceptions (if this message type introduces new failure modes)

```python
# app/exceptions.py
class OrderNotFoundForCancellationError(PermanentError):
    default_message = "Cannot cancel an order that doesn't exist"
```

## 3. Service — the business logic, as a function

```python
# app/services/handle_order_cancelled.py
from aws_lambda_powertools.utilities.idempotency import idempotent_function

@idempotent_function(data_keyword_argument="message", persistence_store=persistence, config=config)
def handle_order_cancelled(message: OrderCancelledMessage) -> None:
    if not repository.order_exists(message.order_id):
        raise OrderNotFoundForCancellationError(message.order_id)

    repository.mark_order_cancelled(message.order_id, message.reason)
```

Remember: wrap with `@idempotent_function` if this service has a
non-idempotent side effect (see `patterns/idempotency.md`).

## 4. Register the message type in the handler's dispatch table

```python
# app/handler.py
MESSAGE_HANDLERS = {
    "OrderCreated": (OrderCreatedMessage, handle_order_created),
    "OrderCancelled": (OrderCancelledMessage, handle_order_cancelled),  # new
}
```

## 5. Add a sample event for local testing

```json
// events/sqs-order-cancelled.json
{
  "Records": [
    {
      "messageId": "22222222-2222-2222-2222-222222222222",
      "body": "{\"type\":\"OrderCancelled\",\"payload\":{\"order_id\":\"order-123\",\"reason\":\"customer request\",\"cancelled_at\":\"2026-01-01T00:00:00Z\"}}",
      "...": "rest of the fields, see patterns/local-development.md for the full shape"
    }
  ]
}
```

## 6. Tests — service first, then handler dispatch

```python
# tests/test_services.py
def test_handle_order_cancelled_raises_if_order_not_found():
    message = OrderCancelledMessage(order_id="does-not-exist", cancelled_at=now())
    with pytest.raises(OrderNotFoundForCancellationError):
        handle_order_cancelled(message)
```

```python
# tests/test_handler.py
def test_handler_dispatches_order_cancelled_to_correct_service(mocker):
    spy = mocker.patch("app.handler.handle_order_cancelled")
    event = {"Records": [{"messageId": "1", "body": json.dumps({"type": "OrderCancelled", "payload": valid_payload})}]}

    handler(event, context=MockLambdaContext())

    spy.assert_called_once()
```

## Checklist

- [ ] New failure modes inherit from `RetryableError` or `PermanentError`, whichever fits
- [ ] Service is a function, wrapped with `@idempotent_function` if it has non-idempotent side effects
- [ ] Service doesn't import anything SQS-specific (no message IDs, no receipt handles)
- [ ] Message type registered in the handler's dispatch table
- [ ] A sample event exists in `events/` for local testing
- [ ] Service has a unit test with no batch/SQS machinery involved
- [ ] Handler dispatch has a test confirming the new message type routes correctly
- [ ] Confirm this message type genuinely belongs to this bounded context (see `architecture.md`) before adding it here
