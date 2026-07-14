# Data Access — DynamoDB

Use this pattern when the project persists data in DynamoDB. If the
project uses a relational database instead, use `data-access-sql.md` —
don't read both unless comparing the two for a new project decision.

## No connection pooling problem — but a different trade-off

DynamoDB is HTTP-based, with no concept of a persistent connection to
manage — the connection pooling problem that dominates the SQL pattern
simply doesn't exist here. The trade-off moves elsewhere: **access
patterns must be designed upfront**, because DynamoDB doesn't support
ad-hoc joins or arbitrary queries the way SQL does. Getting the key
structure wrong is expensive to fix later; getting a connection pool wrong
in SQL is comparatively cheap to adjust.

## `boto3.resource`, not `boto3.client`, for everyday item operations

`resource` gives a higher-level, more Pythonic interface (dict-like items,
automatic type conversion for common cases); `client` is lower-level and
closer to the raw API. Default to `resource` for regular CRUD-style item
operations; reach for `client` only when a specific low-level operation
isn't well covered by the resource interface.

```python
# app/repository.py
import boto3
from boto3.dynamodb.conditions import Key

table = boto3.resource("dynamodb").Table(settings.table_name)

def save_order(order: Order) -> None:
    table.put_item(Item={
        "PK": f"ORDER#{order.id}",
        "SK": "METADATA",
        **order.model_dump(),
    })

def get_order(order_id: str) -> Order | None:
    response = table.get_item(Key={"PK": f"ORDER#{order_id}", "SK": "METADATA"})
    item = response.get("Item")
    return Order(**item) if item else None

def list_orders_for_customer(customer_id: str) -> list[Order]:
    response = table.query(
        IndexName="GSI1",
        KeyConditionExpression=Key("GSI1PK").eq(f"CUSTOMER#{customer_id}"),
    )
    return [Order(**item) for item in response["Items"]]
```

## Single-table design — briefly

DynamoDB is generally designed with one table per bounded context (not one
table per entity), with `PK`/`SK` (and one or more Global Secondary
Indexes) encoding multiple entity types and their relationships within the
same table. This is a deliberate departure from relational thinking, and
reintroducing "one table per entity" tends to reintroduce the need for
joins DynamoDB isn't built to do efficiently.

The concrete key design is project-specific and depends on the access
patterns identified upfront — this pattern file establishes the
convention (design keys around access patterns, use GSIs for alternate
query shapes like "orders for a customer"), not a fixed schema.

## Pagination

DynamoDB queries return a `LastEvaluatedKey` when there's more data than
fits the response — always handle this explicitly rather than assuming a
single `query`/`scan` call returns everything:

```python
def list_orders_for_customer(customer_id: str) -> list[Order]:
    items = []
    kwargs = {"IndexName": "GSI1", "KeyConditionExpression": Key("GSI1PK").eq(f"CUSTOMER#{customer_id}")}
    while True:
        response = table.query(**kwargs)
        items.extend(response["Items"])
        if "LastEvaluatedKey" not in response:
            break
        kwargs["ExclusiveStartKey"] = response["LastEvaluatedKey"]
    return [Order(**item) for item in items]
```

## Batch operations

Use `batch_writer()` for bulk writes instead of looping individual
`put_item` calls — it batches requests and handles retries for
unprocessed items automatically:

```python
def save_orders(orders: list[Order]) -> None:
    with table.batch_writer() as batch:
        for order in orders:
            batch.put_item(Item=order.model_dump())
```

## TTL for ephemeral data

If a domain has naturally expiring data (e.g. a session token, a
short-lived cache entry), use DynamoDB's native TTL feature (an epoch
timestamp attribute configured on the table) rather than a manual cleanup
job — it's free, native, and doesn't require any extra infrastructure.

## Local development

Run DynamoDB Local via Docker Compose:

```yaml
# docker-compose.yml (local dev only, not part of the deployed app)
services:
  dynamodb-local:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
```

Point the app at it via an endpoint override, active only when running
locally:

```python
import boto3

dynamodb = boto3.resource(
    "dynamodb",
    endpoint_url=settings.dynamodb_endpoint_url,  # None in deployed environments, set locally
)
```

## Summary

| Rule | Reason |
|---|---|
| `boto3.resource` for everyday item operations | Higher-level, more Pythonic interface |
| Design keys/GSIs around access patterns identified upfront | DynamoDB has no ad-hoc joins; the wrong key design is expensive to fix later |
| One table per bounded context, not one per entity | Matches DynamoDB's single-table design philosophy |
| Always handle `LastEvaluatedKey` | A single `query`/`scan` call doesn't guarantee all matching data |
| `batch_writer()` for bulk writes | Automatic batching and retry handling for unprocessed items |
| Native TTL for ephemeral data | Free, built-in, no extra cleanup infrastructure needed |
