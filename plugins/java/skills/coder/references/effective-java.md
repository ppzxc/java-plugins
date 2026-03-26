# Effective Java — Core Idioms and Best Practices

Based on *Effective Java* (3rd edition) by Joshua Bloch. Items most relevant to Java 25 + Spring Boot 4 projects.

## Table of Contents
1. Object Creation
2. Immutability and Defensive Copying
3. Generics
4. Enums and Annotations
5. Streams and Lambdas
6. Optional
7. Concurrency (Virtual Thread context)
8. Interfaces and Design

---

## 1. Object Creation

### Prefer Static Factory Methods over Constructors (Item 1)

Static factories have names, can return cached instances, and can return subtypes.

```java
// Bad: constructor reveals nothing
new OrderId(uuid);

// Good: intent is clear
OrderId.of(uuid);
OrderId.fromString("550e8400-...");

// Cached instance (e.g., known enum-like values)
public static final OrderStatus PENDING = new OrderStatus("PENDING");
public static OrderStatus of(String code) {
  return REGISTRY.computeIfAbsent(code, OrderStatus::new);
}
```

Common naming conventions: `of`, `from`, `valueOf`, `getInstance`, `newInstance`, `create`, `get<Type>`.

### Builder for Complex Objects (Item 2)

Use when a class has ≥4 parameters or many optional fields. Lombok `@Builder` is acceptable; hand-rolled builder for domain objects needing validation.

```java
// Domain object with validation in builder
public final class CreateOrderCommand {
  private final UUID customerId;    // required
  private final String itemCode;    // required
  private final int quantity;       // required
  private final String couponCode;  // optional
  private final UUID attachmentId;  // optional

  private CreateOrderCommand(Builder b) { ... }

  public static Builder builder() { return new Builder(); }

  public static final class Builder {
    // required fields as constructor params or explicit setters with validation
    public CreateOrderCommand build() {
      Objects.requireNonNull(customerId, "customerId required");
      Objects.requireNonNull(itemCode, "itemCode required");
      return new CreateOrderCommand(this);
    }
  }
}
```

### Avoid Creating Unnecessary Objects (Item 6)

```java
// Bad: new String on every call
String s = new String("hello");

// Good: literal reuse
String s = "hello";

// Bad: autoboxing in tight loop
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;  // boxes each addition

// Good: primitive
long sum = 0L;
```

### Eliminate Obsolete Object References (Item 7)

Null out references in stack-like structures when elements are popped. Be careful with caches — use `WeakHashMap` or time-based eviction.

### Prefer try-with-resources (Item 9)

```java
// Always for Closeable resources
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(SQL)) {
  // use resources
} // both auto-closed, even if exception
```

---

## 2. Immutability and Defensive Copying

### Minimize Mutability (Item 17)

Five rules for immutable classes:
1. No methods that modify state
2. Class cannot be extended (`final` or private constructor + static factory)
3. All fields `final`
4. All fields `private`
5. Exclusive access to mutable components (defensive copy)

```java
// Immutable Value Object using record (Java 16+)
public record PhoneNumber(String e164) {
  public PhoneNumber {
    Objects.requireNonNull(e164);
    if (!e164.matches("\\+[1-9]\\d{1,14}")) {
      throw new IllegalArgumentException("Not a valid E.164 number: " + e164);
    }
  }
}
```

If a class cannot be made fully immutable, limit mutability as much as possible — make every field `final` unless there is a compelling reason to make it non-final.

### Defensive Copies (Item 50)

Copy mutable objects passed to or returned from a class.

```java
public final class Period {
  private final Date start;
  private final Date end;

  public Period(Date start, Date end) {
    // Defensive copy BEFORE validation — prevents TOCTOU attack
    this.start = new Date(start.getTime());
    this.end   = new Date(end.getTime());
    if (this.start.after(this.end)) throw new IllegalArgumentException("start after end");
  }

  public Date start() { return new Date(start.getTime()); }  // defensive copy on return
}
```

**Prefer `Instant`, `LocalDate`, `Duration`** (already immutable) over `Date`.

### Composition over Inheritance (Item 18)

Inheritance breaks encapsulation. Prefer delegation/composition when reusing behavior across classes that do not have a genuine is-a relationship.

