# AWS Lambda SQS Worker Context

You are a senior engineer building an asynchronous message-processing
worker deployed as an AWS Lambda function consuming from SQS, using AWS
SAM. Follow all the contexts below before generating any code.

This context is fully self-contained. It does not reference, depend on,
or assume the presence of any other context in this collection (including
the `lambda-api` context) — it can be adopted on its own, in a project
with no relationship to any other context here.

This is the entry point. Read in this order:

1. `architecture.md` — layers, bounded-context-per-Lambda rule, batch processing model, folder structure
2. `stack.md` — required libraries and why
3. `conventions.md` — naming, message/service pattern
4. `rules.md` — what is FORBIDDEN, with ❌ / ✅ examples
5. `patterns/` — step-by-step recipes for recurring tasks

## Core philosophy

SQS delivers messages with an **at-least-once** guarantee — the same
message can be delivered more than once, with no error involved, simply
as a property of the queue. A Lambda invocation triggered by SQS can also
receive a **batch** of messages at once, where some may succeed and others
fail independently. Both of these are fundamentally different from a
synchronous HTTP request/response model, and they drive every architecture
decision in this context: idempotency is mandatory, partial batch failure
must be reported precisely, and a message's fate (retry vs permanent
failure) must be an explicit decision, never an accident of an uncaught
exception.

If you are generating code and business logic appears to be leaking into
the handler (SQS-specific concerns like message IDs or receipt handles
mixed into decision logic) — stop and move it into `services/`. The
handler orchestrates the batch; it does not decide business outcomes.

## Patterns index

- `patterns/new-message-handler.md` — step-by-step to add a new message type to process
- `patterns/batch-processing.md` — `BatchProcessor`, partial batch failure, sequential vs parallel
- `patterns/idempotency.md` — why it's mandatory, how to configure it
- `patterns/error-handling.md` — `RetryableError` vs `PermanentError`, DLQ, redrive policy
- `patterns/testing.md` — testing pyramid, simulating batch events, mocking AWS
- `patterns/deploy.md` — deploy script, SAM template, queue/DLQ configuration
- `patterns/local-development.md` — `sam local invoke`, sample event payloads
- `patterns/data-access-sql.md` — SQL-specific data access
- `patterns/data-access-dynamodb.md` — DynamoDB-specific data access

## Choosing a data-access pattern

Read the pattern that matches your project's persistence choice:

- Using a relational database → `patterns/data-access-sql.md`
- Using DynamoDB for business data → `patterns/data-access-dynamodb.md`

(Note: DynamoDB is used for the idempotency store regardless of this
choice — see `patterns/idempotency.md`. That's a separate, required table,
not related to which pattern file you read for your own business data.)

## Structure of this context

```
lambda-sqs-worker/
  SKILL.md                     → this file, entry point + index
  architecture.md                → layers, bounded-context rule, batch model, folder structure
  stack.md                         → required libraries and why
  conventions.md                     → naming and code patterns
  rules.md                             → what is FORBIDDEN
  patterns/
    new-message-handler.md
    batch-processing.md
    idempotency.md
    error-handling.md
    testing.md
    deploy.md
    local-development.md
    data-access-sql.md
    data-access-dynamodb.md
```
