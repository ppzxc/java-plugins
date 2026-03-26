# Java 25 Feature Usage Guide

This document covers Java 25 (LTS, released September 2025) features that have the most impact on everyday coding.

## Table of Contents
1. [Records and Pattern Matching](#records-and-pattern-matching)
2. [Sealed Classes / Interfaces](#sealed-classes--interfaces)
3. [Switch Expressions & Pattern Matching](#switch-expressions--pattern-matching)
4. [Primitive Types in Patterns](#primitive-types-in-patterns)
5. [Flexible Constructor Bodies](#flexible-constructor-bodies)
6. [Scoped Values](#scoped-values)
7. [Structured Concurrency](#structured-concurrency)
8. [Stable Values](#stable-values)
9. [Module Import Declarations](#module-import-declarations)
10. [Compact Source Files & Instance Main Methods](#compact-source-files)
11. [Text Blocks](#text-blocks)
12. [Virtual Threads](#virtual-threads)
13. [Other Useful Features](#other-useful-features)

---

## Records and Pattern Matching

Use records for immutable data carriers. They are ideal for DTOs, value objects, and events.

**Do use:**
```java
// DTO
public record CreateOrderRequest(
    String itemCode,
    int quantity,
    @Nullable String couponCode
) {
  // Compact constructor for validation
  public CreateOrderRequest {
    if (quantity <= 0) {
      throw new IllegalArgumentException("Quantity must be positive: " + quantity);
    }
  }
}

// Value object
public record Money(BigDecimal amount, Currency currency) {
  public Money add(Money other) {
    if (!this.currency.equals(other.currency)) {
      throw new IllegalArgumentException("Currency mismatch");
    }
    return new Money(this.amount.add(other.amount), this.currency);
  }
}
```

**Record pattern deconstruction:**
```java
if (shape instanceof Rectangle(Point(var x1, var y1), Point(var x2, var y2))) {
  double width = Math.abs(x2 - x1);
  double height = Math.abs(y2 - y1);
}
```

**Do NOT use records for:**
- JPA Entities (require default constructor and mutable fields)
- Types that need inheritance
- Complex domain objects with extensive business logic

---

## Sealed Classes / Interfaces

Explicitly restrict the set of permitted subtypes for a domain type. When combined with pattern matching switch, the compiler enforces exhaustiveness.

```java
public sealed interface PaymentMethod
    permits CreditCard, BankTransfer, DigitalWallet {
}

public record CreditCard(String cardNumber, YearMonth expiry) implements PaymentMethod {}
public record BankTransfer(String bankCode, String accountNumber) implements PaymentMethod {}
public record DigitalWallet(String walletId, WalletProvider provider) implements PaymentMethod {}

// Exhaustive switch — no default needed
public BigDecimal calculateFee(PaymentMethod method) {
  return switch (method) {
    case CreditCard cc -> cc.expiry().isBefore(YearMonth.now())
        ? BigDecimal.ZERO
        : new BigDecimal("0.03");
    case BankTransfer _ -> new BigDecimal("0.01");
    case DigitalWallet dw -> switch (dw.provider()) {
      case KAKAO_PAY -> BigDecimal.ZERO;
      default -> new BigDecimal("0.02");
    };
  };
}
```

---

## Switch Expressions & Pattern Matching

Use switch expressions for value-returning branches. They are more readable than if-else chains.

```java
// Basic switch expression
String label = switch (status) {
  case PENDING -> "Pending";
  case APPROVED -> "Approved";
  case REJECTED -> "Rejected";
};

// Pattern matching with guards (when clause)
String describe(Object obj) {
  return switch (obj) {
    case Integer i when i > 0 -> "Positive integer: " + i;
    case Integer i -> "Integer: " + i;
    case String s when s.isBlank() -> "Blank string";
    case String s -> "String: " + s;
    case null -> "null";
    default -> "Other: " + obj;
  };
}
```

**Conventions:**
- Default to `->` (arrow) syntax in switch expressions
- Use `:` (colon) syntax only when fall-through is needed (should be extremely rare)
- Use `_` (unnamed variable) for unused bindings

---

## Primitive Types in Patterns

Java 25 allows primitive type patterns in instanceof and switch (Preview).

```java
// Primitive pattern in instanceof
static void process(Object obj) {
  if (obj instanceof int i) {
    System.out.println("Integer: " + i);
  }
}

// Primitive type in switch
static String classify(int value) {
  return switch (value) {
    case 0 -> "zero";
    case int i when i > 0 -> "positive";
    case int i -> "negative";
  };
}
```

> This feature is in Preview in Java 25. Use in production only after team consensus.

---

## Flexible Constructor Bodies

Java 25 allows validation, logging, and transformations before super() or this() calls.

```java
public class PremiumOrder extends Order {

  public PremiumOrder(String itemCode, int quantity, String membershipId) {
    // Validation before super() — Java 25
    if (membershipId == null || membershipId.isBlank()) {
      throw new IllegalArgumentException("Premium orders require a membership ID");
    }
    var normalizedId = membershipId.strip().toUpperCase();
    super(itemCode, quantity);
    this.membershipId = normalizedId;
  }
}
```

---

## Scoped Values

Use ScopedValue instead of ThreadLocal in Virtual Thread environments. Scoped values are immutable and automatically cleaned up when the scope ends.

```java
public class RequestContext {

  private static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

  public static ScopedValue<String> currentUser() {
    return CURRENT_USER;
  }
}

// Usage
ScopedValue.runWhere(RequestContext.currentUser(), authenticatedUserId, () -> {
  orderService.createOrder(request);
  // Downstream methods access via RequestContext.currentUser().get()
});
```

**Conventions:**
- Declare ScopedValue instances as `private static final` with accessor methods
- Use `ScopedValue.callWhere` for nested bindings that return values
- Particularly useful in thread-per-request + Virtual Thread environments

---

## Structured Concurrency

Coordinate multiple concurrent tasks as a single logical unit. If one subtask fails, the rest are automatically cancelled.

```java
OrderSummary fetchOrderSummary(long orderId) throws Exception {
  try (var scope = StructuredTaskScope.open()) {
    var orderTask = scope.fork(() -> orderRepository.findById(orderId));
    var paymentTask = scope.fork(() -> paymentClient.getPaymentStatus(orderId));
    var deliveryTask = scope.fork(() -> deliveryClient.getTrackingInfo(orderId));

    scope.join();

    return new OrderSummary(
        orderTask.get(),
        paymentTask.get(),
        deliveryTask.get()
    );
  }
}
```

---

## Stable Values

Use for lazily initialized immutable values. The JVM optimizes them like final fields.

```java
@Service
public class NotificationService {

  // Initialized once on first access
  private final StableValue<EmailClient> emailClient = StableValue.of();

  private EmailClient getEmailClient() {
    return emailClient.orElseSet(() -> EmailClient.create(emailConfig));
  }
}
```

**Conventions:**
- Use when initialization is expensive and should be deferred
- Replace double-checked locking patterns with StableValue
- Preview in Java 25 — use after team consensus

---

## Module Import Declarations

`import module` imports all exported packages from a module at once.

```java
import module java.base; // imports java.util, java.io, java.time, etc.
import module java.sql;
```

**Conventions:**
- Using `import module java.base` significantly reduces boilerplate
- Standardize usage across the project (no mixing with individual imports for the same module)
- Keep individual imports when you want explicit visibility of which package a class comes from

---

## Compact Source Files

Single-class programs can omit `public`, `static`, and `String[] args`. Suitable for prototypes and scripts, but **keep the traditional format for production Spring Boot projects**.

```java
// Compact entry point (learning / prototypes only)
void main() {
  System.out.println("Hello, Java 25!");
}
```

---

## Text Blocks

Use text blocks for multi-line strings.

```java
String query = """
    SELECT o.id, o.status, o.total_amount
    FROM orders o
    WHERE o.created_at >= :startDate
      AND o.status = :status
    ORDER BY o.created_at DESC
    """;

String json = """
    {
      "orderId": "%s",
      "status": "CREATED"
    }
    """.formatted(orderId);
```

---

## Virtual Threads

In Spring Boot 4, enable with `spring.threads.virtual.enabled=true`.

**Conventions:**
- Default to Virtual Threads for I/O-bound workloads
- Use `ReentrantLock` instead of `synchronized` blocks (prevents Virtual Thread pinning)
- Use ScopedValue instead of ThreadLocal
- Keep platform thread pools for CPU-bound work

---

## Other Useful Features

### Unnamed Variables (`_`)
Use `_` for unused variables:
```java
try {
  processOrder(order);
} catch (OrderNotFoundException _) {
  return defaultOrder();
}

orders.stream()
    .map((var _) -> generateSequenceNumber())
    .toList();
```

### Sequenced Collections
Use `getFirst()`, `getLast()`, `reversed()` on ordered collections:
```java
var firstOrder = orders.getFirst();
var lastOrder = orders.getLast();
var reversed = orders.reversed();
```

### Key Derivation Function API
Use the standard API for password-based key derivation (no third-party libraries needed):
```java
KDF hkdf = KDF.getInstance("HKDF-SHA256");
SecretKey key = hkdf.deriveKey("AES", params);
```
