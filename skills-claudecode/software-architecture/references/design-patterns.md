# Design Patterns Reference

## CQRS (Command Query Responsibility Segregation)

Separate **write** operations (commands, mutate state) from **read** operations (queries, return data).

**Use CQRS when:**
- Read and write loads are significantly asymmetric
- Read models need different shapes than write models
- Working with event sourcing
- System needs independent scaling of reads vs writes

**Don't use CQRS when:**
- Simple CRUD — it adds unnecessary complexity
- Small teams without the operational overhead tolerance

```typescript
// TypeScript — Command side
export class PlaceOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly productId: string,
    public readonly quantity: number,
  ) {}
}

export class PlaceOrderHandler {
  constructor(private readonly orderRepo: OrderRepository) {}

  async handle(command: PlaceOrderCommand): Promise<void> {
    const order = Order.create(command.customerId);
    // ... domain logic
    await this.orderRepo.save(order);
  }
}

// Query side — returns a flat DTO, no domain model needed
export class GetOrderSummaryQuery {
  constructor(public readonly orderId: string) {}
}

export class GetOrderSummaryHandler {
  constructor(private readonly db: ReadDatabase) {}

  async handle(query: GetOrderSummaryQuery): Promise<OrderSummaryDto> {
    // Direct DB query, optimized for reads
    return this.db.query(
      `SELECT o.id, c.name, o.total FROM orders o JOIN customers c ON o.customer_id = c.id WHERE o.id = $1`,
      [query.orderId],
    );
  }
}
```

```python
# Python — CQRS with dataclasses
@dataclass
class PlaceOrderCommand:
    customer_id: str
    product_id: str
    quantity: int

class PlaceOrderHandler:
    def __init__(self, order_repo: OrderRepository) -> None:
        self.order_repo = order_repo

    async def handle(self, command: PlaceOrderCommand) -> None:
        order = Order.create(command.customer_id)
        # ... domain logic
        await self.order_repo.save(order)

@dataclass
class GetOrderSummaryQuery:
    order_id: str

class GetOrderSummaryHandler:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def handle(self, query: GetOrderSummaryQuery) -> OrderSummaryDto:
        result = await self.session.execute(
            text("SELECT o.id, c.name, o.total FROM orders o JOIN customers c ON o.customer_id = c.id WHERE o.id = :id"),
            {"id": query.order_id},
        )
        return OrderSummaryDto(**result.mappings().one())
```

---

## Event-Driven Architecture

Events represent **facts that happened** — they're immutable, past-tense, and broadcast to interested parties.

**Use event-driven when:**
- Decoupling between bounded contexts is needed
- Actions in one context should trigger reactions in another
- Audit trail / event log is a business requirement
- High scalability or async processing is needed

### Publish-Subscribe Pattern

```typescript
// TypeScript
// Publisher (in application layer after saving aggregate)
await this.eventBus.publish(new OrderPlacedEvent(order.id, order.customerId, order.total));

// Subscriber (in another bounded context or service)
export class NotificationService {
  @EventHandler(OrderPlacedEvent)
  async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    await this.emailSender.sendOrderConfirmation(event.customerId, event.orderId);
  }
}
```

```python
# Python
# Publisher
await event_bus.publish(OrderPlacedEvent(order_id=order.id, customer_id=order.customer_id))

# Subscriber
class NotificationService:
    @event_handler(OrderPlacedEvent)
    async def on_order_placed(self, event: OrderPlacedEvent) -> None:
        await self.email_sender.send_order_confirmation(event.customer_id, event.order_id)
```

### Outbox Pattern (Reliable Event Publishing)
Never publish events directly in the same DB transaction as saving the aggregate — events can be lost if the broker is down. Use the Outbox pattern:

1. Save the aggregate AND the event to the outbox table **in the same DB transaction**
2. A background worker reads the outbox and publishes to the broker
3. Mark events as published after broker confirms

---

## Saga Pattern (Distributed Transactions)

Coordinate multi-step processes across bounded contexts without distributed transactions.

**Choreography-based** (events only, no central coordinator):
- Each service listens for events and reacts
- Simple, but hard to visualize the full flow

**Orchestration-based** (central saga orchestrator):
- One service drives the saga by calling others
- Easier to trace and manage compensating actions

```
OrderSaga:
  1. PlaceOrder → emit OrderPlaced
  2. Listen: PaymentProcessed → emit InventoryReserved
  3. Listen: InventoryReserved → emit OrderConfirmed
  4. On failure at any step → emit compensating events (CancelOrder, RefundPayment)
```

---

## Repository Pattern

See `ddd-patterns.md` for full interface examples. Key points:

- The repository **simulates an in-memory collection** of domain objects
- Methods: `findById`, `findBy<Criteria>`, `save`, `delete`
- **No** `update` method — `save` handles both insert and update (upsert)
- Return domain objects, not DB models
- Use **Specification pattern** for complex queries instead of proliferating `findBy*` methods

---

## Factory Pattern

Used when creating a domain object requires complex logic, validation, or coordination:

```typescript
// TypeScript
export class OrderFactory {
  static createDraftOrder(customerId: CustomerId, items: CreateOrderItemDto[]): Order {
    if (items.length === 0) throw new DomainError('Cannot create empty order');
    const order = new Order(OrderId.generate(), customerId);
    items.forEach(item => order.addItem(item.productId, item.quantity));
    return order;
  }
}
```

```python
# Python
class OrderFactory:
    @staticmethod
    def create_draft_order(customer_id: CustomerId, items: list[CreateOrderItemDto]) -> Order:
        if not items:
            raise DomainError("Cannot create empty order")
        order = Order(id=OrderId.generate(), customer_id=customer_id)
        for item in items:
            order.add_item(item.product_id, item.quantity)
        return order
```

---

## Specification Pattern

Encapsulate complex business rules as reusable, composable objects:

```typescript
// TypeScript
export interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
}

export class PremiumCustomerSpec implements Specification<Customer> {
  isSatisfiedBy(customer: Customer): boolean {
    return customer.totalPurchases.greaterThan(Money.of(10000, 'USD'));
  }
}

export class ActiveCustomerSpec implements Specification<Customer> {
  isSatisfiedBy(customer: Customer): boolean {
    return customer.lastPurchaseDate > subDays(new Date(), 90);
  }
}

// Usage
const eligibleForDiscount = new PremiumCustomerSpec().and(new ActiveCustomerSpec());
const customers = allCustomers.filter(c => eligibleForDiscount.isSatisfiedBy(c));
```