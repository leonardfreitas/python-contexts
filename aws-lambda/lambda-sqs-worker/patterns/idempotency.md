# Idempotency

## Why this is mandatory, not optional

SQS guarantees **at-least-once** delivery — a message can be delivered
more than once with no error involved anywhere in the system, simply as a
normal property of the queue (network retries, visibility timeout
expiring before processing finishes, etc.). If `handle_order_created`
charges a payment or increments a counter, processing the same message
twice causes a real business bug — not a hypothetical edge case, an
expected occurrence under normal operation.

> Every service function that has a non-idempotent side effect (writes
> that aren't naturally safe to repeat, external calls that aren't
> naturally safe to repeat) MUST be wrapped with idempotency handling.
> This applies even on a FIFO queue — FIFO's deduplication window is 5
> minutes and is not a substitute for application-level idempotency over
> a message's full retry lifetime.

## Implementation with Powertools

```python
# app/services/handle_order_created.py
from aws_lambda_powertools.utilities.idempotency import (
    idempotent_function,
    DynamoDBPersistenceLayer,
    IdempotencyConfig,
)

persistence = DynamoDBPersistenceLayer(table_name=settings.idempotency_table_name)
config = IdempotencyConfig(expires_after_seconds=86400)  # 24h, tune per project

@idempotent_function(
    data_keyword_argument="message",
    persistence_store=persistence,
    config=config,
)
def handle_order_created(message: OrderCreatedMessage) -> None:
    repository.save_order_snapshot(message.order_id, message.items)
```

The decorator hashes the `message` argument to derive an idempotency key,
checks the DynamoDB table for a prior successful execution with that key,
and short-circuits (returning the previously recorded result) if found —
the function body never runs twice for the same message content.

## Required infrastructure: the idempotency table

```yaml
# template.yaml
IdempotencyTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: !Sub "${ProjectName}-idempotency-${EnvType}"
    AttributeDefinitions:
      - AttributeName: id
        AttributeType: S
    KeySchema:
      - AttributeName: id
        KeyType: HASH
    BillingMode: PAY_PER_REQUEST
    TimeToLiveSpecification:
      AttributeName: expiration
      Enabled: true
```

TTL is required — idempotency records should expire, not accumulate
indefinitely. `expires_after_seconds` in `IdempotencyConfig` should
reasonably exceed the queue's `maxReceiveCount` × `VisibilityTimeout`
window, so a legitimately-retried message is still recognized as a
duplicate throughout its full retry lifetime.

## What the idempotency key should be

Default to the full validated message body (what `data_keyword_argument`
points at). If the message includes a natural, stable business identifier
(an order id, a request id supplied by the producer), consider scoping
the key to just that identifier instead of the full payload — this
protects against duplicates even if a producer resends the same logical
event with a slightly different payload shape (e.g. an added field).

## Idempotency vs "check if it already exists" as defense in depth

Powertools' idempotency utility is the primary mechanism, but a cheap
existence check inside the service function is a reasonable second layer,
not a replacement:

```python
def handle_order_created(message: OrderCreatedMessage) -> None:
    if repository.order_exists(message.order_id):
        return  # already processed — belt and suspenders alongside the idempotency decorator

    repository.save_order_snapshot(message.order_id, message.items)
```

This is optional and depends on how cheap the existence check is relative
to the cost of relying on the idempotency table alone — it is not a
substitute for the decorator, since it can't protect against concurrent
duplicate invocations racing each other the way the idempotency table's
locking behavior can.

## Summary

| Rule | Reason |
|---|---|
| Every service with a non-idempotent side effect uses `@idempotent_function` | SQS at-least-once delivery makes duplicate delivery a normal occurrence, not an edge case |
| A dedicated DynamoDB table with TTL enabled backs the idempotency store | Required infrastructure for the utility to function; TTL prevents unbounded growth |
| `expires_after_seconds` exceeds the queue's full retry window | Ensures a legitimately-retried message is still recognized throughout its retry lifetime |
| FIFO's native deduplication does not replace application-level idempotency | FIFO's 5-minute window is shorter than a message's full retry lifetime |
