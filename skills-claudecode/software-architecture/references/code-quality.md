# Code Quality & Testing Reference

## SOLID Principles

| Principle | Rule | Common Violation |
|---|---|---|
| **Single Responsibility** | One reason to change | `UserService` handling auth, profile, notifications |
| **Open/Closed** | Open for extension, closed for modification | Adding `if type == "X"` chains instead of polymorphism |
| **Liskov Substitution** | Subtypes must behave like their base type | Override that throws when parent didn't |
| **Interface Segregation** | Prefer small, focused interfaces | One giant `IService` with 20 methods |
| **Dependency Inversion** | Depend on abstractions, not concretions | Instantiating `new EmailSender()` inside a use case |

---

## Error Handling

### Typed Errors (not strings)
```typescript
// TypeScript — domain errors
export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DomainError';
  }
}

export class OrderNotFoundError extends DomainError {
  constructor(orderId: string) {
    super(`Order ${orderId} not found`);
    this.name = 'OrderNotFoundError';
  }
}

// Use case — typed catch
try {
  await placeOrderUseCase.execute(command);
} catch (error) {
  if (error instanceof OrderNotFoundError) {
    return res.status(404).json({ error: error.message });
  }
  if (error instanceof DomainError) {
    return res.status(400).json({ error: error.message });
  }
  throw error; // re-throw unexpected errors
}
```

```python
# Python — domain errors
class DomainError(Exception):
    pass

class OrderNotFoundError(DomainError):
    def __init__(self, order_id: str) -> None:
        super().__init__(f"Order {order_id} not found")

# Use case — typed except
try:
    await place_order_use_case.execute(command)
except OrderNotFoundError as e:
    raise HTTPException(status_code=404, detail=str(e))
except DomainError as e:
    raise HTTPException(status_code=400, detail=str(e))
```

### Result Type (TypeScript)
For operations where failure is a valid domain outcome (not exceptional), use a Result type:

```typescript
type Result<T, E = DomainError> = { ok: true; value: T } | { ok: false; error: E };

export class ApplyCouponUseCase {
  async execute(command: ApplyCouponCommand): Promise<Result<Order>> {
    const coupon = await this.couponRepo.findByCode(command.code);
    if (!coupon) return { ok: false, error: new DomainError('Invalid coupon') };
    if (coupon.isExpired()) return { ok: false, error: new DomainError('Coupon expired') };
    // ...
    return { ok: true, value: order };
  }
}
```

---

## Testing Strategy — The Testing Pyramid

```
        ┌──────────┐
        │    E2E   │  ← Few, slow, test full user journeys (Playwright, Cypress)
        │  (5–10%) │
       ┌┴──────────┴┐
       │Integration │  ← Test layer boundaries: repos, APIs, message queues
       │  (20–30%) │
      ┌┴────────────┴┐
      │     Unit     │  ← Many, fast, test domain logic in isolation
      │   (60–70%)   │
      └──────────────┘
```

### Unit Tests — Domain Layer (highest ROI)
Test domain objects with **no mocks** (pure functions, no I/O):

```typescript
// TypeScript — Jest
describe('Order', () => {
  it('should add items to a draft order', () => {
    const order = Order.create(customerId);
    order.addItem(product, 2);
    expect(order.items).toHaveLength(1);
    expect(order.total).toEqual(Money.of(product.price.amount * 2, 'USD'));
  });

  it('should throw when adding item to a placed order', () => {
    const order = Order.create(customerId);
    order.place();
    expect(() => order.addItem(product, 1)).toThrow(DomainError);
  });
});
```

```python
# Python — pytest
def test_add_items_to_draft_order():
    order = Order.create(customer_id=CustomerId("c1"))
    order.add_item(product_id=ProductId("p1"), price=Money(Decimal("10"), "USD"), quantity=2)
    assert len(order.items) == 1
    assert order.total == Money(Decimal("20"), "USD")

def test_cannot_add_item_to_placed_order():
    order = Order.create(customer_id=CustomerId("c1"))
    order.place()
    with pytest.raises(DomainError):
        order.add_item(product_id=ProductId("p1"), price=Money(Decimal("10"), "USD"), quantity=1)
```

### Unit Tests — Application Layer (use cases with mocked dependencies)

