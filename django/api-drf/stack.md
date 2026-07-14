# Stack

## General rule

Every conditional dependency needs a justification recorded in the pull
request that introduces it — "why this project needs it now," not
"because we might need it later." Adding infrastructure "just in case"
means running a service with no real usage — exactly the kind of decision
this context treats as ❌.

## Core — always present, regardless of project size

| Library | Role |
|---|---|
| `uv` | Package/venv manager. Fast (Rust-based) |
| Django (LTS) + DRF | Base framework |
| `pydantic-settings` | Typed env var validation — the app fails to boot if a required variable is missing |
| `psycopg` (v3, not `psycopg2`) | Actively maintained Postgres driver, native async support |
| `django-filter` | Standardized querystring filtering |
| `drf-spectacular` | OpenAPI/Swagger generated from the actual code (serializers, views) |
| `djangorestframework-simplejwt` | JWT Bearer + refresh token in a cookie |
| `ruff` | Lint + format in a single binary (replaces flake8 + isort + black) |
| `mypy` + `django-stubs` | Static typing — kept as core, not conditional |
| `pre-commit` | Runs ruff/mypy before every commit |
| `pytest` + `pytest-django` | Testing |
| `factory-boy` + `Faker` | Test fixtures |
| `pytest-cov` | Coverage |
| `pytest-mock` | `unittest.mock` wrapper with fixture syntax |

## Conditional — only added when the project justifies it

| Library | Add it when | Sign you don't need it yet |
|---|---|---|
| Celery + a broker (e.g. Redis) | There's work that genuinely needs to run outside the request/response cycle — bulk email, processing that takes >1-2s, real cron jobs | "Send one transactional email here and there" — that can be synchronous |
| An error-tracking service (e.g. Sentry) | The project is in production receiving real user traffic | MVP/low-risk internal project — logs + your log aggregator already cover it |
| Structured logging (e.g. `structlog`) | Meaningful log volume and/or multiple services correlating a trace id | Small project, plain-text logs are enough to debug |
| A health-check library | Orchestrated deployment (auto-scaling, container orchestration) | Single-instance deployment with no load balancer deciding instance health |
| An object storage integration (e.g. `django-storages` + an S3-compatible client) | The project handles user file/media uploads | An API that only manipulates structured data |

## Deployment

| Library | Role |
|---|---|
| `gunicorn` | Production WSGI server (`worker-class gthread` for I/O-bound workloads) |
| `whitenoise` | Serves static files directly from Django, no separate static file server needed |
| `dj-database-url` | Parses a `DATABASE_URL` connection string |

## Admin

The **native** Django Admin. Don't reach for a third-party admin theme
library by default — only if the team genuinely needs a richer UI.

## A note on async

This stack assumes a **synchronous** Django + DRF setup. If a specific view
would genuinely benefit from `async def` (e.g. a heavy external API call),
evaluate `uvicorn` + `asgi.py` for that specific case — don't migrate the
whole project "just in case."
