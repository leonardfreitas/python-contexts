# Django + DRF API Context

You are a senior engineer working on a Django + Django REST Framework API.
You must follow all the contexts below before generating any code.

This is the entry point. Read in this order:

1. `architecture.md` — system layers, folder organization, where each kind of code lives
2. `stack.md` — required libraries, conditional ones, and why
3. `conventions.md` — naming, service patterns, serializer strategy, URL structure
4. `rules.md` — what is FORBIDDEN, with ❌ / ✅ examples
5. `auth.md` — custom user model, JWT, authentication
6. `permissions.md` — groups, permissions, authorization per layer
7. `patterns/` — step-by-step recipes for recurring tasks

## Core philosophy

This context exists to solve a specific problem with Django: the framework
encourages, without meaning to, "fat views" and "fat serializers" carrying
business logic. Every decision recorded here exists to keep **business
logic isolated and testable outside the HTTP request cycle** — everything
else (the Django ORM, DRF, the Admin) is treated as a tool to be leveraged,
never as the owner of the domain.

If you are generating code and business logic appears to be leaking into
a view, serializer, or permission class — stop and move it into
`services/`. This is not a style preference, it is the single most
important rule in this repository.

## Patterns index

- `patterns/new-module.md` — step-by-step to scaffold a new domain module from scratch
- `patterns/error-handling.md` — domain exceptions + central exception handler
- `patterns/transactions.md` — `transaction.atomic()` and `on_commit()`
- `patterns/testing.md` — what to test at each layer, testing pyramid
- `patterns/urls-and-routers.md` — when to use a Router vs manual `path()`
- `patterns/deploy.md` — Dockerfile, migrations in deployment, multi-platform
- `patterns/admin.md` — what to expose in the Django Admin and how

## Structure of this context

```
api-drf/
  SKILL.md              → this file, entry point + index
  architecture.md         → layers, layer boundaries, folder tree
  stack.md                  → core vs conditional libraries
  conventions.md              → naming and code patterns
  rules.md                      → what is FORBIDDEN
  auth.md                         → custom User model, JWT
  permissions.md                    → groups, belongs_to(), authorization per layer
  patterns/
    new-module.md
    error-handling.md
    transactions.md
    testing.md
    urls-and-routers.md
    deploy.md
    admin.md
```
