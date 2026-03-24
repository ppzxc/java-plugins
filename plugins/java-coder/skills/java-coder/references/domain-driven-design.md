# Domain-Driven Design — Patterns and Modular Monolith Mapping

Based on *Domain-Driven Design* by Eric Evans and *Implementing Domain-Driven Design* by Vaughn Vernon.
Adapted for Modular Monolith architecture with Java 25 + Spring Boot 4.

## Table of Contents
1. Strategic Design
2. Tactical Design — Building Blocks
3. Module Mapping
4. Aggregate Design Rules
5. Domain Events
6. Ubiquitous Language
7. Anti-Corruption Layer
8. Common Pitfalls

---

## 1. Strategic Design

### Bounded Context

A Bounded Context is a boundary within which a domain model applies consistently. Each context has its own Ubiquitous Language — the same word can mean different things in different contexts.

**Example Bounded Contexts (e-commerce Modular Monolith)**

| Context | Responsibility | Key Aggregates |
|---------|---------------|----------------|
| `identity` | Users, authentication, sessions | `User` |
| `catalog` | Products, categories, pricing | `Product`, `Category` |
| `ordering` | Orders, order lines, checkout | `Order`, `OrderLineItem` |
| `shipping` | Shipments, tracking, fulfillment | `Shipment` |
| `payment` | Payment processing, refunds | `Payment` |

In a Modular Monolith, Bounded Contexts map to **packages within modules**, not separate services.

### Context Mapping

How Bounded Contexts relate to each other:

| Pattern | When to Use | Example |
|---------|------------|---------|
| Shared Kernel | Two contexts share a model fragment | `customerId` referenced in multiple contexts |
| Customer-Supplier | One context drives another | `catalog` drives `ordering` (product prices) |
| Anti-Corruption Layer | Translate external model to internal | Payment gateway payload → `PaymentReceivedCommand` |
| Open Host Service | Provide a stable API for other contexts | `OrderService` API consumed by `shipping` |

### Ubiquitous Language

Every concept in the codebase must use the same term as the domain expert uses.

**Rule**: When you hear a new term from a product discussion, add it to a project glossary and update code names before writing new code.

```java
// Bad: generic terms that don't reflect the domain
class Thing { ... }
void doStuff(User u) { ... }

// Good: domain terms that match stakeholder vocabulary
class Order { ... }
void fulfillOrder(CustomerId customerId) { ... }
```

---

## 2. Tactical Design — Building Blocks

### Entity

Has identity that persists over time and across state changes.

```java
@Entity
@Table(name = "orders")
public class Order {
  @Id
  @Column(name = "id", columnDefinition = "uuid")
  private UUID id;                        // Identity — never changes

  @Column(name = "customer_id", nullable = false)
  private UUID customerId;

  @Enumerated(EnumType.STRING)
  private OrderStatus status;             // Mutable state

  // Behavior methods (not just getters/setters)
  public void confirm() {
    if (this.status != OrderStatus.PENDING) {
      throw new OrderAlreadyConfirmedException(this.id);
    }
    this.status = OrderStatus.CONFIRMED;
    registerEvent(new OrderConfirmedEvent(this.id, customerId, Instant.now()));
  }
}
```

**Rules**:
- Identity (`@Id`) never changes after creation
- Equality based on identity only, not state
- Behavior goes in the entity, not in a service (avoid Anemic Domain Model)

### Value Object

No identity. Defined entirely by its attributes. Always immutable.

```java
// Good: Java record as Value Object
public record Money(BigDecimal amount, Currency currency) {
  public Money {
    Objects.requireNonNull(amount, "amount required");
    Objects.requireNonNull(currency, "currency required");
    if (amount.compareTo(BigDecimal.ZERO) < 0) {
      throw new IllegalArgumentException("Amount must be non-negative");
    }
  }

  public Money add(Money other) {
    if (!this.currency.equals(other.currency)) {
      throw new IllegalArgumentException("Currency mismatch");
    }
    return new Money(this.amount.add(other.amount), this.currency);
  }
}

// Value Objects as JPA embeddable
@Embeddable
public record Address(
    @Column(name = "street") String street,
    @Column(name = "city") String city,
    @Column(name = "postal_code") String postalCode
) {}
```

