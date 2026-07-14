# Testing

## The `on_commit()` gotcha in tests

`TestCase` / `pytest-django` (the normal `db` fixture) wraps each test in a
transaction with a rollback at the end — fast, but `transaction.on_commit()`
**never fires**, because the transaction is never actually committed.

**Solution:** `TransactionTestCase` /
`@pytest.mark.django_db(transaction=True)`, which commits for real and
cleans up by truncating tables afterward.

```python
@pytest.mark.django_db(transaction=True)
def test_create_order_notifies_on_commit(mocker):
    mock_task = mocker.patch("orders.services.notify_order_created.delay")

    order = create_order(customer=customer, items=items)

    mock_task.assert_called_once_with(order.id)
```

**Trade-off:** `transaction=True` is significantly slower (table truncation
instead of rollback). Use it only when a test specifically needs to verify
`on_commit()` behavior — it shouldn't become the default for the whole
suite.

## Testing pyramid by layer

| Layer | Test type | Database? | HTTP? |
|---|---|---|---|
| Model/Manager | Unit | Yes (real) | No |
| Serializer | Unit | Only if a `SerializerMethodField` hits the DB | No |
| Service | Unit/Integration | Yes (real) | No |
| Service with `on_commit()` | Integration | Yes (`transaction=True`) | No |
| View | Integration | Yes (real) | Yes (`APIClient`) |

The closer to the domain, the more isolated and fast the test. The closer
to the HTTP edge, the "thicker" the test (only covers the contract, doesn't
re-explain business logic).

### Services → unit test, real database, no HTTP

```python
@pytest.mark.django_db
def test_create_order_raises_on_insufficient_stock():
    product = ProductFactory(stock=0)

    with pytest.raises(InsufficientStockError):
        create_order(customer=CustomerFactory(), items=[OrderItem(product, quantity=1)])
```

No `APIClient`, no JSON payload, no view spun up.

### Views → integration test via `APIClient`, validating the HTTP contract

```python
@pytest.mark.django_db
def test_create_order_returns_409_on_insufficient_stock(api_client):
    product = ProductFactory(stock=0)

    response = api_client.post("/api/orders/", {"items": [{"product_id": product.id, "quantity": 1}]})

    assert response.status_code == 409
    assert response.json()["error"]["code"] == "InsufficientStockError"
```

Doesn't re-test every business rule — just the happy path plus one example
per error category (404, 409, 422), making sure the "glue" between view →
service → exception handler → HTTP actually works.

### Serializers → isolated unit test, no database when possible

```python
def test_order_create_serializer_rejects_empty_items():
    serializer = OrderCreateSerializer(data={"items": []})
    assert not serializer.is_valid()
    assert "items" in serializer.errors
```

If a serializer test suddenly needs the database, that's a sign business
logic leaked into the serializer.

## Mocking: external boundaries, never the ORM

> Mock external boundaries (background task calls, third-party HTTP
> clients, payment gateways). Never mock the Django ORM. The urge to mock
> a query is a sign the test should use a real database (fast enough with
> a test Postgres instance) instead of incorrectly simulating ORM behavior.

## Factories

Factory Boy, one per model, living next to the app
(`apps/orders/factories.py`), not centralized in one giant shared file —
avoids cross-domain coupling between independent apps.

```python
class CustomerFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Customer

    name = factory.Faker("name")
    email = factory.Faker("email")
```