```java
// Forwarding wrapper (composition)
public class InstrumentedSet<E> implements Set<E> {
  private final Set<E> delegate;
  private int addCount = 0;

  public InstrumentedSet(Set<E> s) { this.delegate = s; }

  @Override public boolean add(E e) { addCount++; return delegate.add(e); }
  // delegate all other Set methods to this.delegate
}
```

---

## 3. Generics

### Don't Use Raw Types (Item 26)

```java
// Bad — raw type loses type safety
List list = new ArrayList();
list.add("text");

// Good
List<String> list = new ArrayList<>();
```

### Use Bounded Wildcards for Flexibility: PECS (Item 31)

**P**roducer **E**xtends, **C**onsumer **S**uper

```java
// Producer (you read from it) — use extends
public void pushAll(Iterable<? extends E> src) {
  for (E e : src) push(e);
}

// Consumer (you write into it) — use super
public void popAll(Collection<? super E> dst) {
  while (!isEmpty()) dst.add(pop());
}
```

### Prefer Generic Methods (Item 30)

```java
// Type-safe generic utility
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
  var result = new HashSet<>(s1);
  result.addAll(s2);
  return result;
}
```

### Typesafe Heterogeneous Container (Item 33)

When you need a map from type to instance of that type:

```java
public class Favorites {
  private final Map<Class<?>, Object> favorites = new HashMap<>();

  public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(Objects.requireNonNull(type), instance);
  }

  public <T> T getFavorite(Class<T> type) {
    return type.cast(favorites.get(type));
  }
}
```

---

## 4. Enums and Annotations

### Use Enums Instead of int Constants (Item 34)

```java
// Bad
public static final int STATUS_PENDING  = 0;
public static final int STATUS_SENT     = 1;

// Good — type-safe, has namespace, printable
public enum OrderStatus {
  PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED;

  public boolean isTerminal() {
    return this == DELIVERED || this == CANCELLED;
  }
}
```

### Use EnumMap (Item 37)

Prefer `EnumMap` over ordinal indexing; it is as fast as an array but type-safe.

```java
var ordersByStatus = new EnumMap<OrderStatus, List<Order>>(OrderStatus.class);
```

### Strategy Enum Pattern (Item 34)

```java
public enum RetryStrategy {
  IMMEDIATE {
    @Override public long delayMs(int attempt) { return 0; }
  },
  EXPONENTIAL_BACKOFF {
    @Override public long delayMs(int attempt) { return (long) Math.pow(2, attempt) * 100; }
  };

  public abstract long delayMs(int attempt);
}
```

### Annotations over Naming Conventions (Item 39)

Use `@Override` on every method you intend to override. It catches bugs at compile time.

---

## 5. Streams and Lambdas

### Use Streams Judiciously (Item 45)

Streams excel at:
- Transforming sequences (map, filter, flatMap)
- Grouping and collecting (Collectors)
- Finding elements (findFirst, anyMatch)

Prefer loops when:
- You need to modify local variables
- You need to break/continue/return from an enclosing scope
- You need checked exceptions

### Prefer Method References over Lambdas (Item 43)

```java
// Lambda — acceptable
list.stream().map(s -> s.toUpperCase())

// Method reference — cleaner
list.stream().map(String::toUpperCase)
```

### Use Standard Functional Interfaces (Item 44)

`java.util.function` provides 43 interfaces; use them instead of inventing your own.

| Interface | Signature | Example |
|-----------|-----------|---------|
| `Predicate<T>` | `T → boolean` | filter condition |
| `Function<T,R>` | `T → R` | transform |
| `Supplier<T>` | `() → T` | lazy factory |
| `Consumer<T>` | `T → void` | log/notify |
| `UnaryOperator<T>` | `T → T` | transform same type |

### Prefer Side-Effect-Free Functions (Item 46)

Stream operations should be pure functions. Side effects (DB writes, logging) belong in `forEach` at the terminal operation, not in intermediate ops.

```java
// Bad: side effect in intermediate map
orders.stream()
    .map(o -> { repository.save(o); return o; })  // don't
    .collect(toList());

// Good: collect first, then act
var saved = orders.stream()
    .map(repository::save)
    .collect(toList());
```

### Collectors (Item 46)

```java
// Grouping
var byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));

// Joining
var ids = ids.stream()
    .map(UUID::toString)
    .collect(Collectors.joining(", "));

// Counting
var countByStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));
```

