---
name: django-clean-drf
description: >
  Django REST Framework with Clean Architecture and SOLID. Covers greenfield APIs
  (views, use cases, services, models), query optimization, pagination/filtering,
  JWT auth, permissions, production setup, and refactoring legacy Django into this
  architecture. Use when building or extending Django APIs, applying domain-driven
  design, fixing N+1 queries, modernizing fat views, or applying Python 3.12+ /
  Django 5+ patterns. Not for Celery/async task queues.
---

# Django Clean DRF

Expert guidance for Django REST Framework apps using Clean Architecture: layered design, testability, and production-quality APIs — including transforming legacy code into that shape.

## Architecture Layers

```
HTTP Layer (Views/Serializers)
    ↓
Application Layer (Use Cases + DTOs)
    ↓
Domain Layer (Services + Models)
    ↓
Infrastructure Layer (ORM, External APIs, Cache)
```

Dependencies flow **inward only**. Outer layers depend on inner; never reverse.

## Mode selection

| Signals | Mode |
|---|---|
| New app/API, scaffold, CRUD, use case, service layer, "how should I structure…" | **Build** |
| Refactor, modernize, fat view, clean up legacy, SOLID pass, Python 3.12 / Django 5 upgrade | **Refactor** |
| Both (e.g. "refactor this endpoint then add X") | **Refactor first**, then **Build** for new pieces |

## When to use

**Build**
- "Create a new Django API for…"
- "Scaffold an app with clean architecture"
- "Create a use case / service for…"
- "Where should this business logic live?"
- "Design serializers / CRUD for this entity"

**Refactor**
- "Refactor this Django view/service"
- "Extract business logic from the view"
- "Fix N+1 / fat models / bare excepts"
- "Apply Clean Architecture to this module"
- "Modernize to Python 3.12 / Django 5"

## Instructions

### Shared: analyze context

1. Domain: entities, operations, business rules, external deps.
2. Project: existing apps, patterns already in use, Django/Python versions.
3. Scope: single use case vs full CRUD; new app vs extend; API-only vs admin.

### Mode: Build

1. **Load refs** as needed (see Bundled resources).
2. **Implement** with the patterns below (use case / service / thin view / serializer split).
3. **Validate** with the architecture, quality, and performance checklists.

### Mode: Refactor

Follow this loop until done — do not stop for confirmation mid-pass:

1. **Analyze** purpose, inputs, outputs, side effects.
2. **Identify issues**: long functions, deep nesting, duplication, logic in views, missing types, N+1, bare excepts, mutable defaults, magic values, poor names.
3. **Plan**: what becomes services/use cases; which queries; early returns; type hints.
4. **Execute incrementally** (one concern at a time):
   1. Extract view business logic → services/use cases  
   2. Fix N+1 (`select_related` / `prefetch_related`)  
   3. Deduplicate  
   4. Early returns  
   5. Split large functions  
   6. Type hints + docs  
   7. Python 3.12+ / Django 5+ upgrades  
5. **Preserve behavior**; run tests after each step.
6. **Load** `references/refactor/*` for transforms, anti-patterns, and language/framework upgrades. For target architecture detail, use the same architecture refs as Build.

**Refactor output format**
1. Summary — what and why  
2. Key changes — bullets  
3. Refactored code — complete, working  
4. Explanation — decisions  
5. Testing notes  

**Stop when**: single-purpose units; no duplication; nesting ≤2; functions focused (<25 lines when practical); public interfaces typed; N+1 gone; logic in services not views; tests still pass. If unsafe without more context, say so and ask.

### Core implementation patterns (Build; also the refactor target)

**Use case**
```python
from dataclasses import dataclass
from uuid import UUID
from django.db import transaction

@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list['OrderItemInput']
    notes: str | None = None

@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None

class CreateOrderUseCase:
    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService,
    ):
        self._order_service = order_service
        self._inventory_service = inventory_service

    @transaction.atomic
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        is_valid, error = self._inventory_service.validate_availability(input_dto.items)
        if not is_valid:
            return CreateOrderOutput(success=False, error=error)

        order = self._order_service.create(
            customer_id=input_dto.customer_id,
            items=input_dto.items,
        )
        return CreateOrderOutput(success=True, order_id=order.id)
```

