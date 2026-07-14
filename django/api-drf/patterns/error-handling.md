# Error Handling

## The problem

Without a pattern, every view accumulates `try/except` blocks repeating the
same logic — "if it's this error, return 409, if it's that one, return
404." That's pure repetition, it couples the view to every possible domain
error, and one forgotten `except` leaks out as a generic 500.

## The solution: typed exceptions + a central exception handler

### Step 1 — A domain exception hierarchy

Golden rule: **a domain exception knows nothing about HTTP.**

```python
# core/exceptions.py
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

Specific errors inherit from the right category — this is what lets you
map **category → HTTP status** in one place, without listing every specific
exception:

```python
# apps/orders/exceptions.py
class InsufficientStockError(ConflictError):
    default_message = "Not enough stock to fulfill this order"

class OrderAlreadyShippedError(ValidationError):
    default_message = "Order has already shipped and cannot be modified"
```

### Step 2 — Central DRF exception handler

```python
# core/exception_handler.py
from rest_framework.views import exception_handler as drf_default_handler
from rest_framework.response import Response

_STATUS_MAP = {
    NotFoundError: 404,
    ConflictError: 409,
    ValidationError: 422,
    PermissionDeniedError: 403,
}

def custom_exception_handler(exc, context):
    if isinstance(exc, DomainError):
        status_code = next(
            (code for cls, code in _STATUS_MAP.items() if isinstance(exc, cls)),
            400,
        )
        return Response(
            {"error": {"message": exc.message, "code": exc.__class__.__name__, **exc.context}},
            status=status_code,
        )

    return drf_default_handler(exc, context)
```

```python
# settings
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "core.exception_handler.custom_exception_handler",
}
```

### Result — the view stays clean, no try/except

```python
class OrderCreateView(APIView):
    def post(self, request):
        serializer = OrderCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = serializer.save(created_by=request.user)
        return Response(OrderDetailSerializer(order).data, status=201)
```

If `create_order` raises `InsufficientStockError`, the exception handler
intercepts it automatically and returns a formatted 409. The view doesn't
need to know this error exists.

## Why not raise `rest_framework.exceptions.APIException` directly from a service

- ❌ Couples the domain layer to DRF. If a management command or a worker
  task calls `create_order`, it would have to deal with an exception that
  talks about HTTP status codes — meaningless outside of a request.
- ✅ With pure domain exceptions, the same service behaves identically
  whether called from a view, a command, or a test — only the exception
  handler translates to HTTP.

## Rule

> Services never import anything from `rest_framework` or `django.http`.
> Business errors are always a `DomainError` (or subclass). The
> translation to HTTP happens in exactly one place: the central exception
> handler.

Worth automating as an architecture test in CI: fail the build if
`services/` imports `rest_framework`.