```typescript
// TypeScript — Jest with mocks
describe('PlaceOrderUseCase', () => {
  let useCase: PlaceOrderUseCase;
  let orderRepo: jest.Mocked<OrderRepository>;
  let eventBus: jest.Mocked<EventBus>;

  beforeEach(() => {
    orderRepo = { findById: jest.fn(), save: jest.fn() } as any;
    eventBus = { publish: jest.fn() } as any;
    useCase = new PlaceOrderUseCase(orderRepo, eventBus);
  });

  it('should save order and publish event', async () => {
    await useCase.execute(new PlaceOrderCommand('customer-1', 'product-1', 2));
    expect(orderRepo.save).toHaveBeenCalledTimes(1);
    expect(eventBus.publish).toHaveBeenCalledWith(expect.any(OrderPlacedEvent));
  });
});
```

```python
# Python — pytest with unittest.mock
@pytest.mark.asyncio
async def test_place_order_saves_and_publishes_event():
    order_repo = AsyncMock(spec=OrderRepository)
    event_bus = AsyncMock(spec=EventBus)
    use_case = PlaceOrderUseCase(order_repo=order_repo, event_bus=event_bus)

    await use_case.execute(PlaceOrderCommand(customer_id="c1", product_id="p1", quantity=2))

    order_repo.save.assert_awaited_once()
    event_bus.publish.assert_awaited_once()
    published_event = event_bus.publish.call_args[0][0]
    assert isinstance(published_event, OrderPlacedEvent)
```

### Integration Tests — Infrastructure Layer
Test real DB interactions against a test database (use Docker / testcontainers):

```typescript
// TypeScript — Jest with test DB
describe('PrismaOrderRepository', () => {
  let prisma: PrismaClient;
  let repo: PrismaOrderRepository;

  beforeAll(async () => {
    prisma = new PrismaClient({ datasources: { db: { url: process.env.TEST_DATABASE_URL } } });
    repo = new PrismaOrderRepository(prisma);
  });

  afterAll(() => prisma.$disconnect());

  it('should persist and retrieve an order', async () => {
    const order = Order.create(CustomerId.of('customer-1'));
    await repo.save(order);
    const found = await repo.findById(order.id);
    expect(found?.id.equals(order.id)).toBe(true);
  });
});
```

```python
# Python — pytest with real test DB
@pytest.mark.asyncio
@pytest.mark.integration
async def test_persist_and_retrieve_order(async_session: AsyncSession):
    repo = SqlAlchemyOrderRepository(async_session)
    order = Order.create(customer_id=CustomerId("c1"))
    
    await repo.save(order)
    found = await repo.find_by_id(order.id)
    
    assert found is not None
    assert found.id == order.id
```

### Integration Tests — HTTP Layer (API tests)
```python
# Python — FastAPI TestClient
def test_create_order_returns_201(client: TestClient, db_session):
    response = client.post("/orders", json={"customer_id": "c1", "product_id": "p1", "quantity": 2})
    assert response.status_code == 201
    assert "order_id" in response.json()
```

---

## What to Mock vs Not Mock

| Layer | Mock? | Rationale |
|---|---|---|
| Domain entities | ❌ Never | Use real objects — they're pure logic |
| Domain services | ❌ Rarely | Test with real objects when possible |
| Repositories (in use case tests) | ✅ Yes | Isolate use case from DB |
| Event bus (in use case tests) | ✅ Yes | Verify publishing without broker |
| External APIs (in infra tests) | ✅ Yes | Use stubs/VCR cassettes |
| Database (in integration tests) | ❌ No | Test against real DB in Docker |

---

## Code Quality Rules Summary

### DRY, KISS, YAGNI Applied
- **DRY**: If you wrote the same logic twice → extract a function. Third time → extract a module.
- **KISS**: Choose the simplest solution that satisfies the requirement. Complexity is debt.
- **YAGNI**: Don't add abstractions for hypothetical future needs. Add them when the need is real.

### Naming Conventions Quick Reference
```
# TypeScript
const calculateOrderTotal = () => {}           // ✅ verb + noun
const orderTotalCalculator = {}                // ✅ noun (class/object)
const calc = () => {}                          // ❌ abbreviation
const processData = () => {}                   // ❌ generic verb

# Python
def calculate_order_total() -> Money: ...      # ✅
def process_data(): ...                        # ❌ generic
class OrderTotalCalculator: ...               # ✅
class Utils: ...                              # ❌ generic
```

### Guard Clauses (Early Return)
```typescript
// ❌ Nested — hard to read
function processOrder(order: Order) {
  if (order !== null) {
    if (order.status === 'DRAFT') {
      if (order.items.length > 0) {
        // actual logic buried here
      }
    }
  }
}

// ✅ Early return — flat and clear
function processOrder(order: Order) {
  if (!order) return;
  if (order.status !== 'DRAFT') throw new DomainError('Only draft orders can be processed');
  if (order.items.length === 0) throw new DomainError('Order has no items');
  // actual logic at top level
}
```