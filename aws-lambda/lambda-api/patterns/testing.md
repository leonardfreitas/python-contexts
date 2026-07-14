# Testing

## Bypass Mangum entirely for route-level tests

`fastapi.testclient.TestClient` talks to the FastAPI app directly — no
Mangum, no simulated API Gateway event needed:

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_order_returns_409_on_insufficient_stock():
    response = client.post("/orders", json={"items": []})
    assert response.status_code == 409
    assert response.json()["error"]["code"] == "InsufficientStockError"
```

This is the fast path for route-level integration tests — it validates
routing, request/response schema validation, and the exception-handler
translation, without any Lambda-specific machinery in the loop.

Simulating a real API Gateway event end-to-end (via `sam local
start-api` or a deployed environment) is reserved for a small number of
smoke tests, not the default test suite.

## Testing pyramid by layer

| Layer | Test type | Real AWS calls? | Goes through Mangum? |
|---|---|---|---|
| Service | Unit | No (moto if it touches AWS) | No |
| Repository (data layer) | Unit/Integration | No (moto, or a local database) | No |
| Route | Integration | No (moto if the route touches AWS indirectly via a service) | No — `TestClient` |
| End-to-end smoke test | Integration | Real, against a deployed environment | Yes |

### Services → unit test, no HTTP

```python
def test_create_order_raises_on_insufficient_stock():
    with pytest.raises(InsufficientStockError):
        create_order(customer_id="123", items=[OrderItem(product_id="p1", quantity=999)])
```

### Routes → integration test via `TestClient`

Doesn't re-test every business rule from scratch — just the happy path
plus one example per error category, to confirm the glue between route →
service → exception handler → HTTP response works.

## Mocking AWS with moto

`moto` intercepts boto3 calls in-process and simulates the AWS service's
behavior — no real AWS account, no network call, no LocalStack container
required for most cases.

```python
from moto import mock_aws
import boto3

@mock_aws
def test_create_order_persists_to_dynamodb():
    # inside create_order(), a real boto3 call hits moto's in-memory fake
    dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
    dynamodb.create_table(
        TableName="orders",
        KeySchema=[{"AttributeName": "PK", "KeyType": "HASH"}],
        AttributeDefinitions=[{"AttributeName": "PK", "AttributeType": "S"}],
        BillingMode="PAY_PER_REQUEST",
    )

    order = create_order(customer_id="123", items=[...])
    assert order.id is not None
```

## Rule

> Tests never call real AWS. `moto` covers unit and integration tests for
> any AWS-touching logic. A local database (DynamoDB Local, or a test
> Postgres instance — see the data-access pattern files) is a convenience
> for local development, never a requirement for the test suite to pass in
> CI.

## Factories/fixtures

Same principle as elsewhere in this collection: keep fixtures close to the
domain they build, avoid a single shared fixtures file across unrelated
projects. Since one project here already equals one bounded context,
`tests/conftest.py` at the project root is the natural place for shared
fixtures — there's no cross-app sprawl to guard against the way there is
in a multi-app framework.
