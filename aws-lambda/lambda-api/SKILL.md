# AWS Lambda API Context (FastAPI + SAM)

You are a senior engineer building a REST API deployed as an AWS Lambda
function behind API Gateway, using FastAPI and AWS SAM. Follow all the
contexts below before generating any code.

This context is independent from any other context in this collection
(e.g. the Django context) — it shares only foundational principles
(isolating business logic, one source of truth, adding complexity only
when justified), not specific rules.

This is the entry point. Read in this order:

1. `architecture.md` — layers, bounded-context-per-Lambda rule, project folder structure
2. `stack.md` — required libraries and why
3. `conventions.md` — naming, service pattern, schema strategy
4. `rules.md` — what is FORBIDDEN, with ❌ / ✅ examples
5. `patterns/` — step-by-step recipes for recurring tasks

## Core philosophy

A Lambda function has a fundamentally different lifecycle than a
long-running server: it's born, runs briefly, and dies. That changes what
matters architecturally — cold start cost, packaging boundaries, and where
a piece of logic should live all have different answers than in a
traditional server-based API. This context exists to apply the same
underlying discipline (business logic isolated and testable, one source of
truth, dependencies added only when justified) to that different shape of
application.

If you are generating code and business logic appears to be leaking into
a route or into AWS-specific plumbing (boto3 calls scattered through
routes, for example) — stop and move it into `services/` or the
repository layer. This is not a style preference.

## Choosing a data-access pattern

This context supports both SQL (e.g. RDS/Postgres) and DynamoDB as the
persistence layer. Almost everything in this context (layering, FastAPI,
Mangum, SAM, testing) is identical regardless of which one a project uses
— only the data-access layer differs. Read the pattern that matches your
project:

- Using a relational database → `patterns/data-access-sql.md`
- Using DynamoDB → `patterns/data-access-dynamodb.md`

Don't read both unless you're comparing the two for a new project decision.

## Patterns index

- `patterns/new-endpoint.md` — step-by-step to add a new route/endpoint
- `patterns/error-handling.md` — domain exceptions + FastAPI exception handler
- `patterns/testing.md` — testing pyramid, bypassing Mangum, mocking AWS with moto
- `patterns/deploy.md` — the deploy script, SAM template, secrets handling
- `patterns/local-development.md` — running the app locally, Uvicorn vs `sam local`
- `patterns/data-access-sql.md` — SQL-specific: connection pooling, RDS Proxy, migrations
- `patterns/data-access-dynamodb.md` — DynamoDB-specific: single-table design, access patterns

## Structure of this context

```
lambda-api/
  SKILL.md                     → this file, entry point + index
  architecture.md                → layers, bounded-context-per-Lambda rule, folder structure
  stack.md                         → required libraries and why
  conventions.md                     → naming and code patterns
  rules.md                             → what is FORBIDDEN
  patterns/
    new-endpoint.md
    error-handling.md
    testing.md
    deploy.md
    local-development.md
    data-access-sql.md
    data-access-dynamodb.md
```
