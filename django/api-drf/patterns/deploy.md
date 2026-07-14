# Deployment

## Application server

Gunicorn running `wsgi.py` (synchronous, given the current stack):

```
gunicorn config.wsgi:application --workers 4 --worker-class gthread --threads 2 --bind 0.0.0.0:8000
```

- Workers: `(2 x CPU cores) + 1` as a starting point, tune with real load
- `gthread` + threads makes better use of resources for I/O-bound
  workloads (typical of an API waiting on the database/external APIs)
- If a specific view would benefit from `async def`, evaluate `uvicorn`
  for that case only — don't migrate everything "just in case"

## Static files

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    ...
]

STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
```

`collectstatic` runs **during the Docker image build**, never at runtime —
otherwise different instances could serve different static file versions
between deploys.

## Migrations — the most dangerous part of a Django deploy

**Never inside the container's `CMD`/entrypoint:**

```dockerfile
# ❌ dangerous
CMD python manage.py migrate && gunicorn config.wsgi:application
```

Spinning up 3 instances at once fires 3 concurrent `migrate` runs — a race
condition on table locks, or worse, a destructive migration running while
old instances still serve traffic against the old schema.

**Correct pattern:** a single Docker artifact, with the command
overridden, run in isolation as a separate pipeline step, **before**
traffic is routed to the new version:

```bash
# Run the app (stays up)
docker run my-image:v123 gunicorn config.wsgi:application --bind 0.0.0.0:8000

# Run the migration (run-and-exit, doesn't stay up)
docker run my-image:v123 python manage.py migrate
```

### By platform

| Platform | Mechanism | Runs where |
|---|---|---|
| A PaaS with deployment hooks (e.g. platform-level pre-deploy hooks) | A hook that runs once, before traffic cutover | Usually a single leader instance |
| Container orchestrators (e.g. ECS, Kubernetes) | A one-off task/job, run before updating the running service | A dedicated task, isolated from the running service |
| CI/CD pipeline | A separate job that must succeed before the deploy job runs | Fully outside the production infrastructure |

Concretely, most modern platforms — whether a traditional PaaS, a managed
container platform, or a CI/CD-driven deploy — expose some form of
"pre-deploy command" or "release step" that runs once, in an isolated
container, between build and traffic cutover. Look for that mechanism
specifically; it's the right place for migrations regardless of platform
name.

**A relevant caveat:** on some platforms there's a time gap between the
pre-deploy step running and new instances actually going live. This works
fine for additive migrations (a new column, a new table). For destructive
migrations, the safe approach is still a two-phase deploy: ship code that
tolerates both schemas → run the destructive migration → ship code that
only uses the new schema.

### Rule

> Migrations run via the platform's native "pre-deploy"/"release step"
> mechanism. Never inside the application container's `CMD`. The choice of
> platform doesn't change this rule — only the name of the mechanism that
> implements it.

## Setting up default groups

A `setup_default_groups` management command (see `permissions.md`) runs as
a second step, immediately after `migrate` — never as a data migration
(`RunPython`), to avoid a `Permission.DoesNotExist` error the first time it
runs in a fresh environment (permissions declared in `Meta.permissions`
only exist once the `post_migrate` signal fires, which happens after
`migrate` fully completes).

## Secrets

Never hardcoded in a Dockerfile or a production compose file. In
production: a managed secrets store (cloud secrets manager, parameter
store, etc.), injected as an environment variable at boot. The Django code
doesn't change depending on where the env var comes from —
`pydantic-settings` abstracts that away.

## Dockerfile — multi-stage with `uv`

```dockerfile
# build stage
FROM python:3.13-slim AS builder
RUN pip install uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# runtime stage
FROM python:3.13-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN python manage.py collectstatic --noinput
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

Multi-stage: the final image doesn't carry build tooling, only the
resolved venv + code — smaller image, smaller attack surface.

## Health checks

If the deployment is orchestrated (auto-scaling), a health-check library
becomes worth adding:

```python
path("health/", include("health_check.urls")),
```

## Summary

| Rule | Reason |
|---|---|
| `migrate` never in the `CMD`/entrypoint | Race condition across instances |
| `collectstatic` at build time, not runtime | Consistency across instances |
| Secrets via a managed secrets store | Security |
| Multi-stage Dockerfile | Smaller image, no build tooling at runtime |
| `setup_default_groups` as a command, after `migrate` | Avoids `Permission.DoesNotExist` |
| Health check if the deployment is orchestrated | The load balancer needs to know instance health |
