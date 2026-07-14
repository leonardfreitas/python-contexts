# Django Admin

## What the Admin is (and isn't)

**Is:** an internal operations tool — support staff investigating a ticket,
a rare administrative action (approve/correct something), a quick read-only
view for non-developers.

**Isn't:** a product-facing dashboard for end users, an "alternative API"
used via automation, or a place where business logic lives.

## Golden rule

> The Admin never contains its own business logic. If an admin action needs
> to do anything beyond a simple field CRUD, it calls the same service the
> API calls.

```python
# ❌ business logic leaking into the admin
class OrderAdmin(admin.ModelAdmin):
    def save_model(self, request, obj, form, change):
        if obj.status == "cancelled":
            send_cancellation_email(obj)  # duplicates what already lives in services/
        super().save_model(request, obj, form, change)
```

```python
# ✅ admin delegates to the same service the view uses
class OrderAdmin(admin.ModelAdmin):
    actions = ["cancel_selected"]

    @admin.action(description="Cancel selected orders")
    def cancel_selected(self, request, queryset):
        for order in queryset:
            try:
                cancel_order(order, reason="Cancelled via admin", cancelled_by=request.user)
            except DomainError as e:
                self.message_user(request, f"Error on {order.id}: {e.message}", level="ERROR")
```

The Admin is just another "entry point" into the domain, same as `api/` —
never the owner of the logic.

## What to expose

Only register a model in the admin if someone will actually operate it
manually there. A purely supporting model (a join table, an append-only
audit log, a derived cache entry) doesn't need
`admin.site.register()`.

## Avoiding N+1 in list views

The admin does not automatically apply `select_related`/`prefetch_related`:

```python
class OrderAdmin(admin.ModelAdmin):
    list_display = ["id", "customer_name", "status", "total"]
    list_select_related = ["customer"]

    def customer_name(self, obj):
        return obj.customer.name
```

> Any `ModelAdmin` that shows a relation field in `list_display` MUST
> declare the corresponding `list_select_related`.

## Permissions and read-only fields

- No day-to-day account uses `is_superuser` — that's reserved for
  emergencies/initial setup
- Role-based groups (e.g. `support`, `finance`, `operations`) with granular
  `Permission`s per model
- A field that only a service should change (e.g. `status`, `groups` — see
  `permissions.md`) is set as `readonly_fields`, forcing the flow through
  the custom action (which calls the service)

```python
class OrderAdmin(admin.ModelAdmin):
    readonly_fields = ["status", "created_at", "updated_at"]
```

## The Admin in production

Keep it enabled, but protected: mandatory 2FA for staff accounts, IP
allowlisting if access is mostly from a known network, a restricted
`ALLOWED_HOSTS`, access audit logging, and liberal use of `readonly_fields`
to reduce blast radius if an account is compromised.

## Summary

| Rule | Reason |
|---|---|
| The Admin never contains its own business logic — always delegates to `services/` | Avoids duplicated/diverging logic |
| Only register models someone will actually operate manually | Reduces surface for mistakes |
| `list_select_related` required if `list_display` shows a relation | Avoids N+1 |
| Fields only a service should change go in `readonly_fields` | Forces the flow through business rules |
| No day-to-day account uses `is_superuser` | Principle of least privilege |
| Mandatory 2FA for staff in production | The Admin is a common attack target |
