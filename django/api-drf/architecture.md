# Architecture

## The layers

```
Request → View → Serializer → Service → Model/QuerySet → Response
```

Four layers, each with exactly one job:

| Layer | Responsibility | Does NOT do |
|---|---|---|
| **View** | HTTP orchestration: authentication, permission checks, calling the serializer, calling the service, shaping the response | Business logic, complex queries |
| **Serializer** | Format/shape validation of input and output data | Business logic, side effects |
| **Service** | Business logic, orchestration across models, transactions, side effects | Field-format validation, HTTP status codes |
| **Model/Manager/QuerySet** | Data access, reusable queries | Business flow logic |

## Why not pure vertical slice, and not pure clean/hexagonal

Three architectures were considered, and this context picks a hybrid:

- **Classic Django (apps by domain)** — works out of the box with
  migrations and the admin, but tends to accumulate business logic inside
  `views.py`/`models.py` over time.
- **Pure vertical slice** — clean separation by feature, but fights the
  Django ORM/admin, which are designed around `apps` with centralized
  `models.py`.
- **Pure clean/hexagonal** — maximum testability, but expensive
  specifically in Django: the ORM is Active Record, not designed to be
  abstracted behind a repository without generating a large translation
  layer.

**Decision:** Django apps organized by domain (so migrations/admin don't
fight the structure), with a `services/` folder inside each app where every
business operation is isolated into a testable function. This is the
closest to the vertical-slice spirit that Django accepts without friction.

## Boundaries between View and Serializer

**Belongs in the serializer:**
- Type/format validation
- Validation that depends only on the payload's own data (e.g. "end date
  cannot be before start date")
- Representation transforms (`SerializerMethodField`)

**Does NOT belong in the serializer:**
- ❌ Validation that queries other tables/system state — that's business
  logic disguised as validation, it belongs in the service
- ❌ Orchestration (sending an email, firing a webhook) inside
  `create()`/`update()` — the serializer, at most, delegates to the service

```python
# ✅ good
class OrderCreateSerializer(serializers.Serializer):
    customer_id = serializers.UUIDField()
    items = OrderItemSerializer(many=True)

    def save(self, **kwargs):
        return create_order(**self.validated_data, **kwargs)
```

**Belongs in the view:**
- Calling the right serializer for the context
- Calling the service with validated data
- Deciding the status code and response shape

The view does not decide business rules — it receives the decision (or
exception) from the service.

## Folder tree

```
my-project/
  config/
    settings/
      env.py              # pydantic-settings class, the single typed source of truth
      base.py               # builds the real Django settings.py from env
      test.py                # test-only overrides
    urls.py                  # only aggregates each app's urls.py
    celery.py                 # only relevant if a broker URL is set
    wsgi.py
    asgi.py

  core/                        # shared across apps — one-way dependency
    exceptions.py              # DomainError, NotFoundError, ConflictError, ValidationError, PermissionDeniedError
    exception_handler.py       # custom_exception_handler
    pagination.py
    permissions.py
    models.py                  # e.g. TimestampedModel abstract base

  apps/
    accounts/                  # custom User, groups
      models.py
      groups.py                 # Groups.X constants
      services/
      management/commands/setup_default_groups.py
      api/

    orders/
      __init__.py
      apps.py
      models.py
      migrations/

      services/                # business logic — always functions
        __init__.py
        create.py
        cancel.py
        queries.py

      exceptions.py             # OrderAlreadyShippedError, InsufficientStockError

      api/                       # HTTP layer, isolated
        views.py
        urls.py
        serializers/
          list.py
          detail.py
          create.py
          update.py

      tasks.py                   # Celery tasks, only if the app uses one
      admin.py
      factories.py

      tests/
        test_services.py
        test_views.py
        test_serializers.py

    customers/    (same structure)
    billing/      (same structure)

  tests/
    conftest.py                  # global fixtures, cross-app tests (rare)

  manage.py
  pyproject.toml
  .env.example
```

### Why `apps/` as an umbrella folder

Keeps the project root from getting cluttered once there are 8-10 domains —
visually separates "project infrastructure" from "business domain."

### Why `api/` is isolated inside each app

Physically separates "pure Django/domain" (`models.py`, `services/`,
`exceptions.py`) from "HTTP shell" (`api/views.py`, `api/serializers/`).
Makes it visible when something is imported the wrong way — a file in
`services/` importing from `api/` is an instant code smell during review,
before lint even runs.

If a domain ever needs another "entry point" (a queue consumer, a complex
management command), it goes next to `api/`, without touching `services/`
or `models.py`. The domain stays protected from knowing how it's consumed.

### Why tests live inside the app

`apps/orders/tests/`, not a centralized `tests/` folder mirroring the
structure. Moving or deleting the app takes its tests along, no orphans.
Reinforces that tests are part of the domain, not an appendix.

The root `tests/` folder exists only for global fixtures (`conftest.py`,
an `api_client` fixture) and, rarely, tests that intentionally cross
multiple apps (e.g. "creating an order triggers a billing charge" — testing
the seam between `orders` and `billing`).

### Rule about `core/`

`core/` is a one-way dependency: apps may import from `core/`, but `core/`
never imports from a specific app. If `core/` needs to know about
something from an app, that's a signal the thing isn't genuinely generic.