**Rules**:
- No `@Id` field
- Equality based on all attributes (records give this for free)
- Replace primitives with Value Objects when the primitive has invariants
- Side-effect-free methods only (return new VO, don't mutate)

### Aggregate

A cluster of Entities and Value Objects with a single root Entity (Aggregate Root) that enforces invariants for the whole cluster.

```java
public class Order {  // Aggregate Root
  private UUID id;
  private UUID customerId;
  private List<OrderLineItem> lineItems = new ArrayList<>();  // child entities
  private OrderStatus status;

  // All modifications through root
  public OrderLineItem addLineItem(ProductId productId, int quantity, Money unitPrice) {
    validateNotConfirmed();
    var lineItem = OrderLineItem.create(this.id, productId, quantity, unitPrice);
    lineItems.add(lineItem);
    return lineItem;
  }

  // Invariant enforced here, not in service
  private void validateNotConfirmed() {
    if (status != OrderStatus.PENDING) {
      throw new OrderAlreadyConfirmedException(id);
    }
  }

  public Money calculateTotal() {
    return lineItems.stream()
        .map(OrderLineItem::subtotal)
        .reduce(Money.ZERO, Money::add);
  }
}
```

**Rules** (see §4 for the full list):
- Only the Aggregate Root has a repository
- External objects hold only the ID of another aggregate (not a reference)
- One transaction = one aggregate (usually)

### Repository

Provides persistence for exactly one Aggregate Root. The interface belongs in the **domain module**; the implementation in the **infrastructure module**.

```java
// Interface in domain module
public interface OrderRepository {
  Optional<Order> findById(UUID id);
  Optional<Order> findByIdAndCustomerId(UUID id, UUID customerId);
  Order save(Order order);
  void deleteById(UUID id);
}

// Implementation in infrastructure module
@Repository
public class JpaOrderRepository implements OrderRepository {
  private final OrderJpaRepository jpa;  // Spring Data JPA

  @Override
  public Optional<Order> findByIdAndCustomerId(UUID id, UUID customerId) {
    return jpa.findByIdAndCustomerId(id, customerId);
  }
}
```

### Domain Service

Logic that doesn't naturally belong in an Entity or Value Object, typically coordinating multiple aggregates.

```java
// In domain module: stateless, no infrastructure dependencies
public interface OrderFulfillmentService {
  void fulfill(UUID orderId, UUID warehouseId);
}

// In application module: uses repositories and other services
@Service
public class OrderFulfillmentServiceImpl implements OrderFulfillmentService {
  private final OrderRepository orderRepo;
  private final InventoryRepository inventoryRepo;

  @Transactional
  public void fulfill(UUID orderId, UUID warehouseId) {
    var order = orderRepo.findById(orderId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));
    var inventory = inventoryRepo.findByWarehouseId(warehouseId)
        .orElseThrow(() -> new InventoryNotFoundException(warehouseId));
    order.markFulfilled();
    inventory.reserveItems(order.getLineItems());
    orderRepo.save(order);
    inventoryRepo.save(inventory);
  }
}
```

**When to use Domain Service** (not Entity):
- Requires access to multiple aggregates
- Would be unnatural to put in either aggregate
- Conceptually a "verb" not a "noun" (e.g., `Transfer`, `Fulfill`, `Merge`)

### Application Service (Use Case)

Orchestrates domain objects to fulfill a use case. Lives in the application/service module. Has no domain logic.

```java
@Service
@RequiredArgsConstructor
public class PlaceOrderUseCase {
  private final OrderRepository orderRepo;
  private final ApplicationEventPublisher eventPublisher;

  @Transactional
  public OrderResponse execute(PlaceOrderCommand command) {
    // 1. Load or create aggregate
    var order = Order.create(command.customerId());

    // 2. Delegate to domain
    command.lineItems().forEach(item ->
        order.addLineItem(item.productId(), item.quantity(), item.unitPrice()));

    // 3. Persist
    orderRepo.save(order);

    // 4. Publish events
    order.getDomainEvents().forEach(eventPublisher::publishEvent);

    return OrderResponse.from(order);
  }
}
```

---

## 3. Module Mapping

```
<app>-domain                         <app>-application (or -service)
  com.example.order                    com.example.order
    Order.java (Entity)                  OrderService.java (interface)
    OrderLineItem.java (Entity)          OrderServiceImpl.java
    OrderStatus.java (enum)              PlaceOrderCommand.java (record)
    OrderRepository.java (interface)     OrderResponse.java (record)
    Money.java (VO)
    OrderConfirmedEvent.java

  com.example.customer                 com.example.customer
    Customer.java                        CustomerService.java
    CustomerRepository.java              ...

<app>-infrastructure                 <app>-api (presentation)
  com.example.infra.order              com.example.order
    JpaOrderRepository.java              OrderController.java
    OrderJpaRepository.java              (uses service, never domain directly)
```

### Dependency Rules

```
api / presentation  → application (Application Service interfaces)
application         → domain      (Domain model, Repository interfaces)
infrastructure      → domain      (Implements Repository interfaces)
common              ← (all modules depend on common)

FORBIDDEN: application → infrastructure
FORBIDDEN: domain      → application
FORBIDDEN: domain      → infrastructure
```

---

## 4. Aggregate Design Rules

### Rule 1: Reference Other Aggregates by Identity Only

```java
// Bad: holds reference to another Aggregate
public class Order {
  private Customer customer;  // full Aggregate reference
}

// Good: holds only ID
public class Order {
  private UUID customerId;  // ID reference only
}
```

### Rule 2: Enforce Invariants Within the Aggregate

```java
public class Order {
  private final int maxLineItems;
  private final List<OrderLineItem> lineItems;

  public void addLineItem(ProductId productId, int quantity, Money unitPrice) {
    if (lineItems.size() >= maxLineItems) {
      throw new OrderLineItemLimitExceededException(id, maxLineItems);
    }
    lineItems.add(OrderLineItem.create(id, productId, quantity, unitPrice));
  }
}
```

### Rule 3: One Transaction = One Aggregate (as a guideline)

When you need to update two aggregates in one use case, consider:
1. Can the second aggregate be updated eventually (via Domain Event)?
2. Is there a missing aggregate that encompasses both?
3. If truly necessary, use explicit `@Transactional` and document the reason.

### Rule 4: Small Aggregates

Large aggregates cause contention. If loading an Aggregate loads thousands of child entities, redesign.

```java
// Bad: loads ALL line items eagerly
public class Order {
  @OneToMany(fetch = FetchType.EAGER)
  private List<OrderLineItem> allLineItems;  // could be thousands
}

// Good: OrderLineItem is its own Aggregate Root with reference to Order
public class OrderLineItem {
  private UUID id;
  private UUID orderId;  // reference by ID
}
```

---

## 5. Domain Events

Domain Events represent something that happened in the domain. They are immutable facts.

```java
// Domain Event definition (domain module)
public record OrderCompletedEvent(
    UUID orderId,
    UUID customerId,
    Money totalAmount,
    Instant occurredAt
) implements DomainEvent {}

// Publishing from Aggregate Root
public class Order extends AbstractAggregateRoot<Order> {
  public void complete() {
    this.status = OrderStatus.DELIVERED;
    registerEvent(new OrderCompletedEvent(id, customerId, calculateTotal(), Instant.now()));
  }
}

// Handling in Application Service
@Service
public class OrderEventHandler {
  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void onOrderCompleted(OrderCompletedEvent event) {
    // Send confirmation email, update analytics, trigger loyalty points, etc.
    // Runs AFTER the transaction that published the event commits
  }
}
```

### Event Naming

Events are named in past tense: `OrderPlaced`, `OrderConfirmed`, `OrderShipped`, `OrderCompleted`, `PaymentReceived`.

---

## 6. Ubiquitous Language in Code

### Map Domain Terms to Code

Identify domain terms with stakeholders and map them directly to code names. Terms should be consistent across conversations, documentation, and code.

```java
// Example e-commerce glossary → code mapping
// "order" → Order, OrderRepository, OrderService
// "line item" → OrderLineItem (not "cart item" or "product line")
// "customer" → Customer (not "user" or "buyer" in order context)
// "fulfillment" → OrderFulfillment, FulfillOrderCommand
```

### Enforce Language Through Types

```java
// Bad: primitive obsession loses domain meaning
void assign(UUID id1, UUID id2) { ... }

// Good: types carry meaning
void ship(OrderId orderId, ShipmentId shipmentId) { ... }
```

### Project Glossary Pattern

Maintain a `GLOSSARY.md` or wiki entry per Bounded Context listing:
- **Term**: The domain word
- **Code name**: The Java class/method/field name
- **Forbidden synonyms**: Words that must not be used instead

---

## 7. Anti-Corruption Layer

Used when integrating external systems (webhooks, third-party APIs) to translate their model into your domain model.

```java
// External payment webhook payload (external model)
public record StripeWebhookPayload(
    String type,
    String paymentIntentId,
    long amountReceived,
    String currency,
    String status
) {}

// Anti-Corruption Layer translation
@Component
public class StripeWebhookTranslator {
  public PaymentReceivedCommand translate(StripeWebhookPayload payload) {
    return new PaymentReceivedCommand(
        payload.paymentIntentId(),
        Money.of(BigDecimal.valueOf(payload.amountReceived(), 2),
                 Currency.getInstance(payload.currency().toUpperCase())),
        Instant.now()
    );
  }
}

// Domain Command (your model)
public record PaymentReceivedCommand(
    String externalPaymentId,
    Money amount,
    Instant receivedAt
) {}
```

---

## 8. Common Pitfalls

### Anemic Domain Model

```java
// Bad: Entity is just a data bag
public class Order {
  private OrderStatus status;
  public OrderStatus getStatus() { return status; }
  public void setStatus(OrderStatus status) { this.status = status; }
}

// Bad: All logic in service
orderService.setStatus(id, CONFIRMED);  // service does everything

// Good: Behavior in entity, invariants enforced
order.confirm();  // entity validates + emits event
```

### Transaction Script Leaking into Domain

```java
// Bad: Domain object depends on repository
public class Order {
  @Autowired OrderRepository repo;  // never inject infra into domain

  public void confirm() {
    this.status = CONFIRMED;
    repo.save(this);  // domain should not know about persistence
  }
}
```

### Exposing Internal Aggregate Structure

```java
// Bad: returns mutable internal list
public List<OrderLineItem> getLineItems() { return lineItems; }

// Good: return unmodifiable view or DTO
public List<OrderLineItem> getLineItems() { return Collections.unmodifiableList(lineItems); }
```

---

## Cross-References

- For SOLID principles that complement DDD → `design-and-solid.md`
- For Effective Java patterns used in Value Objects and Builders → `effective-java.md`
- For stability patterns for Domain Event publishing → `release-it-stability.md`
