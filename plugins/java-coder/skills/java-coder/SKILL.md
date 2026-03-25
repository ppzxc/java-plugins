---
name: java-coder
description: >
  Use when writing, designing, or refactoring Java code.
  Trigger for .java file creation or modification, REST API design, Spring Boot configuration,
  JPA entity modeling, or domain modeling involving Java 25, Spring Boot 4.
  NOT for test file writing, code review, or quality audits.
---

# Java Coder — Integrated Guide (11-Book Edition)

Unified skill combining software engineering principles from 11 landmark books with Java 25 + Spring Boot 4 conventions.

## Integrated Principle Sources

| Abbr | Book | Author | Core Area |
|------|------|--------|-----------|
| CC | Clean Code | Robert C. Martin | Code quality, naming, function design |
| SOLID | Agile Software Development | Robert C. Martin | 5 OO design principles |
| PP | The Pragmatic Programmer | Hunt & Thomas | Development philosophy, practical habits |
| CodeC | Code Complete | Steve McConnell | Software construction |
| RF | Refactoring | Martin Fowler | Code structure improvement |
| DP | Design Patterns (GoF) | Gamma et al. | Reusable design patterns |
| TDD | Test-Driven Development | Kent Beck | Test-driven development |
| LEG | Working Effectively with Legacy Code | Michael Feathers | Legacy code strategies |
| EJ | Effective Java | Joshua Bloch | Java idioms and best practices |
| DDD | Domain-Driven Design | Eric Evans | Strategic/tactical domain modeling |
| RI | Release It! | Michael Nygard | Production stability patterns |

## Core Principles

1. **Readability first** (CC): Code is read far more often than it is written
2. **Consistency** (PP): Apply the same style across the entire project
3. **Embrace modern Java** (EJ): Actively use Java 25 features and proven idioms
4. **Follow Spring Boot 4 idioms**: Adopt patterns built on Spring Framework 7
5. **Model the domain explicitly** (DDD): Code reflects the Ubiquitous Language; domain objects carry behavior
6. **Design for failure** (RI): Every external call can fail; timeouts, circuit breakers, and bulkheads are not optional
7. **Prefer proven idioms** (EJ): Static factories, builders, immutability, proper equals/hashCode

## Code Writing Workflow

### Step 1: Think About Design (SOLID, DP, DDD)

Before writing code, think about structure and domain model.

- **Bounded Context**: Which layer/module owns this concept? (domain / application / infrastructure / presentation)
- **Aggregate boundary**: What is the transaction boundary? Access through the Aggregate Root.
- **SOLID**: Apply SRP, OCP, LSP, ISP, DIP — especially DIP via constructor injection
- **Pattern**: Apply GoF patterns deliberately, not mechanically

→ See `references/design-and-solid.md` for SOLID/GoF details
→ See `references/domain-driven-design.md` for DDD patterns and module mapping

### Step 2: Write Clean Code (CC, PP, CodeC, EJ)

- **Naming**: Reveal intent. Use domain terms from Ubiquitous Language.
- **Functions**: Do one thing. ≤15 lines. Uniform abstraction level.
- **EJ idioms**: Static factories over constructors, Builder for ≥4 params, immutable Value Objects
- **Null safety**: Return `Optional` or empty collections, never null. No null parameters.

→ See `references/clean-and-pragmatic.md` for detailed naming/function/error handling rules
→ See `references/effective-java.md` for EJ idioms (static factory, Builder, enum, generics)

### Step 3: Write Tests First (TDD)

→ Activate **java-tester** skill for the full TDD workflow, coverage checklist, and test patterns.

### Step 4: Refactor (RF, LEG)

After tests are green, improve structure:

- **Extract Method/Class** for duplicated logic
- **Replace conditional with polymorphism** for type-switching
- **Introduce Parameter Object** when ≥3 related params
- Leave code cleaner than you found it (Boy Scout Rule)

→ See `references/refactoring-catalog.md` for refactoring catalog

### Step 5: Harden for Production (RI)

**트리거**: 아래 키워드가 코드에 포함되면 이 단계는 **필수**:
- `RestClient`, `WebClient`, `@HttpExchange`, `RestTemplate` → Timeout + Circuit Breaker
- `JavaMailSender`, `mailSender.send` → Timeout + Circuit Breaker
- `RedisTemplate`, `StringRedisTemplate`, `ValueOperations` → Timeout + Fallback
- `JdbcTemplate`, `EntityManager`, `@Query` (외부 DB) → Timeout
- `HttpURLConnection`, `URL.openStream` → Timeout + Circuit Breaker + Retry

**필수 체크**:
1. **Timeout**: `connectTimeout` + `readTimeout` 명시적 설정 확인. 설정 없으면 작성 중단하고 추가.
2. **Circuit Breaker**: 불안정한 의존성(이메일, 푸시, 외부 API, 웹훅)에 Resilience4j `@CircuitBreaker` 적용 확인.
3. **Fallback**: Redis 장애 시 degraded mode 가능 여부 검토 (특히 인증 경로).
4. **Retry**: 멱등 연산(GET, PUT)만. POST는 멱등키 없으면 retry 금지.
5. **Virtual Thread 안전**: `synchronized` → `ReentrantLock`, `ThreadLocal` → `ScopedValue` → See **java-25** skill

