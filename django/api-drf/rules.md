# Rules — What is FORBIDDEN

An executive summary of every "hard" rule scattered across the other
files. For a quick check, start here; for the full reasoning behind each
rule, see the referenced file.

## Layers (`architecture.md`)

- ❌ Business logic inside `views.py` or `serializers.py`
- ❌ A serializer's `create()`/`update()` doing orchestration (email,
  webhook, external call) — it should, at most, delegate to a service
- ❌ Serializer validation that queries other tables/system state — that's
  business logic, it belongs in the service

## Services (`conventions.md`)

- ❌ A `Service` class with `execute()` with no real state to justify it —
  a function is the default
- ❌ An `OrderService` with 15 unrelated methods ("God Service")
- ❌ Testing a private helper (`_something`) directly — only public
  behavior is tested
- ✅ Rule of thumb: if the logic needs to exist outside the HTTP cycle
  (a command, a worker, a test with no test client), it's a service

## Errors (`patterns/error-handling.md`)

- ❌ `services/` importing `rest_framework` or `django.http`
- ❌ Domain-error `try/except` scattered across multiple views
- ❌ Raising `rest_framework.exceptions.APIException` directly from a
  service
- ✅ All translation from error to HTTP happens in one place: the central
  exception handler

## Transactions (`patterns/transactions.md`)

- ❌ `@transaction.atomic` on a view
- ❌ Firing a background task/email/webhook/external API inside an
  `atomic()` block without going through `transaction.on_commit()`
- ❌ Silently swallowing a database exception inside a service that can be
  called from within another service (breaks savepoints)
- ❌ `atomic()` on a service with a single write (noise, no benefit)

## Testing (`patterns/testing.md`)

- ❌ Mocking the Django ORM (`Model.objects.filter`, etc.)
- ❌ `transaction=True`/`TransactionTestCase` as the default for the whole
  suite (only where `on_commit()` needs to be verified)
- ❌ Factories centralized in one shared cross-domain file

## URLs (`patterns/urls-and-routers.md`)

- ❌ An `@action` inside a `ModelViewSet` for a named business rule
  (cancel, approve, archive) — it becomes a dedicated `APIView`
- ❌ Forcing `ModelViewSet` on a resource without full CRUD
- ❌ A verb before the resource in the URL (`/cancel-order/{id}/`)

## Settings (`architecture.md`)

- ❌ `os.environ.get()` directly, outside `config/settings/env.py`
- ❌ Multiple `local.py`/`production.py` files with diverging logic — one
  settings module, the difference is the env var value

## Deployment (`patterns/deploy.md`)

- ❌ `migrate` inside the container's `CMD`/entrypoint
- ❌ `collectstatic` running at runtime instead of build time
- ❌ Hardcoded secrets in a Dockerfile/production compose file
- ❌ Default groups created via `RunPython` (data migration) — use a
  management command, run after `migrate`

## Admin (`patterns/admin.md`)

- ❌ Business logic in `save_model()`/`ModelAdmin` — always delegate to
  `services/`
- ❌ `list_display` showing a relation field without a matching
  `list_select_related` (N+1)
- ❌ A day-to-day account with `is_superuser`
- ❌ Registering a purely supporting model that nobody will operate
  manually

## Authentication (`auth.md`)

- ❌ Using Django's default `User` without customizing `AUTH_USER_MODEL`
  from the start
- ❌ `AbstractUser` when the default fields (`username`, `first_name`/
  `last_name`) don't fit the domain
- ❌ Login/registration/password-change logic directly inside a
  `simplejwt` view
- ❌ Generating a JWT manually instead of customizing via `get_token()`

## Permissions (`permissions.md`)

- ❌ A `role` field on `User` coexisting with `Group` (two sources of
  truth that diverge)
- ❌ A group name as a bare string scattered through the code
  (`user.groups.filter(name="admin")`) — always via the `Groups` class
- ❌ An "object-specific" rule (e.g. "only the owner can cancel") in
  `has_object_permission` — it lives in the service
- ❌ Editing a user's `groups`/`role` directly in the Admin without an
  auditable named action
- ❌ Reusing a generic `change_x` for a named business action — declare
  a dedicated `Permission` in `Meta.permissions`
