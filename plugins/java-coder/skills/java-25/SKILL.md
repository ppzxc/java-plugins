---
name: java-25
description: >
  Use when writing or reviewing Java code that uses Java 25 features — records, sealed classes,
  pattern matching (instanceof/switch), virtual threads, ScopedValue, structured concurrency,
  stable values, or other Java 25+ language and API features.
  Also use when checking Virtual Thread safety (synchronized blocks, ThreadLocal usage).
---

# Java 25

Java 25 (LTS, September 2025) feature guide with usage conventions and best practices.

## Feature Quick Reference

| Feature | When to Use |
|---------|------------|
| `var` | Local variables for type inference |
| Records | DTOs, Value Objects, immutable data carriers |
| Sealed classes | Algebraic types, closed hierarchies (domain events, payment methods) |
| Pattern matching (`instanceof`, `switch`) | Replace `instanceof` casts, eliminate type-switching boilerplate |
| Virtual Threads | All I/O-bound work (Spring Boot 4 default) |
| `ScopedValue` | Replace `ThreadLocal` in Virtual Thread contexts |
| Text blocks | Multi-line SQL, JSON templates |
| `SequencedCollection` | When insertion order + first/last access matters |
| Structured Concurrency | Coordinate multiple concurrent tasks as a single logical unit |
| Stable Values | Lazy initialization replacing double-checked locking |

Full reference → `../java-coder/references/java25-features.md`

---

## Virtual Thread Safety (non-negotiable)

```java
// ✅ Virtual Thread safe — use ReentrantLock
private final ReentrantLock lock = new ReentrantLock();

// ❌ Pins Virtual Thread carrier thread — forbidden
synchronized void criticalSection() { ... }

// ✅ Virtual Thread safe — use ScopedValue
private static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

// ❌ Leaks across virtual threads — forbidden
private static final ThreadLocal<String> USER = ThreadLocal.withInitial(() -> null);
```

Rules:
- `synchronized` → `ReentrantLock`
- `ThreadLocal` → `ScopedValue`
- CPU-bound tasks → keep platform thread pool (don't use Virtual Threads for CPU work)

---

## Records

Use for DTOs, Value Objects, and immutable data carriers.

```java
// Value Object
public record Money(BigDecimal amount, Currency currency) {
  public Money add(Money other) {
    if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
    return new Money(this.amount.add(other.amount), this.currency);
  }
}
```

**Do NOT use records for:** JPA entities (require default constructor + mutable fields), types needing inheritance.

---

## Sealed Classes

Explicitly restrict subtypes; combine with pattern matching switch for exhaustive handling.

```java
public sealed interface PaymentMethod permits CreditCard, BankTransfer, DigitalWallet {}

public record CreditCard(String cardNumber, YearMonth expiry) implements PaymentMethod {}
public record BankTransfer(String bankCode, String accountNumber) implements PaymentMethod {}

// Exhaustive switch — compiler enforces all cases, no default needed
public BigDecimal calculateFee(PaymentMethod method) {
  return switch (method) {
    case CreditCard cc -> new BigDecimal("0.03");
    case BankTransfer _ -> new BigDecimal("0.01");
    case DigitalWallet dw -> BigDecimal.ZERO;
  };
}
```

---

## ScopedValue

Replaces `ThreadLocal` in Virtual Thread environments. Immutable and auto-cleaned when scope ends.

```java
public class RequestContext {
  private static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();
  public static ScopedValue<String> currentUser() { return CURRENT_USER; }
}

ScopedValue.runWhere(RequestContext.currentUser(), authenticatedUserId, () -> {
  orderService.createOrder(request);
});
```

---

## Reference File Guide

| File | When to Open |
|------|-------------|
| `../java-coder/references/java25-features.md` | Complete Java 25 reference — all features with examples |
