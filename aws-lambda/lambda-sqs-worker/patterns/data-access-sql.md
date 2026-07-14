# Data Access — SQL

Use this pattern when the worker persists business data in a relational
database (e.g. RDS/Postgres). If the project uses DynamoDB instead, use
`data-access-dynamodb.md`.

## The same connection pooling problem, with a batch-shaped twist

The core issue is identical to any other Lambda context using SQL: each
cold start can open a new database connection, and concurrent invocations
can exhaust the database's connection limit. Here it has an additional
wrinkle — a single invocation processes a **batch** of messages, so a
naive implementation might open a connection per message inside the loop
rather than once per invocation.

```python
# ❌ opens a new connection per message in the batch
def record_handler(record):
    engine = create_engine(settings.database_url)  # wrong — inside the loop
    ...
```

```python
# ✅ engine created once, at module scope — reused across the whole batch, and across warm invocations
# app/repository.py
engine = create_engine(settings.database_url, pool_pre_ping=True, pool_size=1)
```

## RDS Proxy — same recommendation as any SQL-backed Lambda context

Treat RDS Proxy as close to mandatory once the worker handles real
production message volume — it pools connections across concurrent
Lambda instances so the database itself only sees a small, stable number
of connections regardless of concurrency.

## Repository pattern

```python
# app/repository.py
from sqlalchemy.orm import Session

def save_order_snapshot(session: Session, order_id: str, items: list[dict]) -> None:
    session.add(OrderSnapshotModel(order_id=order_id, items=items))
    session.commit()

def order_exists(session: Session, order_id: str) -> bool:
    return session.query(OrderSnapshotModel).filter_by(order_id=order_id).first() is not None
```

Open one `Session` per invocation (not per message in the batch, unless a
specific message's failure should not roll back another message's already
-committed work — which is the common case, since each message is
reported as an independent batch item failure).

## Migrations

Same rule as any other SQL-backed context here: migrations run as an
explicit deploy step (e.g. via Alembic), never inside the handler or at
import time.

## Local development

```yaml
# docker-compose.yml (local dev only)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: local
    ports:
      - "5432:5432"
```

Point `DATABASE_URL` in `.env.dev` at this local container. Combine with
`scripts/invoke-local.sh` (see `patterns/local-development.md`) to test a
message-processing flow against a real local database.

## Summary

| Rule | Reason |
|---|---|
| Engine created once at module scope, never inside the per-message loop | Avoids opening a new connection per message within a single batch |
| RDS Proxy for real production message volume | Prevents exhausting the database's connection limit under concurrent invocations |
| One `Session` per invocation | Keeps one message's failure from rolling back another message's already-committed work |
| Migrations run as a deploy step, never on cold start | Avoids concurrent cold starts racing to apply the same migration |
