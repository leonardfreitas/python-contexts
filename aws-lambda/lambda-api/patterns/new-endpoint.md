# Adding a New Endpoint

Step-by-step for adding a new route within an existing bounded-context
Lambda. If the new functionality actually belongs to a different bounded
context, don't add it here — start a new project instead (see
`architecture.md`'s bounded-context rule).

## 1. Domain exceptions (if this endpoint introduces new business errors)

```python
# app/exceptions.py
class OrderAlreadyShippedError(ValidationError):
    default_message = "Order has already shipped and cannot be modified"
```

## 2. Schemas — one for the request, one for the response

```python
# app/schemas/order_cancel.py
class OrderCancelSchema(BaseModel):
    reason: str | None = None

# app/schemas/order_response.py (likely already exists, reused here)
class OrderResponseSchema(BaseModel):
    id: str
    status: str
```

## 3. Service — the business logic, as a function

```python
# app/services/cancel_order.py
def cancel_order(order_id: str, reason: str | None) -> Order:
    order = repository.get_order(order_id)
    if order is None:
        raise NotFoundError(f"Order {order_id} not found")
    if order.status == "shipped":
        raise OrderAlreadyShippedError(order_id)

    order.status = "cancelled"
    repository.save_order(order)
    return order
```

## 4. Route — thin, delegates immediately

```python
# app/routes/orders.py
@router.post("/{order_id}/cancel", response_model=OrderResponseSchema)
def cancel_order_endpoint(order_id: str, payload: OrderCancelSchema):
    return cancel_order(order_id=order_id, reason=payload.reason)
```

If this is a new route file entirely (new sub-resource), remember to
include the router in `app/main.py`:

```python
app.include_router(orders.router)
```

## 5. Tests — service first, then the route

```python
# tests/test_services.py
def test_cancel_order_raises_if_already_shipped():
    order = OrderFactory(status="shipped")
    with pytest.raises(OrderAlreadyShippedError):
        cancel_order(order_id=order.id, reason=None)
```

```python
# tests/test_routes.py
def test_cancel_order_endpoint_returns_422_if_already_shipped(client):
    order = OrderFactory(status="shipped")
    response = client.post(f"/orders/{order.id}/cancel", json={})
    assert response.status_code == 422
```

## Checklist

- [ ] New business errors inherit from `DomainError`/the right category
- [ ] Request and response use separate schemas
- [ ] Service is a function, doesn't import anything from `fastapi`
- [ ] Route delegates immediately, no business logic inline
- [ ] Service has a unit test with no HTTP involved
- [ ] Route has an integration test via `TestClient`, no Mangum involved
- [ ] Confirm this endpoint genuinely belongs to this bounded context (see `architecture.md`) before adding it here
