# URLs and Routers

## Router only where `ModelViewSet` fits

`DefaultRouter`/`SimpleRouter` generates URLs from a `ViewSet` — it assumes
honest CRUD. A router only makes sense where `ModelViewSet` was already
the right choice.

```python
# api/urls.py
router = DefaultRouter()
router.register("orders", OrderViewSet, basename="order")

urlpatterns = router.urls
```

## Where the Router becomes a problem

Routers encourage a "God ViewSet" — it's too convenient to bolt on one more
`@action` instead of creating a new `APIView`:

```python
# ❌ God ViewSet
class OrderViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=["post"])
    def cancel(self, request, pk=None): ...

    @action(detail=True, methods=["post"])
    def refund(self, request, pk=None): ...

    @action(detail=True, methods=["post"])
    def confirm(self, request, pk=None): ...
```

URLs generated via decorators are less explicit than a hand-written
`urls.py`, and reinforce coupling into one oversized ViewSet.

## Recommended split

**Router (`DefaultRouter` + `ModelViewSet`)** → only the resource's plain
CRUD.

**Manual `path()` + a dedicated `APIView`** → any business action that
isn't CRUD:

```python
urlpatterns = [
    *router.urls,
    path("orders/<uuid:pk>/cancel/", OrderCancelView.as_view(), name="order-cancel"),
    path("orders/<uuid:pk>/refund/", OrderRefundView.as_view(), name="order-refund"),
]
```

**Why this is better:**
- Visibility — opening `urls.py` shows the full list of endpoints without
  reading a ViewSet's body looking for `@action`
- One view = one file/class = one focused test
- Reinforces: CRUD is generic and replicable (a router handles it),
  a business action is specific and deserves explicit code

## Rule

> Router + `ModelViewSet` only for the 5 standard CRUD operations. Any
> action representing a named business rule (cancel, approve, confirm,
> archive) becomes a dedicated `APIView` with an explicit `path()` —
> never an `@action` inside the ViewSet.
>
> If a resource doesn't have full CRUD (e.g. it can only be created and
> listed, never deleted), don't force `ModelViewSet` — use
> `GenericAPIView` + only the mixins that apply, avoiding exposing methods
> the business rules don't even allow.

## Naming

- Resources are always plural: `/orders/`, not `/order/`
- Business action as a verb suffix: `/orders/{id}/cancel/` — not
  `/orders/{id}/actions/cancel/` (too verbose) nor `/cancel-order/{id}/`
  (verb before the resource breaks the REST hierarchy)
- Router `basename` is always singular

## Versioning

Prefix at the project's root URL, not inside each app:

```python
# config/urls.py
urlpatterns = [
    path("api/v1/", include("config.urls_v1")),
]
```

Keeps each app from needing to know which version it's in — versioning is
a project-wide decision.