**Service**
```python
class OrderService:
    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        return Order.objects.select_related('customer').filter(id=order_id).first()

    def create(self, customer_id: UUID, items: list[OrderItemInput]) -> Order:
        order = Order.objects.create(customer_id=customer_id)
        for item in items:
            OrderItem.objects.create(order=order, **item.__dict__)
        return order

    def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
        if order.status == Order.Status.SHIPPED:
            return False, "Cannot cancel shipped orders"
        return True, ""
```

**Thin view**
```python
class CreateOrderView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request: Request) -> Response:
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase(
            order_service=OrderService(),
            inventory_service=InventoryService(),
        )
        input_dto = CreateOrderInput(**serializer.validated_data)
        output = use_case.execute(input_dto)

        if not output.success:
            return Response({'error': output.error}, status=400)
        return Response({'order_id': str(output.order_id)}, status=201)
```

**Serializer separation**
```python
class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True, min_length=1)
    notes = serializers.CharField(max_length=500, required=False)

class OrderReadSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source='customer.name', read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'customer_name', 'status', 'created_at']
        read_only_fields = fields
```

### Checklists

**Architecture**
- [ ] Views thin (< ~20 lines of logic)
- [ ] Use cases = one operation each
- [ ] Services hold reusable domain logic
- [ ] Dependencies flow inward only
- [ ] No circular imports between layers

**Quality**
- [ ] Type hints on public interfaces
- [ ] DTOs frozen + slots
- [ ] Business errors via Output, not exceptions
- [ ] Validation returns `tuple[bool, str]`
- [ ] `@transaction.atomic` around state changes

**Performance**
- [ ] `select_related` for FK access
- [ ] `prefetch_related` for reverse/M2M
- [ ] Indexes on hot filters
- [ ] No N+1 in serializers

## Directory structure

```
apps/<app_name>/
├── models.py
├── views.py
├── serializers.py
├── urls.py
├── admin.py
├── services/
│   ├── __init__.py
│   └── <entity>_service.py
├── use_cases/
│   ├── __init__.py
│   └── <action>_<entity>.py
├── tests/
│   ├── test_models.py
│   ├── test_services.py
│   ├── test_use_cases.py
│   ├── test_views.py
│   └── factories.py
└── migrations/
```

## Bundled resources

Load only what the task needs.

### Architecture (Build target + Refactor destination)

| File | Use for |
|---|---|
| `references/project-structure.md` | Layout, naming, dependency rules |
| `references/use-cases-pattern.md` | DTOs, injection, transactions, errors |
| `references/services-pattern.md` | Static vs instance, validation, cross-service |
| `references/models-domain.md` | Domain on models, TextChoices, UUID, indexes |
| `references/views-serializers.md` | Thin views, Read vs Write serializers |
| `references/testing-clean-arch.md` | Unit/integration testing |
| `references/examples.md` | Full CRUD and workflows |

### API

| File | Use for |
|---|---|
| `references/query-optimization.md` | N+1, bulk, aggregations |
| `references/api-patterns.md` | Pagination, filter, versioning, throttle |
| `references/authentication.md` | JWT, permissions, API keys |
| `references/production-api.md` | Settings, cache, logging, Docker, health |
| `references/django-admin.md` | ModelAdmin, inlines, admin security |

### Quality

| File | Use for |
|---|---|
| `references/coding-standards.md` | Ruff, imports, security, logging |

### Refactor pack

| File | Use for |
|---|---|
| `references/refactor/clean-architecture-transforms.md` | Fat view → use case, exception → Output, etc. |
| `references/refactor/django-anti-patterns.md` | Fat views, N+1, null misuse, bare excepts |
| `references/refactor/django-5-patterns.md` | GeneratedField, db_default, async views |
| `references/refactor/python-312-features.md` | Type params, `@override`, modern typing |

## Core principles (short)

**No exceptions for business logic** — return `Output(success=False, error=...)`.  
**Validation returns** `tuple[bool, str]`.  
**Constructor injection** on use cases for testability.  
**DRY / SRP / SoC / early returns / small functions**.

## Versions and naming

- Python 3.10+ required (`X | None`); 3.12+ preferred  
- Django 4.2 LTS min; 5.0+ preferred  
- Use cases: `<action>_<entity>.py`  
- Services: `<entity>_service.py`  
- Serializers: `<Entity>ReadSerializer`, `<Entity>CreateSerializer`
