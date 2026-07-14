# Local Development

## The script

`scripts/start-local.sh` builds the SAM application and runs it locally
via `sam local start-api`, which emulates API Gateway on your machine and
invokes the real Lambda handler for each request.

```bash
#!/bin/bash
# scripts/start-local.sh
set -euo pipefail

ENV_TYPE="${1:-dev}"
ENV_FILE=".env.${ENV_TYPE}"

if [ ! -f "$ENV_FILE" ]; then
  echo "Environment file $ENV_FILE not found!"
  exit 1
fi

export $(grep -v '^#' "$ENV_FILE" | xargs)

sam build
sam local start-api \
  --parameter-overrides \
    EnvType="$ENV_TYPE" \
    JwtSecretKey="$JWT_SECRET_KEY" \
    EncryptionKey="$ENCRYPTION_KEY" \
    JwtExpireHours="$JWT_EXPIRE_HOURS"
```

Usage:

```bash
./scripts/start-local.sh          # defaults to .env.dev
./scripts/start-local.sh staging  # runs locally against staging config
```

## Why the default environment is `dev`, never `production`

> The default argument for `start-local.sh` must always resolve to a
> non-production `.env` file. If a developer runs the script with no
> argument by habit, the safe outcome is testing against local/dev
> configuration — not accidentally exercising production secrets or
> pointing at production infrastructure.

This is a small detail with an outsized blast radius if it's wrong: a
script meant for local iteration should never have production as its
silent default.

## Two ways to run locally, and when to use each

### `sam local start-api` — closest to production behavior

Emulates the full Lambda invocation lifecycle: cold start behavior, the
API Gateway event shape, IAM-adjacent configuration from the template.
Slower to start and to iterate (each request goes through a simulated
Lambda invocation, including a Docker container per function), but it's
the most faithful way to catch integration issues before they reach a
real environment — a broken `Handler` path in `template.yaml`, a missing
environment variable, a Mangum adapter issue.

Use this when validating the deployable artifact itself, not just
business logic.

### Running the FastAPI app directly with Uvicorn — fastest iteration loop

Because `app/main.py` and `app/handler.py` are separated (see
`architecture.md`), the same FastAPI app can run with zero Lambda
emulation at all:

```bash
uvicorn app.main:app --reload
```

- ✅ Instant reload on code change, no Docker, no `sam build` step
- ✅ Ideal for day-to-day route/service development
- ❌ Doesn't validate the SAM template, the Mangum adapter, or anything
  Lambda-specific (cold start behavior, IAM, the API Gateway event shape)

**Recommended workflow:** use Uvicorn directly for the bulk of day-to-day
development (fast feedback loop while writing routes and services), and
run `sam local start-api` before opening a pull request or before a real
deploy, to catch anything that only shows up when going through the actual
Lambda invocation path.

## Local dependencies that don't exist in Lambda

Local development often needs services Lambda doesn't have locally
available by default — a local SQL database, or DynamoDB Local. These are
covered per data-access strategy:

- SQL: see `patterns/data-access-sql.md` for running a local database
  alongside `sam local start-api` or Uvicorn
- DynamoDB: see `patterns/data-access-dynamodb.md` for running DynamoDB
  Local and pointing the app at it via an endpoint override

## Summary

| Rule | Reason |
|---|---|
| `start-local.sh` defaults to a non-production environment | A missing argument should never silently touch production |
| `set -euo pipefail` in the script | Stops on the first error instead of continuing with a broken build |
| Uvicorn for day-to-day iteration, `sam local start-api` before shipping | Balances fast feedback with fidelity to the real deployable artifact |
| `app/main.py` stays Mangum-free | Keeps the app runnable with plain Uvicorn, with no Lambda emulation required |
