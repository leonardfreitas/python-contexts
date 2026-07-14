# Scaffolding a New Module

Step-by-step to add a new domain (`apps/<domain>/`), tying together every
decision from `architecture.md`, `conventions.md`, and `rules.md`.

## 1. Folder structure

```
apps/<domain>/
  __init__.py
  apps.py
  models.py
  migrations/
  services/
    __init__.py
  exceptions.py
  api/
    __init__.py
    views.py
    urls.py
    serializers/
      __init__.py
  admin.py
  factories.py
  tests/
    test_services.py
    test_views.py
    test_serializers.py
```

## 2. Model

Inherit from `core.models.TimestampedModel` where applicable. Declare
`Meta.permissions` at this point for any named business action the domain
will have (see `permissions.md`):

```python
class Order(TimestampedModel):
    class Meta:
        permissions = [
            ("cancel_order", "Can cancel order"),
        ]
```

## 3. Domain exceptions

Before writing the first service, map out the possible business errors in
`exceptions.py`, inheriting from the right category (see
`patterns/error-handling.md`):

```python
class InsufficientStockError(ConflictError):
    default_message = "Not enough stock to fulfill this order"
```

## 4. Services

One file per operation, functions (not classes, unless there's a
configurable dependency):

```python
# services/create.py
def create_order(customer: Customer, items: list[OrderItem]) -> Order:
    if not _has_available_stock(items):
        raise InsufficientStockError()

    with transaction.atomic():
        order = Order.objects.create(...)

    transaction.on_commit(lambda: notify_order_created.delay(order.id))
    return order


def _has_available_stock(items) -> bool:
    return all(item.product.stock >= item.quantity for item in items)
```

## 5. Serializers — one per operation

```python
# api/serializers/create.py
class OrderCreateSerializer(serializers.Serializer):
    customer_id = serializers.UUIDField()
    items = OrderItemSerializer(many=True)

    def save(self, **kwargs):
        return create_order(**self.validated_data, **kwargs)
```

## 6. Views + URLs

Plain CRUD → `ModelViewSet` + router. Business action → dedicated `APIView`
+ explicit `path()` (see `patterns/urls-and-routers.md`).

```python
# api/views.py
class OrderCancelView(APIView):
    permission_classes = [IsAuthenticated, HasPermission("orders.cancel_order")]

    def post(self, request, pk):
        order = get_object_or_404(Order, pk=pk)
        cancel_order(order, cancelled_by=request.user, reason=request.data.get("reason", ""))
        return Response(status=204)
```

```python
# api/urls.py
router = DefaultRouter()
router.register("orders", OrderViewSet, basename="order")

urlpatterns = [
    *router.urls,
    path("orders/<uuid:pk>/cancel/", OrderCancelView.as_view(), name="order-cancel"),
]
```

Register in `config/urls.py`:
```python
path("api/v1/orders/", include("apps.orders.api.urls")),
```

## 7. Admin

```python
class OrderAdmin(admin.ModelAdmin):
    list_display = ["id", "customer_name", "status"]
    list_select_related = ["customer"]
    readonly_fields = ["status"]
    actions = ["cancel_selected"]
```

## 8. Factory

```python
class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order
    customer = factory.SubFactory(CustomerFactory)
```

## 9. Tests — in pyramid order

1. `test_services.py` — unit test, real database, no HTTP
2. `test_serializers.py` — isolated unit test (no database if possible)
3. `test_views.py` — integration test via `APIClient`, HTTP contract

## Final checklist

- [ ] Domain exceptions inherit from `DomainError`/the right category
- [ ] Services are functions, no `rest_framework` import
- [ ] `atomic()` only where there are 2+ related writes
- [ ] External side effects go through `transaction.on_commit()`
- [ ] Serializer named after the operation, no business logic in `create()`/`update()`
- [ ] Non-CRUD business action is an `APIView`, not an `@action`
- [ ] Custom permission declared in `Meta.permissions` if applicable
- [ ] `list_select_related` in the admin if `list_display` shows a relation
- [ ] Service tests don't depend on HTTP; view tests don't re-test every business rule
