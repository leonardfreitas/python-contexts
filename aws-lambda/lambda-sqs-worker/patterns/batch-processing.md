# Batch Processing

## The problem this solves

A single Lambda invocation triggered by SQS can receive multiple messages
at once (a batch). If one message fails and the code lets an exception
propagate uncaught, the **entire invocation** is marked as failed and
**every message in the batch** goes back to the queue — including ones
that already succeeded. This causes unnecessary reprocessing, and without
idempotency (see `patterns/idempotency.md`), duplicate side effects for
messages that didn't need to be retried at all.

## The solution: `BatchProcessor` + `process_partial_response`

```python
# app/handler.py
from aws_lambda_powertools.utilities.batch import BatchProcessor, EventType, process_partial_response
from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.utilities.batch.types import BatchTypeModels

processor = BatchProcessor(event_type=EventType.SQS)

MESSAGE_HANDLERS = {
    "OrderCreated": (OrderCreatedMessage, handle_order_created),
    "OrderCancelled": (OrderCancelledMessage, handle_order_cancelled),
}

def record_handler(record):
    body = json.loads(record.body)
    message_type = body["type"]
    schema, service_fn = MESSAGE_HANDLERS[message_type]
    message = schema.model_validate(body["payload"])
    service_fn(message)

def handler(event, context: LambdaContext):
    return process_partial_response(
        event=event,
        record_handler=record_handler,
        processor=processor,
        context=context,
    )
```

`process_partial_response` handles the per-record try/except and returns
a correctly shaped `{"batchItemFailures": [...]}` response automatically —
you write `record_handler` as if it were processing one message at a
time, and the utility handles the batch mechanics.

## Required SAM configuration

The SQS event source mapping must explicitly enable partial batch
response reporting, or the `batchItemFailures` response is silently
ignored and the old all-or-nothing behavior applies:

```yaml
Events:
  SQSEvent:
    Type: SQS
    Properties:
      Queue: !GetAtt MainQueue.Arn
      BatchSize: 10
      FunctionResponseTypes:
        - ReportBatchItemFailures
```

> `FunctionResponseTypes: [ReportBatchItemFailures]` is required. Without
> it, returning `batchItemFailures` from the handler has no effect — SQS
> silently falls back to treating the whole batch as either fully
> succeeded or fully failed.

## Sequential vs parallel processing within a batch

`BatchProcessor` defaults to sequential processing. Parallel processing is
available via `max_workers`:

```python
processor = BatchProcessor(event_type=EventType.SQS, max_workers=4)
```

### Default: sequential

- ✅ Simpler, no risk of a race condition between two messages in the
  same batch touching the same data
- ✅ Safe on any queue type, including FIFO
- ❌ Slower for a batch containing many independent, slow-to-process
  messages

### Parallel — opt-in when volume justifies it

- ✅ Higher throughput for batches of independent, I/O-bound messages
  (e.g. each message triggers a separate external API call)
- ❌ **Unsafe on a FIFO queue** if messages sharing the same
  `MessageGroupId` can appear in the same batch — parallelizing breaks the
  ordering guarantee that's the entire reason to choose FIFO. Only enable
  parallel processing on FIFO if you can guarantee messages from the same
  group never land in the same batch (uncommon in practice), or if
  ordering genuinely doesn't matter within a group despite using FIFO for
  its deduplication behavior
- ❌ Adds complexity: services processed in parallel must be safe to run
  concurrently against shared state (the data layer, external APIs)

> Rule: start every new project with sequential processing. Switch to
> parallel only once real throughput measurements justify it, and never on
> a FIFO queue without confirming the ordering guarantee isn't needed
> within a batch.

## Standard vs FIFO — what changes in this pattern

| | Standard | FIFO |
|---|---|---|
| Ordering guarantee | None | Guaranteed within the same `MessageGroupId` |
| Deduplication | None (at-least-once, duplicates possible) | Native, 5-minute window — idempotency (`patterns/idempotency.md`) is still recommended as defense in depth |
| Safe to parallelize within a batch | Yes | Only if messages from the same `MessageGroupId` never co-occur in a batch |

The `BatchProcessor` code above works identically for both queue types —
the difference is entirely in the SAM template (`QueueName` must end in
`.fifo`, `FifoQueue: true`) and in the sequential-vs-parallel decision
above.
