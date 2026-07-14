# Conventions

## Services: functions, not classes

Use a function by default. A class is justified only when the service
needs to hold configurable state across calls (e.g. an injected HTTP
client, a payment gateway pointing at sandbox vs production). "It could be
a class" isn't a good enough reason — "it needs to be a class" is.

```python
# ✅ default — function
def create_order(customer: Customer, items: list[OrderItem]) -> Order:
    ...

# ✅ valid exception — service with configurable dependency
class PaymentGateway:
    def __init__(self, client: PaymentClient):
        self.client = client

    def charge(self, amount: Decimal, method: PaymentMethod) -> Charge:
        ...
```

**Litmus test to decide if something is a service:** if the logic needs to
exist outside the context of an HTTP request (a management command, a
worker task, a test that doesn't spin up the full request/response cycle),
it's a service.

## Organizing service files

By domain, one file per area of action, verb_noun naming:

```
services/
  create.py       # create_order
  cancel.py        # cancel_order
  queries.py         # get_pending_orders, get_customer_orders
```

Never an `OrderService` with 15 unrelated methods just because they touch
the same model — that's a "God Service."

Private helpers (`_has_available_stock`) live in the same file as the
public function that uses them, prefixed with `_`, and are never exported
or tested directly — only the public behavior is tested.

## Multiple serializers per model — yes, almost always

Split by **intent**, not by model:

```
serializers/
  list.py      # lightweight fields, tuned for listing performance
  detail.py     # everything, including nested relations
  create.py      # only the fields a client is allowed to send on creation
  update.py       # usually a subset of create
```

Name serializers after the operation (`OrderCreateSerializer`), not a
generic model-wide serializer reused everywhere.

**Why this matters:**
- Security — an update serializer that accidentally accepts `status`
  becomes a way for a client to change state directly, bypassing the
  service
- Performance — the list serializer shouldn't serialize heavy nested
  relations that only the detail view needs (avoids silent N+1 queries)
- Contract clarity — the create serializer documents exactly what the API
  accepts

## ModelViewSet is not a universal default

- Simple CRUD, no heavy business logic → `ModelViewSet`
- Specific business action (cancel, approve, archive) → dedicated `APIView`
- Never let a ViewSet turn into a drawer of custom `@action`s (a "God
  ViewSet")

See `patterns/urls-and-routers.md` for the full breakdown of Router vs
manual `path()`.

## Route naming

- Resources are always plural: `/orders/`, not `/order/`
- Business action as a verb suffix: `/orders/{id}/cancel/`
- Router `basename` is always singular (`basename="order"`)

## Environment variables

No configuration value is read via `os.environ.get()` directly outside of
`config/settings/env.py`. Every env var goes through the pydantic
settings class, with an explicit type and default (or no default, if
required).
