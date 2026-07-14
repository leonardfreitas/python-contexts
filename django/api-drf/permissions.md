# Permissions and Groups

## Two different questions

1. **"Who has access to which endpoint/action?"** → API authorization,
   resolved with Django's `Permission`/`Group` + DRF permission classes
2. **"Can this specific person act on THIS specific record?"** → business
   logic, resolved inside the service

Mixing these two is the source of most permission headaches.

## `Group` is the single source of truth for user identity/role

There is no `role` field on `User` duplicating this information. Reason:
two write paths for the same information (a `role` field + a `Group`
membership maintained separately) will diverge over time — it's a matter of
when, not if. The most robust way to eliminate that synchronization bug is
to eliminate the need to synchronize, not to validate consistency after the
fact.

Bonus: adding a new role (e.g. "senior reviewer") becomes a data operation
(create a `Group`), with no code change or deploy required.

## Group names are never bare strings

A bare string (`user.belongs_to("admin")`) suffers from a worse problem
than visible divergence: a silent failure. A typo (e.g. a missing letter)
never raises an exception — it just returns `False` forever, silently
denying access without leaving a trace. Much harder to track down than an
obvious bug.

**Solution:** named constants, editor autocomplete, a localized import
error if the constant doesn't exist:

```python
# apps/accounts/groups.py
class Groups:
    ADMINS = "admins"
    REVIEWERS = "reviewers"
    SUPPORT = "support"
    FINANCE = "finance"
```

## `belongs_to()` on `User`, cached per request

```python
# apps/accounts/models.py
class User(AbstractBaseUser, PermissionsMixin):
    ...

    @cached_property
    def group_names(self) -> set[str]:
        return set(self.groups.values_list("name", flat=True))

    def belongs_to(self, group_name: str) -> bool:
        return group_name in self.group_names
```

```python
# usage
if not user.belongs_to(Groups.REVIEWERS):
    raise PermissionDeniedError(...)
```

`cached_property` avoids unnecessary repeated queries when `belongs_to()`
is called multiple times in the same request (a permission class, then a
service, for instance) — the first call hits the database, subsequent ones
reuse the cache on that `request.user` instance.

**Trade-off worth knowing:** bulk queries filtering by role
(`User.objects.filter(groups__name=Groups.REVIEWERS)`) become a `JOIN`
through the M2M table, more expensive at very large scale than a direct
indexed field. For most projects this never becomes an actual bottleneck.

## Default groups come from code, applied by a management command

Never created manually in the Admin in production — every environment
would diverge, with no single source of truth.

**Don't use a data migration (`RunPython`) for this:** custom permissions
from `Meta.permissions` are only created by the `post_migrate` signal,
which fires **after** every migration in a `migrate` run has finished. A
data migration that tries to fetch a freshly declared `Permission` in the
same `migrate` run risks a real `Permission.DoesNotExist` — it works in
local development (because the permission already exists from an earlier
run) and fails in a fresh environment (CI, a new staging setup).

```python
# apps/accounts/management/commands/setup_default_groups.py
class Command(BaseCommand):
    help = "Creates/updates the system's default groups idempotently."

    def handle(self, *args, **options):
        for group_name, perm_codenames in DEFAULT_GROUPS.items():
            group, created = Group.objects.get_or_create(name=group_name)
            perms = Permission.objects.filter(codename__in=perm_codenames)
            group.permissions.set(perms)
```

Advantages over `RunPython`:
- Runs after `migrate` (and `post_migrate`) has already finished — no risk
  of `Permission.DoesNotExist`
- Uses the "live" model (custom managers, methods like `belongs_to()`), not
  the frozen historical version that `apps.get_model()` returns inside a
  migration
- Naturally idempotent (`get_or_create` + `.set()`), no `RunPython.noop`
  ceremony
- Testable with `call_command()`, no need for a migration-testing harness

Runs as the second step of the deploy pipeline, immediately after
`migrate` (see `patterns/deploy.md`).

## Custom permission per business action

A "special" business action (cancel, approve, archive) deserves its own
`Permission`, not a reuse of the generic `change_x` — "editing" and
"cancelling" are conceptually different authorizations.

```python
# models.py
class Order(models.Model):
    class Meta:
        permissions = [
            ("cancel_order", "Can cancel order"),
            ("refund_order", "Can refund order"),
        ]
```

## "Specific object" business rules live in the service, not in `has_object_permission`

```python
# ❌ mixes technical authorization with domain logic, and disappears
# outside the DRF request cycle (management command, background task)
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```

```python
# ✅ lives in the service, works in any calling context
def cancel_order(order: Order, cancelled_by: User, reason: str) -> Order:
    if not cancelled_by.is_staff and order.owner_id != cancelled_by.id:
        raise PermissionDeniedError("You can only cancel your own orders")

    if order.status == OrderStatus.SHIPPED:
        raise OrderAlreadyShippedError(order.id)
    ...
```

`PermissionDeniedError` is the same `DomainError` family already designed
in `patterns/error-handling.md` — the central exception handler already
translates it to a 403 automatically, no new mechanism needed.

## Decision matrix

| Question | Layer | Example |
|---|---|---|
| "Can this type of user generally access this type of resource?" | `Permission`/`Group` (DRF permission class) | Support staff can view orders |
| "Can this specific person act on THIS record?" | Service (domain rule) | A customer can only cancel their own order |
| "Does the record's current state allow this action?" | Service (domain rule) | Can't cancel an order that already shipped |

## The Admin doesn't edit `groups` freely

Preserves traceability/auditability — a named admin action ("add as
reviewer") that logs the intent, instead of raw M2M editing with no
context for the "why" (see `patterns/admin.md`).

## Summary

| Rule | Reason |
|---|---|
| `Group` is the single source of truth for identity — no duplicated `role` | Eliminates divergence structurally |
| Group names only via the `Groups` class, never bare strings | A typo becomes a localized import error, not a silent `False` |
| `belongs_to()` uses `cached_property` | Avoids N+1 permission checks per request |
| Custom business actions get their own `Permission` | Correct granularity |
| Default groups via `setup_default_groups` (command), not `RunPython` | Avoids `Permission.DoesNotExist` in a fresh environment |
| Object-specific rules live in the service | Works outside the HTTP cycle too |
