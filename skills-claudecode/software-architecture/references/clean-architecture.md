# Clean Architecture Reference

## The Dependency Rule

Dependencies flow **inward only**. Inner layers know nothing about outer layers.

```
┌─────────────────────────────────────────┐
│  Infrastructure / Frameworks / Drivers  │  ← outermost: DB, HTTP, UI, 3rd party
│  ┌───────────────────────────────────┐  │
│  │    Interface Adapters / Ports     │  │  ← controllers, presenters, gateways
│  │  ┌─────────────────────────────┐ │  │
│  │  │     Application / Use Cases  │ │  │  ← orchestrates domain logic
│  │  │  ┌───────────────────────┐  │ │  │
│  │  │  │    Domain / Entities  │  │ │  │  ← innermost: pure business rules
│  │  │  └───────────────────────┘  │ │  │
│  │  └─────────────────────────────┘ │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**The domain layer must have zero external dependencies** — no ORM imports, no HTTP clients, no framework decorators in domain classes.

---

## Layer Responsibilities

### Domain Layer (innermost)
- Entities with identity and business rules
- Value Objects (immutable, no identity)
- Domain Services (logic spanning multiple aggregates)
- Domain Events
- Repository **interfaces** (not implementations)
- **Zero** framework or infrastructure imports

```typescript
// ✅ Domain Entity — TypeScript
export class Order {
  private items: OrderItem[] = [];

  addItem(product: Product, quantity: number): void {
    if (quantity <= 0) throw new DomainError('Quantity must be positive');
    this.items.push(new OrderItem(product, quantity));
  }

  get total(): Money {
    return this.items.reduce((sum, item) => sum.add(item.subtotal), Money.zero());
  }
}
```

```python
# ✅ Domain Entity — Python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Order:
    id: OrderId
    items: List[OrderItem] = field(default_factory=list)

    def add_item(self, product: Product, quantity: int) -> None:
        if quantity <= 0:
            raise DomainError("Quantity must be positive")
        self.items.append(OrderItem(product, quantity))

    @property
    def total(self) -> Money:
        return sum((item.subtotal for item in self.items), Money.zero())
```

---

### Application Layer (Use Cases)
- One class/function per use case (single responsibility)
- Orchestrates domain objects — contains **no business logic itself**
- Fetches entities via repository interfaces
- Emits domain events
- Returns DTOs, never domain entities

```typescript
// ✅ Use Case — TypeScript
export class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly productRepo: ProductRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<OrderId> {
    const product = await this.productRepo.findById(command.productId);
    if (!product) throw new NotFoundError('Product not found');

    const order = Order.create(command.customerId);
    order.addItem(product, command.quantity);

    await this.orderRepo.save(order);
    await this.eventBus.publish(new OrderPlacedEvent(order.id));

    return order.id;
  }
}
```

```python
# ✅ Use Case — Python
class PlaceOrderUseCase:
    def __init__(
        self,
        order_repo: OrderRepository,
        product_repo: ProductRepository,
        event_bus: EventBus,
    ) -> None:
        self.order_repo = order_repo
        self.product_repo = product_repo
        self.event_bus = event_bus

    async def execute(self, command: PlaceOrderCommand) -> OrderId:
        product = await self.product_repo.find_by_id(command.product_id)
        if product is None:
            raise NotFoundError("Product not found")

        order = Order.create(command.customer_id)
        order.add_item(product, command.quantity)

        await self.order_repo.save(order)
        await self.event_bus.publish(OrderPlacedEvent(order.id))

        return order.id
```

---

### Interface Adapters Layer
- **Controllers**: parse HTTP request → call use case → format response
- **Presenters**: transform domain/use case output to view model
- **Gateways**: implement repository interfaces defined in domain

Controllers must be thin — no business logic, no direct DB access:

```typescript
// ✅ Thin Controller — TypeScript (Express)
export class OrderController {
  constructor(private readonly placeOrder: PlaceOrderUseCase) {}

  async create(req: Request, res: Response): Promise<void> {
    const command = PlaceOrderCommand.fromRequest(req.body);
    const orderId = await this.placeOrder.execute(command);
    res.status(201).json({ orderId });
  }
}
```

```python
# ✅ Thin Controller — Python (FastAPI)
@router.post("/orders", status_code=201)
async def create_order(
    body: PlaceOrderRequest,
    use_case: PlaceOrderUseCase = Depends(get_place_order_use_case),
) -> OrderResponse:
    command = PlaceOrderCommand.from_request(body)
    order_id = await use_case.execute(command)
    return OrderResponse(order_id=order_id)
```

---

### Infrastructure Layer (outermost)
- Concrete repository implementations (SQLAlchemy, Prisma, etc.)
- External API clients
- Message queue producers/consumers
- Configuration and DI wiring

```typescript
// ✅ Repository Implementation — TypeScript (Prisma)
export class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({ where: { id: id.value } });
    return row ? OrderMapper.toDomain(row) : null;
  }

  async save(order: Order): Promise<void> {
    const data = OrderMapper.toPersistence(order);
    await this.prisma.order.upsert({ where: { id: data.id }, create: data, update: data });
  }
}
```

```python
# ✅ Repository Implementation — Python (SQLAlchemy)
class SqlAlchemyOrderRepository(OrderRepository):
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def find_by_id(self, id: OrderId) -> Order | None:
        row = await self.session.get(OrderModel, id.value)
        return OrderMapper.to_domain(row) if row else None

    async def save(self, order: Order) -> None:
        data = OrderMapper.to_persistence(order)
        await self.session.merge(data)
```

---

## Ports & Adapters (Hexagonal Architecture)

Ports are **interfaces defined by the application**; adapters are their implementations.

- **Inbound ports** (Driving): `OrderService` interface called by HTTP controller, CLI, or test
- **Outbound ports** (Driven): `OrderRepository`, `EmailSender`, `PaymentGateway` — implemented in infrastructure

This is preferred when the app has **many external integrations** that must be swappable.

---

## Project Structure

### TypeScript
```
src/
├── domain/
│   ├── order/
│   │   ├── Order.ts
│   │   ├── OrderItem.ts
│   │   ├── OrderRepository.ts      ← interface
│   │   └── OrderPlacedEvent.ts
│   └── shared/
│       ├── Money.ts                ← value object
│       └── DomainError.ts
├── application/
│   └── order/
│       ├── PlaceOrderUseCase.ts
│       └── PlaceOrderCommand.ts
├── infrastructure/
│   ├── persistence/
│   │   └── PrismaOrderRepository.ts
│   └── messaging/
│       └── RabbitMQEventBus.ts
└── interface/
    └── http/
        └── OrderController.ts
```

### Python
```
src/
├── domain/
│   ├── order/
│   │   ├── order.py
│   │   ├── order_item.py
│   │   ├── order_repository.py    # abstract class / Protocol
│   │   └── events.py
│   └── shared/
│       ├── money.py
│       └── errors.py
├── application/
│   └── order/
│       ├── place_order.py
│       └── commands.py
├── infrastructure/
│   ├── persistence/
│   │   └── sqlalchemy_order_repo.py
│   └── messaging/
│       └── rabbitmq_event_bus.py
└── interface/
    └── http/
        └── order_router.py
```