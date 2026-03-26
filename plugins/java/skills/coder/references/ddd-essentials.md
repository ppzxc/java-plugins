# DDD Essentials
> Sources: Domain-Driven Design (Eric Evans), Implementing DDD (Vaughn Vernon)
> 전술적 DDD 핵심 패턴만 포함. 전략적 DDD(Bounded Context, Context Map)는 제외.
> Format: RULE → WHEN → PATTERN → EXCEPTION

---

## Entity

RULE: Entity는 식별자(ID)로 동일성 판단
WHEN: `equals()`/`hashCode()` 구현
PATTERN:
```java
@Entity
public class Order {
    @Id
    private final UUID id;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Order other)) return false;
        return id.equals(other.id);
    }

    @Override
    public int hashCode() { return id.hashCode(); }
}
```
EXCEPTION: 없음 — 비즈니스 속성으로 equals를 구현하지 않는다

RULE: Entity 생성 시 ID를 도메인에서 생성 (DB 자동 생성 지양)
WHEN: 신규 Entity 생성
PATTERN: `UUID.randomUUID()` 또는 도메인 고유 ID 생성기 사용
EXCEPTION: 외부 시스템 연동, 레거시 DB 스키마

---

## Value Object

RULE: 값만 의미 있는 객체는 Value Object
WHEN: 주소, 금액, 좌표, 이름 등 값 자체가 동일성인 경우
PATTERN:
```java
// record로 구현 (JDK 16+)
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("amount must not be negative");
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
}
```
EXCEPTION: JPA 매핑 시 `@Embeddable` + class 사용 (record JPA 미지원)

RULE: Value Object는 불변
WHEN: 값을 "변경"해야 할 때
PATTERN: 새 인스턴스를 반환한다 (수정하지 않는다)
```java
// GOOD
Money discounted = price.subtract(discount);
// BAD
price.setAmount(price.amount().subtract(discount)); // X
```

---

## Aggregate

RULE: Aggregate는 일관성 경계를 정의
WHEN: 함께 변경되어야 하는 객체 묶음
PATTERN:
- Aggregate Root만 외부에서 참조 가능
- 내부 Entity는 Root를 통해서만 접근
- 하나의 트랜잭션 = 하나의 Aggregate

```java
public class Order {  // Aggregate Root
    private final UUID id;
    private OrderStatus status;
    private final List<OrderLine> lines;  // 내부 Entity

    public void addLine(Product product, int qty) {
        // 비즈니스 규칙 검증은 Root에서
        if (status != OrderStatus.DRAFT)
            throw new IllegalStateException("Cannot modify confirmed order");
        lines.add(new OrderLine(product, qty));
    }

    public void confirm() {
        if (lines.isEmpty()) throw new IllegalStateException("No items");
        this.status = OrderStatus.CONFIRMED;
        // 도메인 이벤트 발행
        registerEvent(new OrderConfirmedEvent(id));
    }
}
```

RULE: Aggregate 간 참조는 ID로만
WHEN: 다른 Aggregate를 참조할 때
PATTERN:
```java
// GOOD
public record OrderLine(UUID productId, int quantity, Money price) {}
// BAD
public class OrderLine { Product product; }  // Product Aggregate 직접 참조 X
```

RULE: Aggregate 크기는 작게 유지
WHEN: Aggregate 설계 시
PATTERN: 하나의 Aggregate에 Entity가 5개 이상이면 분리를 고려
EXCEPTION: 일관성 규칙이 반드시 함께 유지되어야 하는 경우

---

## Domain Event

RULE: 상태 변경 후 Domain Event 발행
WHEN: Aggregate의 중요한 상태 변화 (주문 확정, 결제 완료 등)
PATTERN:
```java
// Spring의 AbstractAggregateRoot 사용
public class Order extends AbstractAggregateRoot<Order> {
    public void confirm() {
        this.status = CONFIRMED;
        registerEvent(new OrderConfirmedEvent(this.id, LocalDateTime.now()));
    }
}

// 이벤트 핸들러
@EventListener
public void on(OrderConfirmedEvent event) {
    emailService.sendConfirmation(event.orderId());
}
```
EXCEPTION: 단순 CRUD 작업

---

## Repository

RULE: Repository 인터페이스는 도메인 레이어에
WHEN: 영속성 추상화
PATTERN:
```java
// domain/repository 패키지
public interface OrderRepository {
    Order findById(UUID id);
    void save(Order order);
    List<Order> findByCustomerId(UUID customerId);
}

// infrastructure/persistence 패키지
@Repository
public class JpaOrderRepository implements OrderRepository { ... }
```

RULE: Repository는 Aggregate 단위로 — 내부 Entity 개별 Repository 금지
WHEN: Repository 정의
PATTERN: `OrderRepository`만, `OrderLineRepository`는 만들지 않는다
EXCEPTION: 독립적인 Aggregate로 분리가 필요한 경우

---

## Domain Service

RULE: 여러 Aggregate에 걸친 비즈니스 로직 → Domain Service
WHEN: 하나의 Entity에 넣기 어색한 도메인 로직
PATTERN:
```java
// 두 계좌 간 이체 — Account 하나에 넣기 어색
public class TransferService {
    public void transfer(Account from, Account to, Money amount) {
        from.debit(amount);
        to.credit(amount);
    }
}
```
EXCEPTION: Application Service와 혼동하지 말 것 — Domain Service는 인프라 의존성 없음