---

## 6. Optional

### Optional as Return Type Only (Item 55)

`Optional` is designed exclusively as a method return type to indicate a value that may be absent.

```java
// Good: return type
public Optional<User> findByEmail(String email) { ... }

// Bad: Optional field
private Optional<String> nickname;  // Serialization issues, unnecessary overhead

// Bad: Optional parameter
public void process(Optional<String> name) { ... }  // Just use overloading or nullable

// Bad: Optional in collection
List<Optional<Order>> orders;  // Use List<Order> with absent items filtered out
```

### Return Empty Collections, Not Null (Item 54)

```java
// Bad
public List<Order> findAll() {
  if (noResults) return null;
}

// Good
public List<Order> findAll() {
  if (noResults) return Collections.emptyList();
}
```

### Optional Combinators

```java
// orElse — always evaluated
var name = findByEmail(email).orElse("Unknown");

// orElseGet — lazy, evaluated only if empty (preferred for expensive defaults)
var user = findById(id).orElseGet(() -> createGuestUser());

// orElseThrow — throw if absent
var order = findOrderById(id).orElseThrow(() -> new OrderNotFoundException(id));

// map + flatMap
var customerName = findOrder(orderId)
    .map(Order::getCustomerId)
    .flatMap(this::findCustomer)
    .map(Customer::getName)
    .orElse("Unknown customer");
```

---

## 7. Concurrency (Virtual Thread Context)

### Prefer Concurrency Utilities over wait/notify (Item 81)

Use `java.util.concurrent` — `CountDownLatch`, `Semaphore`, `BlockingQueue`, `CompletableFuture`.
Never implement your own synchronization primitives.

### ReentrantLock over synchronized

Virtual Threads are pinned when blocking inside a `synchronized` block. Use `ReentrantLock` instead.

```java
// Bad — pins Virtual Thread
synchronized void process() { expensiveIO(); }

// Good — Virtual Thread safe
private final ReentrantLock lock = new ReentrantLock();

void process() {
  lock.lock();
  try {
    expensiveIO();
  } finally {
    lock.unlock();
  }
}
```

### ScopedValue over ThreadLocal (Item 83 analog, Java 25)

`ThreadLocal` does not compose well with Virtual Threads and structured concurrency.

```java
// Bad — ThreadLocal leaks across Virtual Threads
private static final ThreadLocal<RequestContext> CTX = new ThreadLocal<>();

// Good — ScopedValue, bounded lifetime
private static final ScopedValue<RequestContext> CTX = ScopedValue.newInstance();

ScopedValue.where(CTX, requestContext).run(() -> {
  // CTX is readable within this scope
  processRequest();
});
```

### Avoid Excessive Synchronization (Item 79)

Do as little as possible inside synchronized regions. Never call alien methods (methods you don't control) while holding a lock — it can cause deadlocks or liveness failures.

### Document Thread Safety (Item 82)

Every class must document its thread-safety level:
- `@Immutable` — always thread-safe
- `@ThreadSafe` — safe for concurrent use
- `@NotThreadSafe` — not safe

---

## 8. Interfaces and Design

### Use Interfaces to Define Types (Item 41)

Interfaces should define what a type can do, not accumulate constants.

```java
// Bad — constant interface anti-pattern
public interface PhysicalConstants {
  static final double AVOGADROS_NUMBER = 6.022_140_857e23;
}

// Good — constants in enum or utility class
public enum PhysicalConstant {
  AVOGADROS_NUMBER(6.022_140_857e23);
  private final double value;
  // ...
}
```

### Design Interfaces for Posterity (Item 21)

Default methods added to interfaces can break existing implementations. Test new default methods with at least three concrete implementations before releasing.

### Use Abstract Classes for Skeletal Implementations (Item 20)

When providing a non-trivial implementation alongside an interface, use the Template Method pattern with a `Abstract<Type>` class.

```java
public interface Repository<T, ID> { ... }
public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
  // Common CRUD implementation
}
public class JpaOrderRepository extends AbstractRepository<Order, UUID> { ... }
```

---

## Cross-References

- For DDD-aligned Value Object patterns → `domain-driven-design.md`
- For Clean Code naming rules (complementary) → `clean-and-pragmatic.md`
- For Java 25 records/sealed classes (modern EJ idioms) → `java25-features.md`
