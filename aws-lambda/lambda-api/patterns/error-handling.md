# Error Handling

## The pattern, translated from a central exception handler to FastAPI's mechanism

Same philosophy used elsewhere in this collection: business errors are
typed domain exceptions, and exactly one place in the codebase knows how
to translate them into an HTTP response.

### Step 1 — Domain exception hierarchy

```python
# app/exceptions.py
class DomainError(Exception):
    """Base class for any business rule error."""
    default_message = "Domain error"

    def __init__(self, message: str = None, **context):
        self.message = message or self.default_message
        self.context = context
        super().__init__(self.message)


class NotFoundError(DomainError):
    default_message = "Resource not found"

class ConflictError(DomainError):
    default_message = "State conflict"

class ValidationError(DomainError):
    default_message = "Business rule violated"

class PermissionDeniedError(DomainError):
    default_message = "Action not allowed"
```

Specific errors inherit from the right category:

```python
# app/exceptions.py (continued)
class InsufficientStockError(ConflictError):
    default_message = "Not enough stock to fulfill this order"

class OrderAlreadyShippedError(ValidationError):
    default_message = "Order has already shipped and cannot be modified"
```

### Step 2 — A single exception handler registered on the FastAPI app

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from app.exceptions import DomainError, NotFoundError, ConflictError, ValidationError, PermissionDeniedError

app = FastAPI()

_STATUS_MAP = {
    NotFoundError: 404,
    ConflictError: 409,
    ValidationError: 422,
    PermissionDeniedError: 403,
}

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    status_code = next(
        (code for cls, code in _STATUS_MAP.items() if isinstance(exc, cls)),
        400,
    )
    return JSONResponse(
        status_code=status_code,
        content={"error": {"message": exc.message, "code": exc.__class__.__name__, **exc.context}},
    )
```

### Result — routes stay clean, no try/except

```python
@router.post("/{order_id}/cancel", status_code=204)
def cancel_order_endpoint(order_id: str):
    cancel_order(order_id=order_id)
```

If `cancel_order` raises `OrderAlreadyShippedError`, the exception handler
intercepts it automatically and returns a formatted 422. The route never
needs to know this error exists.

## Rule

> Services never import anything from `fastapi`. Business errors are
> always a `DomainError` (or subclass). The translation to HTTP happens in
> exactly one place: the exception handler registered on the app.

This keeps a service testable and callable from anywhere — a route today,
a queue consumer in a companion worker context tomorrow — without dragging
along any HTTP-specific concern.
