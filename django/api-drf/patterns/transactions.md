# Transactions

## Golden rule: transactions live in the service, never in the view

The view doesn't know (and shouldn't know) how many database operations
happen behind a service call. `@transaction.atomic` on a view assumes
knowledge of an internal implementation that can change, and it encourages
the view to call multiple services inside one transaction "for safety" —
a sign of orchestration that belongs in a service instead.

```python
# ❌ view deciding about transactions
class OrderCreateView(APIView):
    def post(self, request):
        with transaction.atomic():
            order = create_order(...)
            reserve_stock(...)
        return Response(...)
```

```python
# ✅ the service encapsulates atomicity
def create_order(customer, items) -> Order:
    with transaction.atomic():
        order = Order.objects.create(...)
        _reserve_stock(items)

    transaction.on_commit(lambda: notify_order_created.delay(order.id))
    return order
```

## Never fire external side effects inside a transaction

External side effect = anything that isn't the database itself: email,
webhook, background task, third-party API call.

**Why:** if the transaction rolls back after the task was already fired,
you've notified something about an event that doesn't exist. Depending on
isolation level, the task could even run before the commit happens at all.

**Solution:** `transaction.on_commit()`. Guarantees the callback only fires
if the transaction committed successfully. If a rollback happens, it never
runs.

> Any call to a background task, email, webhook, or external API inside a
> transactional flow MUST go through `transaction.on_commit()`. Never fire
> an external side effect directly inside an `atomic()` block.

## Nested transactions — a service calling a service

```python
def create_order(customer, items, payment_method) -> Order:
    with transaction.atomic():
        order = Order.objects.create(...)
        charge_payment(customer, amount, payment_method)  # has its own atomic() inside
    return order
```

Django handles this via **savepoints** — the inner `atomic()` becomes a
savepoint inside the outer transaction (Postgres has no true nested
transaction).

**Gotcha:** if a service silently swallows a database exception internally
(a try/except that falls back to something else), the savepoint can end up
in an inconsistent state and Django raises `TransactionManagementError` on
the next query.

> If a service can be called both standalone and from inside another
> service, it never silently swallows a database exception — let it
> propagate, the outer layer decides what to do.

## When NOT to use `atomic()`

If a service does a single write (`Model.objects.create()` alone), Django
already treats that atomically by default in autocommit mode — wrapping it
in an extra `atomic()` is noise with no benefit.

> `atomic()` is required when a service performs 2+ related writes that
> need to be all-or-nothing. For a single write, it's not needed.

## Summary

| Situation | Rule |
|---|---|
| Where to declare `atomic()` | Always in the service, never in the view |
| Service with a single write | No explicit `atomic()` needed |
| Service with 2+ related writes | `atomic()` required |
| Background task/email/webhook in the flow | Always via `transaction.on_commit()` |
| Service A calls service B (both atomic) | Fine — resolved via savepoints, neither one silently swallows a DB exception |