→ See `references/release-it-stability.md` for stability patterns and anti-patterns

---

## Formatting & Naming

**Style**: Google Java Style, max 120 chars/line, 2-space indent, `google-java-format`

| Element | Convention | Example |
|---------|-----------|---------|
| Class | PascalCase | `OrderService` |
| Method/Variable | camelCase | `findByCustomerId` |
| Constant | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Redis key | snake_case + colon separator → constants class | `session:user:{userId}` |
| DB column | snake_case | `customer_id` |
| Package | lowercase, domain-aligned | `com.example.order` |

**DTO Rules**:
- Always use `record`
- Request DTOs: Bean Validation annotations required
- Never expose `@Entity` in controller — convert in Service layer
- **No inner classes/records** — every DTO in its own file

---

## Java 25 Feature Usage

→ See **java-25** skill for the full Java 25 feature reference, Virtual Thread safety rules, and usage conventions.

---

## Spring Boot 4 Conventions

→ See **spring** skill for Spring Boot 4 conventions, test patterns, and review checklist.

---

## Effective Java Essentials

Full reference → `references/effective-java.md`

**Object Creation**
```java
// Prefer static factory over constructor
public static OrderId of(UUID value) { return new OrderId(value); }

// Builder for ≥4 params or optional fields
var cmd = CreateOrderCommand.builder()
    .customerId(customerId)
    .itemCode(itemCode)
    .quantity(qty)
    .build();
```

**Immutability**
```java
// Value Objects should be immutable records
public record Money(BigDecimal amount, Currency currency) {
  public Money add(Money other) { return new Money(amount.add(other.amount), currency); }
}
```

**Optional** — return type only, never as field/parameter
```java
Optional<User> findByEmail(String email);  // OK
// NOT: Optional<String> field;            // Bad
// NOT: void method(Optional<X> param);    // Bad
```

**Enum over int constants**
```java
public enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

---

## DDD in Practice

Full reference → `references/domain-driven-design.md`

### Module Mapping

| DDD Layer | Module | Package Pattern |
|-----------|--------|-----------------|
| Domain Model (Entity, VO, Aggregate) | `<app>-domain` | `com.example.<context>` |
| Application Service | `<app>-application` | `com.example.<context>` |
| Repository Interface | `<app>-domain` | `com.example.<context>` |
| Repository Implementation | `<app>-infrastructure` | `com.example.infra.<context>` |
| REST Controller | `<app>-api` | `com.example.<context>` |

### Aggregate Rules
```java
// Access child entities through Aggregate Root
order.addLineItem(productId, quantity, unitPrice);

// Reference other aggregates by ID only
public record OrderRef(UUID orderId) {}  // NOT the full Order object
```

### Domain Events
```java
// Publish events from Aggregate Root, handle in Application Service
public class Order extends AbstractAggregateRoot<Order> {
  public void complete() {
    this.status = OrderStatus.DELIVERED;
    registerEvent(new OrderCompletedEvent(this.id, customerId, Instant.now()));
  }
}
```

---

## Stability Patterns (Release It!)

Full reference → `references/release-it-stability.md`

| Pattern | Application |
|---------|-------------|
| Timeout | All cache/DB/HTTP calls — explicit `connectTimeout` + `readTimeout` |
| Circuit Breaker | Push notification, Email, external webhook endpoints |
| Bulkhead | Separate Virtual Thread executor for CPU-bound tasks |
| Fail Fast | Validate tenant/context at filter/controller boundary |
| Retry + Backoff | Idempotent operations only (GET, PUT); never on POST without idempotency key |

**Virtual Thread Safety** (non-negotiable)
```java
// Virtual Thread safe
private final ReentrantLock lock = new ReentrantLock();

// Pins Virtual Thread carrier thread — forbidden
synchronized void criticalSection() { ... }
```

---

## Error Handling / Logging

**Errors** (RFC 9457 ProblemDetail):
```java
var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, "Order not found");
pd.setType(URI.create("/errors/order-not-found"));
pd.setProperty("orderId", id);
```

**Logging** (`@Slf4j`, structured, masked):
```java
log.info("Order placed: orderId={}, tenantId={}", orderId, tenantId);
// Never log: passwords, tokens, card numbers, PII
```

---

## Reference File Guide

| File | When to Open |
|------|-------------|
| `references/clean-and-pragmatic.md` | Naming, function size, comments, error handling (CC + PP + CodeC) |
| `references/design-and-solid.md` | SOLID deep-dive, GoF patterns, anti-patterns |
| `references/refactoring-catalog.md` | Specific refactoring technique lookup (RF) |
| **java-25** skill | Java 25 features — records, sealed classes, virtual threads, ScopedValue |
| **spring** skill | Spring Boot 4 conventions, test patterns, review checklist |
| `references/effective-java.md` | Static factory, Builder, generics, enum, Optional, concurrency |
| `references/domain-driven-design.md` | Aggregate, Value Object, Domain Event, Context Mapping, module mapping |
| `references/release-it-stability.md` | Circuit Breaker, Bulkhead, Timeout, Retry, anti-patterns |
