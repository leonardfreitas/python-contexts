# Data Access — SQL

Use this pattern when the project persists data in a relational database
(e.g. RDS/Postgres). If the project uses DynamoDB instead, use
`data-access-dynamodb.md` — don't read both unless comparing the two for a
new project decision.

## The connection pooling problem — the most important thing to get right

This is the single biggest way SQL-in-Lambda differs from SQL in a
traditional long-running server.

**The problem:** every Lambda cold start can open a new database
connection. Under concurrent load — dozens or hundreds of simultaneous
invocations — you can exhaust the database's connection limit in seconds.
A traditional server holds one stable connection pool for its entire
lifetime; Lambda has no equivalent of "lifetime" to hold a pool across —
every concurrent invocation is potentially its own connection.

## The fix, in order of maturity

### 1. RDS Proxy — treat as close to mandatory once traffic is real

RDS Proxy sits between Lambda and RDS, pooling and reusing connections
across invocations so the database itself only ever sees a small, stable
number of connections regardless of how many Lambda instances are
concurrently running. This is the AWS-native answer to the problem above,
and for any project expecting real concurrent traffic, it should be
considered required infrastructure, not an optional add-on.

### 2. Keep the engine/connection at module scope

Even with RDS Proxy in place, avoid creating a new SQLAlchemy engine
inside the handler function on every invocation. Create it once, at
import time, so a warm container reuses it:

```python
# app/repository.py
from sqlalchemy import create_engine

engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,
    pool_size=1,
)
```

`pool_size=1` is intentional: a single Lambda instance handles one
invocation at a time, so a larger local pool per instance just wastes
connections that RDS Proxy is already pooling upstream.

## Repository pattern

```python
# app/repository.py
from sqlalchemy.orm import Session

def save_order(session: Session, customer_id: str, items: list[OrderItem]) -> Order:
    order = OrderModel(customer_id=customer_id, items=[...])
    session.add(order)
    session.commit()
    return order.to_domain()

def get_order(session: Session, order_id: str) -> Order | None:
    row = session.get(OrderModel, order_id)
    return row.to_domain() if row else None
```

Services receive a `Session`, they don't construct one themselves — this
keeps transaction boundaries controlled by the caller (typically the
route, opening one session per request) rather than hidden inside the
repository.

## Migrations

Use Alembic. Run migrations as a deploy step, the same way a
traditionally-hosted app would — never on Lambda cold start. A migration
running as part of a cold start would run once per concurrent cold
container, causing the exact same race condition problem that connection
pooling causes at the database-connection level, but at the schema level
instead.

> Migrations run as an explicit step in the deploy script (or CI
> pipeline), before the new Lambda version is published — never inside
> the handler or at import time.

## Local development

Run a local Postgres via Docker Compose alongside `sam local start-api` or
plain Uvicorn:

```yaml
# docker-compose.yml (local dev only, not part of the deployed app)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: local
    ports:
      - "5432:5432"
```

Point `DATABASE_URL` in `.env.dev` at this local container.

## Summary

| Rule | Reason |
|---|---|
| RDS Proxy for any project with real concurrent traffic | Prevents exhausting the database's connection limit |
| Engine created once at module scope, not inside the handler | Warm containers reuse the connection instead of reconnecting every invocation |
| `pool_size=1` per Lambda instance | Avoids wasting connections already pooled upstream by RDS Proxy |
| Migrations run as a deploy step, never on cold start | Avoids concurrent cold starts racing to apply the same migration |
| Services receive a `Session`, don't construct one | Keeps transaction boundaries controlled by the caller |
