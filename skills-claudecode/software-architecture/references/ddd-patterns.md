# Domain-Driven Design (DDD) Patterns Reference

## Core Building Blocks

### Entities
Objects with a **unique identity** that persists over time. Two entities are equal if their IDs match, regardless of attribute values.

```typescript
// TypeScript
export class Customer {
  constructor(
    public readonly id: CustomerId,
    private name: CustomerName,
    private email: Email,
  ) {}

  changeName(name: CustomerName): void {
    this.name = name;
  }

  equals(other: Customer): boolean {
    return this.id.equals(other.id);
  }
}
```

```python
# Python
@dataclass
class Customer:
    id: CustomerId
    name: CustomerName
    email: Email

    def change_name(self, name: CustomerName) -> None:
        self.name = name

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Customer):
            return False
        return self.id == other.id
```

---

### Value Objects
**Immutable** objects with no identity — equal if all attributes are equal. Encapsulate validation and domain rules.

```typescript
// TypeScript
export class Money {
  private constructor(
    private readonly amount: number,
    private readonly currency: string,
  ) {
    if (amount < 0) throw new DomainError('Money cannot be negative');
  }

  static of(amount: number, currency: string): Money {
    return new Money(amount, currency);
  }

  static zero(): Money {
    return new Money(0, 'USD');
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new DomainError('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

```python
# Python
from dataclasses import dataclass

@dataclass(frozen=True)   # frozen = immutable
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise DomainError("Money cannot be negative")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise DomainError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)
```

**When to use Value Objects**: postal addresses, phone numbers, money, coordinates, email addresses, date ranges, quantities with units.

---

### Aggregates
A **cluster of entities and value objects** with a single root entity (Aggregate Root). All changes go through the root; no direct access to internal entities from outside.

**Rules:**
1. Reference other aggregates **by ID only** — never hold object references across aggregate boundaries
2. Enforce invariants within the aggregate boundary
3. Only the Aggregate Root has a repository
4. Keep aggregates **small** — if they grow large, consider splitting

```typescript
// TypeScript — Order is the Aggregate Root
export class Order {
  private readonly items: Map<ProductId, OrderItem> = new Map();
  private status: OrderStatus = OrderStatus.DRAFT;

  addItem(product: Product, quantity: number): void {
    if (this.status !== OrderStatus.DRAFT) throw new DomainError('Cannot modify placed order');
    const existing = this.items.get(product.id);
    if (existing) {
      this.items.set(product.id, existing.increaseQuantity(quantity));
    } else {
      this.items.set(product.id, OrderItem.create(product.id, product.price, quantity));
    }
  }

  place(): void {
    if (this.items.size === 0) throw new DomainError('Order has no items');
    this.status = OrderStatus.PLACED;
    this.addDomainEvent(new OrderPlacedEvent(this.id));
  }
}
```

```python
# Python
@dataclass
class Order:
    id: OrderId
    customer_id: CustomerId        # reference by ID, not Customer object
    _items: dict[ProductId, OrderItem] = field(default_factory=dict)
    _status: OrderStatus = OrderStatus.DRAFT
    _domain_events: list[DomainEvent] = field(default_factory=list)

    def add_item(self, product_id: ProductId, price: Money, quantity: int) -> None:
        if self._status != OrderStatus.DRAFT:
            raise DomainError("Cannot modify placed order")
        if product_id in self._items:
            self._items[product_id] = self._items[product_id].increase_quantity(quantity)
        else:
            self._items[product_id] = OrderItem(product_id, price, quantity)

    def place(self) -> None:
        if not self._items:
            raise DomainError("Order has no items")
        self._status = OrderStatus.PLACED
        self._domain_events.append(OrderPlacedEvent(self.id))
```

---

### Domain Services
Used when logic **spans multiple aggregates** or doesn't naturally belong to any single entity.

```typescript
// TypeScript
export class TransferService {
  transfer(from: Account, to: Account, amount: Money): void {
    from.debit(amount);
    to.credit(amount);
    // Logic belongs here, not on Account, because it involves two aggregates
  }
}
```

```python
# Python
class TransferService:
    def transfer(self, from_account: Account, to_account: Account, amount: Money) -> None:
        from_account.debit(amount)
        to_account.credit(amount)
```

---

### Domain Events
Represent something that **happened** in the domain. Past tense. Published after state changes. Consumed by other parts of the system.

```typescript
// TypeScript
export class OrderPlacedEvent {
  readonly occurredAt = new Date();
  constructor(
    public readonly orderId: OrderId,
    public readonly customerId: CustomerId,
    public readonly total: Money,
  ) {}
}
```

```python
# Python
@dataclass(frozen=True)
class OrderPlacedEvent:
    order_id: OrderId
    customer_id: CustomerId
    total: Money
    occurred_at: datetime = field(default_factory=datetime.utcnow)
```

---

### Repositories
Abstract the persistence of aggregates. Define the **interface in the domain layer**, implement in infrastructure.

```typescript
// TypeScript — interface in domain
export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomer(customerId: CustomerId): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}
```

```python
# Python — abstract class or Protocol in domain
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    @abstractmethod
    async def find_by_id(self, id: OrderId) -> Order | None: ...
    @abstractmethod
    async def find_by_customer(self, customer_id: CustomerId) -> list[Order]: ...
    @abstractmethod
    async def save(self, order: Order) -> None: ...
```

---

## Strategic DDD

### Bounded Contexts
A **bounded context** is an explicit boundary within which a particular domain model applies. The same word can mean different things in different contexts.

```
┌─────────────────────┐    ┌─────────────────────┐
│   Sales Context     │    │  Shipping Context   │
│                     │    │                     │
│  Customer:          │    │  Customer:          │
│   - name            │    │   - delivery address│
│   - credit limit    │    │   - contact phone   │
│   - purchase history│    │                     │
│  Order: cart+items  │    │  Order: parcel+route│
└─────────────────────┘    └─────────────────────┘
         │                           │
         └──────────── Integration ──┘
              (via events or ACL)
```

**Rules:**
- Each bounded context has its own model, its own database (ideally), its own team
- Cross-context communication: domain events, REST APIs, or message queues
- Use an **Anti-Corruption Layer (ACL)** to translate between contexts — never let a foreign model leak in

### Ubiquitous Language
- Code must use the **same terms** as domain experts use in conversation
- If the business says "Invoice", the class is `Invoice` — not `BillingDocument`
- No divergence between what devs say and what the code reads

### Context Map (Integration Patterns)
| Pattern | Use When |
|---|---|
| **Shared Kernel** | Two teams share a small, stable sub-model |
| **Customer-Supplier** | One context depends on another; upstream owns the contract |
| **Anti-Corruption Layer** | Integrating with legacy or external systems |
| **Open Host Service** | Publishing a protocol for many consumers (REST API) |
| **Published Language** | Shared event schema (e.g., Avro, JSON Schema) |