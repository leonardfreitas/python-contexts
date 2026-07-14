# Architecture

## The layers

```
SQS (batch event) → Handler (batch orchestration) → Message parser/validator → Service → Data layer
```

| Layer | Responsibility | Does NOT do |
|---|---|---|
| **Handler** | Receives the batch event, orchestrates processing per record via `BatchProcessor`, collects and reports partial failures | Business logic, message-specific validation |
| **Message parser/validator** | Validates and parses the raw SQS message body into a typed schema | Business logic |
| **Service** | Business logic for handling one validated message | Anything SQS-specific (message IDs, receipt handles, batch mechanics) |
| **Data layer** | Persistence — same spirit as any repository layer: queries, writes | Business flow logic |

The separation between Handler and Service matters even more here than in
a synchronous API: the handler knows about SQS (message IDs, receipt
handles, batch mechanics), the service does not know a queue exists at all
— it only receives an already-validated domain object. This is what makes
business logic testable with zero SQS event simulation.

```python
# app/services/handle_order_created.py
def handle_order_created(message: OrderCreatedMessage) -> None:
    if repository.order_exists(message.order_id):
        return  # already processed — defense in depth alongside idempotency

    repository.save_order_snapshot(message.order_id, message.items)
```

## Why a Lambda function exists per bounded context, same rule as any other Lambda context

> A worker Lambda exists per bounded context — the set of message types it
> consumes and the data it owns. Message types belong to the same worker
> when they operate on the same domain model and are mostly owners of the
> same data. They become separate workers when they represent different
> domains that would be designed as distinct services anyway.

Example: a worker consuming `OrderCreated` and `OrderCancelled` events —
both operate on the `Order` aggregate — belongs together. A worker that
also started consuming `NotificationRequested` events would be mixing two
unrelated domains into one deployable unit, and should instead be two
separate worker projects (each with its own queue, its own
`template.yaml`).

## The batch processing model — the core mechanic of this context

A single Lambda invocation triggered by SQS can receive **more than one
message at once** (a batch). Some messages in that batch might succeed,
others might fail — and only the failed ones should go back to the queue.

Without explicit handling, an uncaught exception fails the **entire**
invocation, sending every message in the batch back to the queue —
including ones that already succeeded. This causes unnecessary
reprocessing and, without idempotency (see `patterns/idempotency.md`),
duplicate side effects.

**The required mechanism: partial batch failure reporting.** Each message
is processed independently; failures are collected and reported by
`messageId`, so only those specific messages are retried:

```python
def handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            process_message(record)
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})
    return {"batchItemFailures": failures}
```

In practice, this context uses AWS Lambda Powertools' `BatchProcessor`
utility rather than writing this loop by hand — see
`patterns/batch-processing.md` for the full implementation, and for how
sequential vs parallel processing within a batch is a configuration
choice, not a rewrite.

## Project folder structure (a single worker Lambda / bounded context)

```
order-events-worker/            ← project root = one Lambda = one bounded context
  template.yaml
  samconfig.toml

  scripts/
    deploy.sh
    invoke-local.sh                # runs `sam local invoke` with a sample event

  events/                            # sample SQS event payloads used for local testing
    sqs-batch.json
    sqs-single-message.json

  .env.example                        # committed
  .env.dev                             # gitignored
  .env.staging                          # gitignored
  .env.production                        # gitignored
  .gitignore

  pyproject.toml / requirements.txt

  app/
    handler.py                           # the Lambda entry point, wires up BatchProcessor

    messages/                              # one Pydantic schema per message type consumed
      order_created.py
      order_cancelled.py

    services/                               # business logic, one function per message type handled
      handle_order_created.py
      handle_order_cancelled.py

    exceptions.py                            # WorkerError, RetryableError, PermanentError

    repository.py                             # data layer — SQL or DynamoDB

  tests/
    test_services.py
    test_handler.py
```

## Rules that define this structure

### Rule 1 — One project = one worker Lambda = one bounded context

Same rule as any other Lambda context in this collection: if a second,
unrelated set of message types needs handling, it becomes a **separate
project root**, with its own `template.yaml`, its own queue, its own
`app/` — not a second consumer bolted onto the same project.

### Rule 2 — `messages/` holds one schema per message type, not one shared "envelope" schema

Each message type this worker consumes gets its own Pydantic model. This
mirrors the "split by intent" principle used for API schemas elsewhere:
`OrderCreatedMessage` and `OrderCancelledMessage` are different contracts,
even if they happen to share some fields.

### Rule 3 — `services/` has one function per message type handled, never a single dispatcher function with embedded business logic

```python
# ✅ one function per message type, dispatch is a thin lookup, not embedded logic
services/
  handle_order_created.py
  handle_order_cancelled.py
```

The handler's `record_handler` function is the dispatcher (see
`patterns/batch-processing.md`); it looks up which service function to
call based on the message type, it doesn't contain the business logic
itself.

### Rule 4 — `events/` exists specifically to support local development

There is no equivalent of "start a local server and curl it" for a queue
consumer. Sample event JSON files are what `sam local invoke` uses instead
— see `patterns/local-development.md`.

### Rule 5 — `repository.py` follows the same single-file-until-justified rule

Starts as a single file, only becomes a `repository/` folder once the
bounded context genuinely persists more than one entity.
