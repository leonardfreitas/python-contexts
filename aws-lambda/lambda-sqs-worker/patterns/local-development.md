# Local Development

## No local server equivalent — use `sam local invoke` with sample events

Unlike an HTTP API context (where `sam local start-api` or plain Uvicorn
gives you a server to send requests to), a queue consumer has no server to
talk to locally. Local development works by invoking the handler directly
with a sample event payload.

```bash
#!/bin/bash
# scripts/invoke-local.sh
set -euo pipefail

ENV_TYPE="${1:-dev}"
EVENT_FILE="${2:-events/sqs-batch.json}"
ENV_FILE=".env.${ENV_TYPE}"

if [ ! -f "$ENV_FILE" ]; then
  echo "Environment file $ENV_FILE not found!"
  exit 1
fi

export $(grep -v '^#' "$ENV_FILE" | xargs)

sam build
sam local invoke WorkerFunction --event "$EVENT_FILE"
```

Usage:

```bash
./scripts/invoke-local.sh                              # defaults to dev + events/sqs-batch.json
./scripts/invoke-local.sh dev events/sqs-single-message.json
```

Same rule as any other context here: the default environment is `dev`,
never `production` — a script meant for local iteration should never
silently touch production by default.

## Sample event files

```json
// events/sqs-single-message.json
{
  "Records": [
    {
      "messageId": "11111111-1111-1111-1111-111111111111",
      "receiptHandle": "local-test-receipt-handle",
      "body": "{\"type\":\"OrderCreated\",\"payload\":{\"order_id\":\"order-123\",\"customer_id\":\"customer-1\",\"items\":[],\"created_at\":\"2026-01-01T00:00:00Z\"}}",
      "attributes": {
        "ApproximateReceiveCount": "1",
        "SentTimestamp": "1700000000000",
        "SenderId": "local",
        "ApproximateFirstReceiveTimestamp": "1700000000000"
      },
      "messageAttributes": {},
      "md5OfBody": "local-test-md5",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:us-east-1:000000000000:local-queue",
      "awsRegion": "us-east-1"
    }
  ]
}
```

```json
// events/sqs-batch.json
{
  "Records": [
    { "...": "first message, same shape as above" },
    { "...": "second message, same shape as above" }
  ]
}
```

Keep at least one file with a single message and one with a multi-message
batch — the batch file is what exercises partial-failure behavior during
local development.

## Faster iteration: calling the handler directly in a Python shell/test

For the fastest feedback loop while developing a new service function,
skip `sam local invoke` (which rebuilds and runs a Docker container) and
call the handler directly with a Python dict, the same way the automated
tests do (see `patterns/testing.md`):

```python
from app.handler import handler

event = {"Records": [...]}
print(handler(event, context=None))
```

**Recommended workflow:** use direct handler invocation for the bulk of
day-to-day service development, and run `sam local invoke` before opening
a pull request or before a real deploy, to catch anything that only shows
up in the actual Lambda build (a missing dependency, an IAM permission
issue that only bites in a real invocation context, and so on).

## Local dependencies

If the worker persists business data locally (a database alongside the
required idempotency table), see the relevant data-access pattern file
for running that dependency locally:

- SQL: `patterns/data-access-sql.md`
- DynamoDB: `patterns/data-access-dynamodb.md`

The idempotency table itself can also run against DynamoDB Local during
development — point `DynamoDBPersistenceLayer` at a local endpoint the
same way the DynamoDB data-access pattern does for business data.

## Summary

| Rule | Reason |
|---|---|
| `invoke-local.sh` defaults to a non-production environment | A missing argument should never silently touch production |
| At least one single-message and one multi-message sample event exist in `events/` | The batch file is what exercises partial-failure behavior |
| Direct handler invocation for day-to-day iteration, `sam local invoke` before shipping | Balances fast feedback with fidelity to the real deployable artifact |
