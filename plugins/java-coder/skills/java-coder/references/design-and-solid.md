# Design Principles and Design Patterns (SOLID + GoF Design Patterns)

## Table of Contents
1. SOLID Principles In Depth
2. Frequently Used Design Patterns in Java
3. Pattern Selection Guide
4. Anti-Pattern Avoidance

---

## 1. SOLID Principles In Depth

### SRP (Single Responsibility Principle)

If a class has more than one reason to change, it violates SRP.

**Violation example:**
```java
// Bad: Two responsibilities — order processing and email sending
public class OrderService {
    public void processOrder(Order order) { /* order logic */ }
    public void sendConfirmationEmail(Order order) { /* email logic */ }
}
```

**Improved:**
```java
public class OrderProcessor {
    public void process(Order order) { /* order logic only */ }
}

public class OrderNotifier {
    public void sendConfirmation(Order order) { /* notification logic only */ }
}
```

**Judgment criteria**: If you cannot describe why this class might change in a single sentence, split it.

### OCP (Open-Closed Principle)

New features should be addable without modifying existing code.

**Key strategy**: Create extension points through abstractions (interfaces / abstract classes).

```java
// OCP applied: Adding a new discount policy requires no change to existing code
public interface DiscountPolicy {
    Money calculateDiscount(Order order);
}

public class PercentageDiscount implements DiscountPolicy {
    private final double rate;
    public PercentageDiscount(double rate) { this.rate = rate; }
    public Money calculateDiscount(Order order) {
        return order.getTotalPrice().multiply(rate);
    }
}

// Adding a new policy: just add a new class, no existing code modified
public class BulkDiscount implements DiscountPolicy {
    public Money calculateDiscount(Order order) {
        if (order.getItemCount() > 10) {
            return order.getTotalPrice().multiply(0.15);
        }
        return Money.ZERO;
    }
}
```

**Java 25: sealed interface로 닫힌 계층 + pattern matching switch (OCP 강화)**
```java
// 허용된 구현체를 컴파일 타임에 고정 — 새 구현 추가 시 switch가 컴파일 에러로 알려줌
public sealed interface DiscountPolicy permits PercentageDiscount, BulkDiscount {
    Money calculateDiscount(Order order);
}

public record PercentageDiscount(double rate) implements DiscountPolicy {
    public Money calculateDiscount(Order order) {
        return order.getTotalPrice().multiply(rate);
    }
}

public record BulkDiscount(int threshold, double discountRate) implements DiscountPolicy {
    public Money calculateDiscount(Order order) {
        return order.getItemCount() > threshold
            ? order.getTotalPrice().multiply(discountRate)
            : Money.ZERO;
    }
}

// Exhaustive switch — permits 목록에 없는 타입 추가 시 컴파일 에러
Money discount = switch (policy) {
    case PercentageDiscount p -> p.calculateDiscount(order);
    case BulkDiscount b -> b.calculateDiscount(order);
};
```

### LSP (Liskov Substitution Principle)

Subtypes must work correctly wherever their base type is used.

**Classic signs of violation:**
- Overriding a method with an empty implementation in a subclass
- Growing number of `instanceof` checks
- Subtype violating preconditions/postconditions of the base type

**Principle**: Favor composition over inheritance. Verify that the "is-a" relationship holds behaviorally, not just structurally.

### ISP (Interface Segregation Principle)

Split bloated interfaces by role.

```java
// Bad: An interface that knows everything
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// Good: Segregated by role
public interface Workable { void work(); }
public interface Feedable { void eat(); }
public interface Restable { void sleep(); }

public class HumanWorker implements Workable, Feedable, Restable { ... }
public class RobotWorker implements Workable { ... }  // Doesn't eat or sleep
```

### DIP (Dependency Inversion Principle)

High-level modules should not depend directly on low-level modules. Both should depend on abstractions.

```java
// Bad: High-level depends directly on low-level
public class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository();
}

// Good: Depends on abstraction + constructor injection
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

---

## 2. Frequently Used Design Patterns in Java

### Creational Patterns

**Builder** — Step-by-step construction of complex objects
- When constructor parameters exceed 4
- When there are many optional parameters
- When building immutable objects

```java
public class User {
    private final String name;        // required
    private final String email;       // required
    private final int age;            // optional
    private final String phone;       // optional

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.phone = builder.phone;
    }

    public static class Builder {
        private final String name;
        private final String email;
        private int age;
        private String phone;

        public Builder(String name, String email) {
            this.name = name;
            this.email = email;
        }

        public Builder age(int age) { this.age = age; return this; }
        public Builder phone(String phone) { this.phone = phone; return this; }
        public User build() { return new User(this); }
    }
}
```

**Java 25: 파라미터가 모두 필수·불변이면 record로 충분**
```java
// 모든 필드가 필수이고 불변이면 Builder 불필요
public record User(String name, String email, int age, String phone) {}

