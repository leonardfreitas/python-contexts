# Architecture

## The layers

```
API Gateway → Mangum (adapter) → FastAPI route → Service → Data layer (SQL or DynamoDB) → Response
```

| Layer | Responsibility | Does NOT do |
|---|---|---|
| **Route (FastAPI)** | Request parsing/validation via Pydantic, calling the service, shaping the response and status code | Business logic, direct data access |
| **Service** | Business logic, orchestration, flow decisions | HTTP details, SQL/DynamoDB-specific details |
| **Data layer** | Data access — queries, `put_item`, transactions | Business flow logic |

This mirrors the same separation of concerns used in server-based
architectures, but it matters even more here: Lambda's cold start is
penalized by heavy imports. Isolating the data-access layer makes it clear
exactly what needs to be imported, and makes it easy to apply lazy imports
where it helps.

## Where services live

Functions, not classes — same rule as elsewhere: a class is only justified
when the service needs to hold configurable state across calls (e.g. an
injected payment client). The litmus test here: if the logic would need to
be called both from an HTTP route today and, potentially, from a queue
consumer in a companion `lambda-worker` context tomorrow, it's a service.

```python
# app/services/create_order.py
def create_order(customer_id: str, items: list[OrderItem]) -> Order:
    if not _has_stock(items):
        raise InsufficientStockError()

    order = repository.save_order(customer_id, items)
    return order
```

## The core rule: one Lambda per bounded context

> A Lambda function exists per bounded context, never per an arbitrary
> route count. Routes belong to the same Lambda when they share the same
> domain model, are mostly owners of the same data, and change together in
> practice (a pull request touching one rarely leaves the other
> untouched). Routes become separate Lambdas when they represent domains
> a team would design as distinct services anyway — even if today they're
> only a handful of routes.

Signals that two routes **belong to the same context** (same Lambda):
- They mostly share the same tables/entities
- One route's business logic internally calls the other's
- They're part of the same continuous business flow (create → list →
  update → cancel the same resource)

Signals that they are **different contexts** (separate Lambdas):
- They only communicate via an async event or an explicit API call, never
  a direct code import
- They own completely different data, with no table overlap
- One could plausibly be extracted to a different team/repository without
  drama

### Example — same context (one Lambda)

```
orders-api/                (one Lambda)
  app/
    main.py
    routes/
      orders.py               # POST /orders, GET /orders/{id}, PATCH /orders/{id}
      order_items.py           # POST /orders/{id}/items — same aggregate
    services/
      create_order.py
      add_item_to_order.py
      cancel_order.py
    repository.py               # data access for Order — SQL or DynamoDB
```

`order_items.py` lives in the same Lambda as `orders.py` because an order
item doesn't exist without the order — it's the same aggregate, the same
data owner, and it changes together in practice.

### Example — different contexts (separate Lambdas)

```
orders-api/        (Lambda 1 — owns the Order entity)
notifications-api/  (Lambda 2 — owns the Notification entity, sends emails/push)
```

Even if `notifications-api` today only has two routes, it's a separate
Lambda because: it owns different data, it changes for different reasons
(a different team owns notifications vs orders), and communication between
them should happen via an event (`orders-api` publishes `OrderCreated`,
`notifications-api` reacts) — never a direct import of one domain's code
into the other's.

## Project folder structure (a single Lambda / bounded context)

```
orders-api/                    ← project root = one Lambda = one bounded context
  template.yaml                  # defines this Lambda in SAM
  samconfig.toml                  # SAM CLI config (optional)

  scripts/
    deploy.sh
    start-local.sh

  .env.example                    # committed
  .env.dev                         # gitignored
  .env.staging                      # gitignored
  .env.production                    # gitignored
  .gitignore

  pyproject.toml / requirements.txt

  app/
    main.py                          # builds the FastAPI app, Mangum-free — runs with plain Uvicorn
    handler.py                        # handler = Mangum(app) — the actual Lambda entry point

    routes/                            # the View layer — one file per route/sub-resource
      orders.py
      order_items.py

    services/                           # business logic, always functions
      create_order.py
      cancel_order.py
      queries.py

    schemas/                             # Pydantic — split by operation
      order_create.py
      order_response.py

    exceptions.py                         # DomainError and subclasses

    repository.py                          # data layer — SQL or DynamoDB

  tests/
    test_services.py
    test_routes.py
```

## The rules that define this structure

### Rule 1 — One project = one Lambda = one bounded context

There is no "multiple Lambdas inside the same project root" in this
context. If a team decides `notifications` is a separate context from
`orders`, it becomes **another project root** with its own
`template.yaml`, its own `app/`, its own scripts — not a second folder
inside the same project.

### Rule 2 — `app/main.py` and `app/handler.py` are always two separate files

This is not optional: `main.py` **never** imports Mangum. Only
`handler.py` knows about the adapter. This is what lets
`uvicorn app.main:app --reload` work with zero Lambda-specific machinery
involved.

### Rule 3 — `routes/`, `services/`, `schemas/` are always folders, never a single catch-all file

Even in a small project with few routes, the convention starts as a
folder — this avoids the "let's just put it all in one `views.py`"
anti-pattern. One file per operation/sub-resource inside each.

### Rule 4 — `repository.py` starts as a single file, becomes a folder only with multiple relevant entities

```
# context with one main entity (Order)
repository.py

# context with multiple related entities (Order + OrderItem + Payment)
repository/
  orders.py
  payments.py
```

Start as a single file, split into a folder only once the bounded context
genuinely persists more than one entity — not preemptively.

### Rule 5 — `tests/` lives at the project root, not inside `app/`

Unlike a multi-app framework, an entire project here is already a single
domain (one Lambda = one bounded context) — so there's no "multiple apps
within the same project" to justify scattering tests. `tests/` at the
root, mirroring `app/`, is sufficient because bounded-context granularity
was already resolved one layer up (one Lambda per context), not within
the project.

### Rule 6 — `scripts/`, `.env.*`, `template.yaml` always live at the root

They aren't application code — they're project operations tooling
(deploy, local execution, infrastructure). They sit alongside `app/` and
`tests/`, not inside either.

## What changes between SQL and DynamoDB

Only the contents of `repository.py`/`repository/`. The surrounding folder
structure is identical either way — this confirms that keeping one shared
context with two separate data-access pattern files (rather than
duplicating the entire context per database) was the right call: the
folder tree doesn't fork, only one piece's internal implementation does.
