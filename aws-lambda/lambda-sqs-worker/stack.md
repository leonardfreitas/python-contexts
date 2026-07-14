# Stack

## Core — always present

| Library | Role |
|---|---|
| **AWS Lambda Powertools for Python** | `BatchProcessor` (partial batch failure reporting), the `Idempotency` utility, and structured JSON logging — kept as core, not conditional (see rationale below) |
| **Pydantic v2** | Message body validation/parsing |
| **boto3** | AWS SDK for the data layer and any downstream calls |
| **pytest** | Testing |
| **moto** | Mocks SQS, DynamoDB (including the idempotency table), and any other AWS service touched in tests |
| **ruff** | Lint + format |
| **mypy** | Static typing |
| **AWS SAM CLI** | Build, local invoke, deploy |

Notably absent compared to a synchronous HTTP API context: no web
framework, no ASGI adapter. There is no HTTP surface in this context —
everything is oriented around batch/event processing.

## Why Powertools' batch and idempotency utilities are core, not conditional

Both partial batch failure handling and idempotency solve problems that
are not edge cases in an SQS-triggered worker — they are the direct
consequence of SQS's delivery contract (batches can partially fail;
messages can be delivered more than once). A worker that skips either one
isn't taking a simplicity shortcut, it's shipping a worker with a known
data-correctness bug waiting to happen under normal, expected queue
behavior. Both utilities are kept in core for every project using this
context.

## Conditional — only added when the project justifies it

| Library | Add it when |
|---|---|
| `SQLAlchemy` | The project persists business data in a relational database — see `patterns/data-access-sql.md` |
| RDS Proxy (infrastructure, not a library) | SQL under real concurrent invocation volume — same reasoning as any other Lambda context using SQL |

## A note on parallel processing within a batch

Powertools' `BatchProcessor` supports parallel processing via a
`max_workers` parameter, but this context defaults every new project to
**sequential** processing within a batch. Parallelizing is a configuration
change to make once volume genuinely justifies it — see
`patterns/batch-processing.md` for when it's safe (and when, on a FIFO
queue, it actively breaks the ordering guarantee that's the reason to use
FIFO in the first place).
