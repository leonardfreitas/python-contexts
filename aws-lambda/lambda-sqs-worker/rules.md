# Rules — What is FORBIDDEN

An executive summary of every "hard" rule scattered across the other
files. For a quick check, start here; for the full reasoning behind each
rule, see the referenced file.

## Bounded context (`architecture.md`)

- ❌ Multiple unrelated message types/domains handled by the same worker
  "just because they're small"
- ❌ A `services/` function in this worker importing code directly from
  an unrelated domain — cross-context communication happens via a
  separate event/message, not a direct code import

## Layers (`architecture.md`)

- ❌ Business logic inside `handler.py`
- ❌ A service function receiving a raw SQS record instead of an
  already-validated message object
- ❌ SQS-specific concerns (message IDs, receipt handles) referenced
  inside `services/`

## Batch processing (`patterns/batch-processing.md`)

- ❌ Letting an unhandled exception propagate out of the handler for the
  whole batch — every message must be processed independently with
  failures collected and reported by `messageId`
- ❌ Deploying an SQS event source mapping without
  `FunctionResponseTypes: [ReportBatchItemFailures]` — without it,
  returning `batchItemFailures` has no effect
- ❌ Enabling parallel processing (`max_workers`) on a FIFO queue without
  confirming messages from the same `MessageGroupId` never co-occur in a
  batch — this breaks the ordering guarantee FIFO exists to provide
- ❌ Defaulting a new project to parallel processing — sequential is the
  default, parallel is an opt-in once volume justifies it

## Idempotency (`patterns/idempotency.md`)

- ❌ A service function with a non-idempotent side effect that isn't
  wrapped with `@idempotent_function`
- ❌ Relying on FIFO's native deduplication window as a substitute for
  application-level idempotency — the 5-minute window is shorter than a
  message's full retry lifetime
- ❌ An idempotency DynamoDB table without TTL enabled

## Error handling (`patterns/error-handling.md`)

- ❌ Silently swallowing an exception inside message processing and
  treating it as success — this causes data loss, since the message gets
  acknowledged and removed from the queue without being correctly handled
- ❌ A main SQS queue deployed without an associated DLQ and
  `maxReceiveCount`
- ❌ `VisibilityTimeout` set lower than the Lambda function's timeout
  (should be at least 6x, per AWS guidance) — risks duplicate concurrent
  processing of the same message

## Testing (`patterns/testing.md`)

- ❌ Tests making real AWS or real SQS calls — use `moto`
- ❌ A local database or DynamoDB Local being a requirement for the
  automated test suite to pass in CI
- ❌ Testing only the happy path of a single message, without a test
  covering partial batch failure behavior

## Deploy (`patterns/deploy.md`)

- ❌ Hardcoding the project/stack name inside `deploy.sh`
- ❌ Broad, non-scoped IAM policies (e.g. full SQS/DynamoDB access)
  instead of policies scoped to the specific queue and table this
  function uses
- ❌ Committing any `.env.<environment>` file — only `.env.example` is
  committed

## Local development (`patterns/local-development.md`)

- ❌ `invoke-local.sh` defaulting to a production environment file when
  no argument is passed
- ❌ Having only a single-message sample event and no multi-message batch
  sample — partial-failure behavior can't be exercised locally without one

## SQL data access (`patterns/data-access-sql.md`)

- ❌ Creating a new database engine/connection inside the per-message
  loop instead of once at module scope
- ❌ Running database migrations inside the handler or at import time

## DynamoDB data access (`patterns/data-access-dynamodb.md`)

- ❌ Batching writes across multiple messages by default — start with one
  message → one service call → one write, batch only when volume
  justifies it
