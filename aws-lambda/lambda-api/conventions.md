# Conventions

## Services: functions, not classes

Same rule as the rest of this collection: a function is the default. A
class is only justified when the service needs to hold configurable state
across calls (e.g. an injected third-party client). The litmus test: if
the logic would need to run outside an HTTP route too — from a queue
consumer, from a test with no FastAPI app spun up — it's a service.

## One file per operation, in every layer

```
routes/
  orders.py           # all order-related endpoints for this context
  order_items.py

services/
  create_order.py      # create_order
  cancel_order.py        # cancel_order
  queries.py               # get_order, list_orders_for_customer

schemas/
  order_create.py            # OrderCreateSchema — request body
  order_response.py            # OrderResponseSchema — response body
```

Private helpers stay in the same file as the public function that uses
them, prefixed with `_`, and are never tested directly — only the public
function's behavior is tested.

## Schemas split by operation, not one schema per model

Same reasoning as separating serializers by intent in a server-based
context: a request schema for creation is not the same contract as a
response schema for reading. Splitting them prevents a create schema from
accidentally accepting a field it shouldn't (like a server-managed
`status`), and keeps response payloads free of internal fields that were
never meant to leave the service.

```python
# schemas/order_create.py
class OrderCreateSchema(BaseModel):
    customer_id: str
    items: list[OrderItemSchema]

# schemas/order_response.py
class OrderResponseSchema(BaseModel):
    id: str
    status: str
    total: Decimal
    created_at: datetime
```

## Route naming

- Resources always plural: `/orders/`, not `/order/`
- Business action as a verb suffix on the resource: `/orders/{id}/cancel`
- One `APIRouter` per route file, included into `main.py`:

```python
# app/routes/orders.py
router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("", response_model=OrderResponseSchema, status_code=201)
def create_order_endpoint(payload: OrderCreateSchema):
    order = create_order(**payload.model_dump())
    return order
```

```python
# app/main.py
app = FastAPI()
app.include_router(orders.router)
```

## Environment variables

Same principle as elsewhere in this collection: configuration is never
read via `os.environ.get()` scattered through the codebase. Centralize
env var access in a single settings module (a pydantic `BaseSettings`
class is the natural fit here too), with explicit types and defaults —
or no default, if the value is required for the app to function.
