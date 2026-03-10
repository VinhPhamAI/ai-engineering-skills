# Anti-Patterns Reference

## Anemic Domain Model
The most common DDD anti-pattern: domain objects are just data containers with no behavior. All logic lives in services.

```typescript
// ❌ Anemic — Order is just a bag of data
class Order {
  id: string;
  status: string;
  items: OrderItem[];
  total: number;
}

class OrderService {
  placeOrder(order: Order): void {
    if (order.items.length === 0) throw new Error('No items');
    order.status = 'PLACED';         // logic leaks into service
    order.total = this.calculateTotal(order.items);
  }
}

// ✅ Rich domain model — Order encapsulates its own rules
class Order {
  private status: OrderStatus = OrderStatus.DRAFT;

  place(): void {
    if (this.items.size === 0) throw new DomainError('No items');
    this.status = OrderStatus.PLACED;
  }

  get total(): Money {
    return [...this.items.values()].reduce((sum, item) => sum.add(item.subtotal), Money.zero());
  }
}
```

---

## God Class / God Service
A single class that does everything — violates Single Responsibility and becomes unmaintainable.

```typescript
// ❌ God Service
class UserService {
  register(data: RegisterDto) { /* auth logic */ }
  sendWelcomeEmail(user: User) { /* email logic */ }
  generateReport(userId: string) { /* reporting logic */ }
  calculateSubscriptionFee(userId: string) { /* billing logic */ }
  updateProfile(userId: string, data: ProfileDto) { /* profile logic */ }
}

// ✅ Decomposed
class UserRegistrationService { register(data: RegisterDto) {} }
class UserNotificationService { sendWelcomeEmail(user: User) {} }
class UserReportingService    { generateReport(userId: string) {} }
class SubscriptionCalculator  { calculateFee(userId: string) {} }
class UserProfileService      { updateProfile(userId: string, data: ProfileDto) {} }
```

---

## NIH (Not Invented Here) Syndrome
Writing custom code for solved problems instead of using established libraries.

| Don't Build | Use Instead |
|---|---|
| Custom retry logic | `cockatiel` (TS) / `tenacity` (Python) |
| Custom form validation | `zod` (TS) / `pydantic` (Python) |
| Custom auth (JWT, OAuth) | `Auth0`, `Supabase`, `Clerk`, `django-allauth` |
| Custom date manipulation | `date-fns` / `arrow`, `pendulum` |
| Custom HTTP client with retries | `axios` with interceptors / `httpx` |
| Custom ORM | `Prisma`, `TypeORM` / `SQLAlchemy`, Django ORM |
| Custom state management | `Zustand`, `Redux` |
| Custom test factories | `faker-js` / `factory_boy`, `polyfactory` |

Every line of custom code needs maintenance, documentation, and tests. Use libraries.

---

## Leaky Abstractions
Infrastructure concerns bleeding into the domain or application layer.

```typescript
// ❌ Domain entity with Prisma decorator — tied to ORM
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  status: string;
}

// ✅ Pure domain object — persistence handled by mapper in infrastructure
export class Order {
  constructor(
    public readonly id: OrderId,
    private status: OrderStatus,
  ) {}
}

// Infrastructure: OrderMapper converts between Order ↔ OrderModel
```

```python
# ❌ Domain class with SQLAlchemy Base — tied to ORM
from sqlalchemy.orm import DeclarativeBase, mapped_column

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[str] = mapped_column(primary_key=True)
    status: Mapped[str]

# ✅ Pure domain object
@dataclass
class Order:
    id: OrderId
    status: OrderStatus
    # No SQLAlchemy dependency here
```

---

## Transaction Script (instead of Domain Model)
Writing procedural code in service methods for complex business logic instead of rich domain objects.

```typescript
// ❌ Transaction Script — 200-line service method
async function processOrderPayment(orderId: string, paymentData: PaymentData): Promise<void> {
  const orderRow = await db.query('SELECT * FROM orders WHERE id = $1', [orderId]);
  if (!orderRow) throw new Error('Not found');
  if (orderRow.status !== 'PLACED') throw new Error('Invalid status');
  const paymentResult = await paymentGateway.charge(paymentData);
  if (!paymentResult.success) {
    await db.query('UPDATE orders SET status = $1 WHERE id = $2', ['PAYMENT_FAILED', orderId]);
    return;
  }
  await db.query('UPDATE orders SET status = $1, payment_id = $2 WHERE id = $3',
    ['PAID', paymentResult.id, orderId]);
  await emailService.sendReceipt(orderRow.customer_email);
  // ... 150 more lines
}

// ✅ Use case + domain model
async function processPayment(command: ProcessPaymentCommand): Promise<void> {
  const order = await orderRepo.findById(command.orderId);
  order.processPayment(command.paymentData); // domain logic on entity
  await orderRepo.save(order);
  await eventBus.publish(new PaymentProcessedEvent(order.id));
  // notification handled by event subscriber
}
```

---

## Primitive Obsession
Using primitive types instead of Value Objects for domain concepts.

```typescript
// ❌ Primitives everywhere — easy to mix up arguments
function transferMoney(fromAccountId: string, toAccountId: string, amount: number, currency: string) {}

// Called incorrectly — compiler can't catch this:
transferMoney(amount, currency, fromId, toId); // wrong order!

// ✅ Value Objects — self-documenting and validated
function transferMoney(from: AccountId, to: AccountId, money: Money) {}
```

---

## Shotgun Surgery
A single conceptual change requires edits in many unrelated files — sign of poor cohesion.

**Cause**: Logic for one concept is scattered across many layers and files.
**Fix**: Follow the DDD principle — put logic where it belongs in the domain model.

---

## Over-Engineering Checklist

Before adding complexity, ask:
- **Do I have this problem today**, or am I solving a hypothetical?
- **Will the team understand** this abstraction in 6 months?
- **Does this bounded context** truly need CQRS + Event Sourcing, or is a simple service enough?
- **Is this microservice** genuinely independent, or is it just a distributed monolith with network calls?

> "A complex system that works is invariably found to have evolved from a simple system that worked." — Gall's Law