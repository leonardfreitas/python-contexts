# Testing

## Testing pyramid by layer

| Layer | Test type | Real AWS? |
|---|---|---|
| Service | Unit | No (moto if it touches AWS) |
| Message parsing/validation | Unit | No |
| Handler (`BatchProcessor` wiring) | Integration | No — invoke `handler(event, context)` directly with a plain Python dict simulating the event |
| Partial failure behavior | Integration | No — same as above, asserting on `batchItemFailures` |

There is no equivalent of `TestClient` here (no HTTP framework involved)
— the handler function itself, called with a plain dict, is already the
fastest integration test surface available.

## Services → unit test, no batch machinery involved

```python
def test_handle_order_created_saves_snapshot():
    message = OrderCreatedMessage(order_id="123", customer_id="c1", items=[...], created_at=now())
    handle_order_created(message)
    assert repository.order_exists("123")
```

## Handler → integration test, simulating a batch event as a plain dict

```python
def test_handler_reports_only_failed_message_in_batch():
    event = {
        "Records": [
            {"messageId": "1", "body": json.dumps({"type": "OrderCreated", "payload": valid_payload})},
            {"messageId": "2", "body": json.dumps({"type": "OrderCreated", "payload": malformed_payload})},
        ]
    }

    response = handler(event, context=MockLambdaContext())

    assert response["batchItemFailures"] == [{"itemIdentifier": "2"}]
```

This test validates the most important behavior in this entire context: a
failing message in a batch doesn't take down the ones that succeeded, and
the failure is reported with the correct `messageId`.

## Testing idempotency

```python
@mock_aws
def test_handle_order_created_is_idempotent(dynamodb_idempotency_table):
    message = OrderCreatedMessage(order_id="123", ...)

    handle_order_created(message)
    handle_order_created(message)  # same message, called twice

    assert repository.count_order_snapshots("123") == 1
```

Requires the idempotency DynamoDB table to exist in the test environment
— create it via `moto` in a fixture, mirroring the table definition in
`template.yaml`.

## Mocking AWS with moto

Same approach as any other context in this collection: `moto` intercepts
boto3 calls in-process, no real AWS account or network call needed.

```python
from moto import mock_aws

@mock_aws
def test_handle_order_created_persists_to_dynamodb():
    # setup: create the tables this test needs via boto3, same as in template.yaml
    ...
    handle_order_created(message)
    ...
```

## Rule

> Tests never call real AWS or a real SQS queue. `moto` covers every test
> that touches AWS, including the idempotency table. `sam local invoke`
> with a sample event file (see `patterns/local-development.md`) is a
> local development convenience, never a requirement for the automated
> test suite to pass in CI.
