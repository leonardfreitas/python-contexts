# Stack

## Core — always present

| Library | Role |
|---|---|
| **FastAPI** | Routing, Pydantic-based validation, automatic OpenAPI |
| **Mangum** | Adapter that translates the API Gateway event into ASGI for FastAPI |
| **Pydantic v2** | Request/response schema validation |
| **boto3** | AWS SDK — DynamoDB, S3, EventBridge, Secrets Manager, or whatever the project needs |
| **AWS Lambda Powertools for Python** | Structured JSON logging, tracing, and metrics, Lambda-aware out of the box |
| **pytest** | Testing |
| **moto** | Mocks AWS services in tests — no real AWS calls needed for most test cases |
| **ruff** | Lint + format in a single binary |
| **mypy** | Static typing |
| **AWS SAM CLI** | Build, local emulation, deploy |

## Why Powertools is core, not conditional

Structured logging isn't a "nice to have" here the way it can be
conditional in a traditional server-based context. In Lambda, every cold
start, timeout, and retry is effectively invisible unless the log is
structured and correlated — a log aggregator can't usefully query
unstructured text across thousands of short-lived invocations. Powertools
provides a request-scoped logger with a correlation id for close to zero
setup cost, so it stays in core rather than being gated behind a
usage-justification threshold.

```python
from aws_lambda_powertools import Logger

logger = Logger()

@logger.inject_lambda_context
def handler(event, context):
    ...
```

## Conditional — only added when the project justifies it

| Library | Add it when |
|---|---|
| `SQLAlchemy` | The project uses a relational database (RDS/Postgres) — see `patterns/data-access-sql.md` |
| RDS Proxy (infrastructure, not a library) | The project uses SQL under real concurrent traffic — see `patterns/data-access-sql.md` for why this becomes close to mandatory, not optional, once traffic is real |
| `aws-xray-sdk` | Distributed tracing needs beyond what Powertools' default integration already covers |

## A note on Mangum vs the AWS Lambda Web Adapter

This context defaults to **Mangum**: it's simpler, fully transparent (a
pure Python translation layer, no extra infrastructure), and covers the
large majority of REST API use cases.

The **AWS Lambda Web Adapter** is a valid alternative worth knowing about:
it runs the FastAPI app exactly as it would run anywhere else (via
Uvicorn, listening on a real port), with a Lambda Extension translating
the incoming event into a real HTTP call against that local server. It
supports response streaming natively and makes migrating from Lambda to a
container platform later closer to a non-event — but it adds an
infrastructure layer that's less transparent than Mangum's pure-Python
approach, and a slightly heavier cold start.

Default to Mangum. Consider the Web Adapter specifically when response
streaming is required, or when portability between Lambda and a container
runtime is a known future requirement — not preemptively.