// Builder가 여전히 필요한 경우:
// - optional 파라미터가 많을 때 (null 대신 기본값 세팅)
// - 빌드 시 복잡한 유효성 검증이 필요할 때
// - 단계적(Step Builder) 빌드가 필요할 때
```

**Factory Method** — Delegate object creation logic to subclasses
- When the concrete type to create is determined at runtime
- When creation logic is complex and repeated in multiple places

**Singleton** — Use with great caution
- Creates global state that makes testing difficult
- If needed, prefer enum-based singletons or DI framework scoping

### Structural Patterns

**Adapter** — Connect incompatible interfaces
- Bridge between legacy code and new code
- Fit external libraries to internal interfaces

**Decorator** — Dynamically add functionality
- Extend responsibilities without inheritance
- Java I/O streams are the classic example (`BufferedInputStream`)

**Facade** — Provide a simple interface to a complex subsystem
- Library wrapping, simplifying module boundaries

### Behavioral Patterns

**Strategy** — Encapsulate algorithms to make them interchangeable
- When if-else/switch chains grow long
- When algorithms need to be swapped at runtime

```java
public interface SortStrategy {
    <T extends Comparable<T>> List<T> sort(List<T> items);
}

public class DataProcessor {
    private SortStrategy strategy;

    public DataProcessor(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public <T extends Comparable<T>> List<T> process(List<T> data) {
        return strategy.sort(data);
    }
}
```

**Java 25: sealed interface + pattern matching으로 타입 안전한 전략 분기**
```java
public sealed interface SortStrategy permits QuickSort, MergeSort, TimSort {
    <T extends Comparable<T>> List<T> sort(List<T> items);
}

public record QuickSort() implements SortStrategy { ... }
public record MergeSort() implements SortStrategy { ... }
public record TimSort() implements SortStrategy { ... }

// 패턴 매칭으로 전략별 설명 분기 (컴파일 타임 exhaustiveness 보장)
String description = switch (strategy) {
    case QuickSort q -> "Quick sort: O(n log n) average, in-place";
    case MergeSort m -> "Merge sort: O(n log n) stable";
    case TimSort t -> "Tim sort: hybrid, Python/Java default";
};
```

**Observer** — Automatically notify dependents on state change
- Event-driven systems, UI updates

**Template Method** — Define algorithm skeleton, defer specific steps to subclasses
- When a common workflow has only a few varying steps

---

## 3. Pattern Selection Guide

| Problem Situation | Recommended Pattern |
|-------------------|-------------------|
| Complex object creation or many parameters | Builder, Factory Method |
| Conditionals branching on type | Strategy, State |
| Extend existing code without modification | Decorator, Adapter |
| Multiple objects must react to state changes | Observer |
| Need to simplify a complex API | Facade |
| Same algorithm skeleton, different details | Template Method |
| Traverse an object structure and perform operations | Iterator, Visitor |

**Ask before applying a pattern:**
1. Does this problem truly require a pattern?
2. Is there a simpler solution?
3. Does the team understand this pattern?

---

## Cross-References

- For DDD tactical patterns that operationalize these principles (Aggregate, Repository, Domain Service) → `domain-driven-design.md`
- For Effective Java idioms that implement these patterns in Java (Builder, enum Strategy, interfaces) → `effective-java.md`

---

## 4. Anti-Pattern Avoidance

- **God Object**: A class that knows everything → Decompose via SRP
- **Anemic Domain Model**: Models with only getters/setters → Put behavior in domain objects
- **Premature Optimization**: Optimizing without measuring → Make it correct first, then fast
- **Cargo Cult Programming**: Applying patterns without reason → Understand the problem first
- **Golden Hammer**: Trying to solve every problem with one pattern/technology → Choose the right tool for the problem

**instanceof 체인 → sealed + pattern matching switch**
```java
// Anti-pattern: 타입 체크 체인 (새 타입 추가 시 모든 분기 찾아야 함)
if (shape instanceof Circle) {
    var c = (Circle) shape;
    return Math.PI * c.radius() * c.radius();
} else if (shape instanceof Rectangle) {
    var r = (Rectangle) shape;
    return r.width() * r.height();
}

// Modern Java: sealed + pattern matching switch (exhaustive, 컴파일 타임 안전)
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}
record Triangle(double base, double height) implements Shape {}

double area = switch (shape) {
    case Circle(var r) -> Math.PI * r * r;
    case Rectangle(var w, var h) -> w * h;
    case Triangle(var b, var h) -> 0.5 * b * h;
};
// Shape에 새 permits 타입 추가 시 switch에 컴파일 에러 → 누락 불가
```
