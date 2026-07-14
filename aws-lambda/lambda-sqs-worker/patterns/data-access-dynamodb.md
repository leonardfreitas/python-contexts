# Data Access — DynamoDB

Use this pattern when the worker persists business data in DynamoDB. If
the project uses a relational database instead, use
`data-access-sql.md`.

Note: this is about the worker's **business data** table(s), separate
from the idempotency table required by `patterns/idempotency.md` — a
project can use DynamoDB for the idempotency table while using SQL for
its own business data, or DynamoDB for both. They're independent
decisions.

## Same foundation as any DynamoDB-backed Lambda context

`boto3.resource` for everyday item operations, keys/GSIs designed around
access patterns identified upfront, `LastEvaluatedKey` always handled for
paginated reads, `batch_writer()` for bulk writes. See the equivalent
pattern file in the `lambda-api` context for the full reasoning — the
underlying DynamoDB principles don't change based on what triggers the
Lambda.

## What's specific to a batch worker

### Batch writes map naturally onto SQS batches

If a single invocation processes multiple messages that all need to
persist data, `batch_writer()` is a natural fit — but remember that a
batch write success/failure is not automatically tied to the per-message
`batchItemFailures` reporting. If a batch write partially fails, the
service function needs to determine which specific messages succeeded
before returning control to the handler, so the handler can report the
correct `messageId`s as failed:

```python
def handle_batch_of_snapshots(messages: list[OrderCreatedMessage]) -> list[str]:
    """Returns the list of order_ids that failed to persist."""
    failed_ids = []
    with table.batch_writer() as batch:
        for message in messages:
            try:
                batch.put_item(Item=message.model_dump())
            except Exception:
                failed_ids.append(message.order_id)
    return failed_ids
```

In practice, most workers process one message per service call (as shown
throughout this context's other pattern files) rather than batching
writes across multiple messages in one service call — that keeps the
mapping between a message and its processing outcome one-to-one, which is
simpler to reason about and matches how `BatchProcessor` already reports
partial failures per message. Only reach for batched writes across
multiple messages when the write volume genuinely requires it.

### Conditional writes for defense-in-depth alongside idempotency

`put_item` with a `ConditionExpression` checking the key doesn't already
exist is a cheap way to add a second layer of protection against
duplicate processing, complementary to (not a replacement for) the
idempotency utility from `patterns/idempotency.md`:

```python
def save_order_snapshot(order_id: str, items: list[dict]) -> None:
    table.put_item(
        Item={"PK": f"ORDER#{order_id}", "SK": "SNAPSHOT", "items": items},
        ConditionExpression="attribute_not_exists(PK)",
    )
```

## Local development

Run DynamoDB Local via Docker Compose, same as any other DynamoDB-backed
context:

```yaml
# docker-compose.yml (local dev only)
services:
  dynamodb-local:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
```

## Summary

| Rule | Reason |
|---|---|
| Default to one message → one service call → one write | Keeps the mapping between a message and its outcome one-to-one, matching per-message batch failure reporting |
| Batch writes across multiple messages only when volume justifies it | Simpler default, added complexity only when needed |
| Conditional writes as defense-in-depth alongside the idempotency utility | Cheap extra protection, not a replacement for `patterns/idempotency.md` |
