# Rules — What is FORBIDDEN

An executive summary of every "hard" rule scattered across the other
files. For a quick check, start here; for the full reasoning behind each
rule, see the referenced file.

## Bounded context (`architecture.md`)

- ❌ Multiple bounded contexts living inside the same project root/Lambda
  "just because they're small"
- ❌ Splitting a Lambda by arbitrary route count instead of by bounded
  context
- ❌ A `services/` function in one context importing code directly from
  another context — cross-context communication happens via an event or
  an explicit API call

## Layers (`architecture.md`)

- ❌ Business logic inside `routes/`
- ❌ `services/` importing anything from `fastapi`
- ❌ Data-access details (SQL/DynamoDB specifics) leaking into a service —
  the service calls the repository layer, it doesn't know how persistence
  is implemented

## Project structure (`architecture.md`)

- ❌ `app/main.py` importing Mangum — only `app/handler.py` may
- ❌ A single catch-all file for all routes/services/schemas instead of
  one file per operation
- ❌ `scripts/`, `.env.*`, or `template.yaml` living inside `app/` instead
  of the project root

## Error handling (`patterns/error-handling.md`)

- ❌ `try/except` of a domain error scattered across multiple routes
- ❌ Raising an HTTP-aware exception directly from a service
- ✅ All translation from error to HTTP happens in exactly one place: the
  exception handler registered on the FastAPI app

## Testing (`patterns/testing.md`)

- ❌ Tests making real AWS calls — use `moto`
- ❌ Simulating a full API Gateway event (via Mangum or `sam local`) as
  the default way to test a route — use `TestClient` instead, reserve full
  event simulation for a small number of smoke tests
- ❌ A local database/DynamoDB Local being a requirement for the test
  suite to pass in CI

## Deploy (`patterns/deploy.md`)

- ❌ Hardcoding the project/stack name inside `deploy.sh`
- ❌ A sensitive template parameter without `NoEcho: true`
- ❌ Committing any `.env.<environment>` file — only `.env.example` is
  committed
- ❌ A deploy/local script without `set -euo pipefail`

## Local development (`patterns/local-development.md`)

- ❌ `start-local.sh` defaulting to a production environment file when no
  argument is passed

## SQL data access (`patterns/data-access-sql.md`)

- ❌ Creating a new SQLAlchemy engine inside the handler function on every
  invocation — create it once, at module scope
- ❌ Running database migrations inside the Lambda handler or at import
  time — migrations run as an explicit deploy step
- ❌ Skipping RDS Proxy (or an equivalent connection-pooling solution) for
  a project with real concurrent production traffic

## DynamoDB data access (`patterns/data-access-dynamodb.md`)

- ❌ Designing one table per entity — DynamoDB is designed around one
  table per bounded context, with keys/GSIs modeling access patterns
- ❌ Assuming a single `query`/`scan` call returns all matching data
  without checking for `LastEvaluatedKey`
- ❌ Looping individual `put_item` calls for bulk writes instead of using
  `batch_writer()`
