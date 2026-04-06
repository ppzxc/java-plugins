# Java Coding Rules
> Sources: Effective Java, Clean Code, Clean Architecture, Refactoring, GoF, Java Concurrency in Practice, Modern Java in Action, Release It!
> Format: RULE → WHEN → PATTERN → EXCEPTION

---

## 객체 생성

RULE: static factory 메서드 사용
WHEN: 동일 시그니처 생성자가 2개 이상 필요하거나, 의미 있는 이름이 필요할 때
PATTERN: `of()`, `from()`, `valueOf()`, `getInstance()`, `create()`
EXCEPTION: Spring `@Component` 등록 클래스는 `public` 생성자 유지

RULE: Builder 패턴 사용
WHEN: 매개변수가 4개 이상이거나, 선택적 매개변수가 있을 때
PATTERN:
```java
public class Order {
    private final String id;
    private final String item;
    private final int quantity;
    private Order(Builder b) { this.id = b.id; ... }
    public static Builder builder() { return new Builder(); }
    public static class Builder {
        public Builder id(String id) { this.id = id; return this; }
        public Order build() { return new Order(this); }
    }
}
```
EXCEPTION: record 사용 가능한 경우 Builder 불필요

RULE: DTO는 record 사용
WHEN: 데이터 운반용 객체 (Request/Response DTO, 이벤트 페이로드)
PATTERN: `public record CreateOrderRequest(String item, int quantity) {}`
EXCEPTION: JPA `@Entity`, Jackson 커스텀 직렬화가 복잡한 경우

RULE: 불변(immutable) 클래스 설계
WHEN: Value Object, 도메인 값 타입
PATTERN: `final` 클래스, `final` 필드, setter 없음, 방어적 복사
EXCEPTION: 성능상 가변이 필수인 경우

---

## Null 처리

RULE: null 반환 금지
WHEN: 메서드 반환값이 없을 수 있을 때
PATTERN: `Optional<T>` 반환, 빈 컬렉션 반환(`List.of()`)
EXCEPTION: 성능 임계 경로에서 `Optional` 박싱 비용이 문제가 될 때

RULE: Optional 올바른 사용
WHEN: null 가능성이 있는 값 처리
```java
// GOOD
return Optional.ofNullable(user).map(User::getName).orElse("unknown");
// BAD - Optional을 필드로 저장하거나 매개변수로 받지 않는다
private Optional<String> name; // X
void process(Optional<User> user) {} // X
```

---

## 동시성 (Java Concurrency in Practice + JDK 25)

> 상세 규칙은 `jdk25-rules.md` 참조

핵심 원칙:
- I/O 바운드 → Virtual Thread (`newVirtualThreadPerTaskExecutor()`)
- CPU 바운드 → Platform Thread Pool (`newFixedThreadPool(availableProcessors())`)
- 동기화 → `synchronized` 기본 사용 (JDK 24+ JEP 491로 Pinning 해결됨)
- 스레드 로컬 → 애플리케이션 도메인 코드에서 `ScopedValue` 사용 (Preview, --enable-preview 필요)
  단, Spring Security/Logback MDC/트랜잭션 동기화 등 프레임워크 내부는 ThreadLocal 유지
- 공유 가변 상태 → `AtomicXxx`, `ConcurrentHashMap`, 불변 객체

---

## 현대 Java (Modern Java in Action)

RULE: Stream API 올바른 사용
WHEN: 컬렉션 변환, 필터링, 집계
```java
// GOOD - 명확한 의도
List<String> names = orders.stream()
    .filter(o -> o.status() == ACTIVE)
    .map(Order::customerName)
    .distinct()
    .toList(); // JDK 16+, unmodifiable

// BAD - forEach로 상태 변경
orders.stream().forEach(o -> result.add(o.name())); // 사용하지 말 것
```
EXCEPTION: 단순 순회는 향상된 for문이 더 명확

RULE: `Collectors.toList()` 대신 `.toList()` 사용 (JDK 16+)
WHEN: Stream 종단 연산
PATTERN: `stream.toList()` — unmodifiable List 반환

RULE: 함수형 인터페이스 직접 정의 금지
WHEN: 1메서드 함수형 인터페이스 필요 시
PATTERN: `Function<T,R>`, `Predicate<T>`, `Supplier<T>`, `Consumer<T>` 등 표준 사용
EXCEPTION: checked exception이 필요한 경우 커스텀 정의 가능

---

## 예외 처리 (Effective Java)

RULE: 비즈니스 실패는 Sealed Class Result Pattern 반환
WHEN: 예상 가능한 비즈니스 실패 (잔액 부족, 상품 품절, 권한 없음 등)
PATTERN:
```java
sealed interface OrderResult permits OrderResult.Success, OrderResult.Failure {}
record Success(Order order) implements OrderResult {}
sealed interface Failure extends OrderResult permits InsufficientBalance, OutOfStock {}
record InsufficientBalance(Money required, Money actual) implements Failure {}
```
WHY: Exception은 제어 흐름에 사용하지 않는다. 비즈니스 실패는 타입 시스템으로 명시한다.

RULE: 인프라/시스템 에러는 Unchecked Exception 사용
WHEN: DB 접속 실패, 네트워크 오류 등 회복 불가능한 인프라 오류
PATTERN: RuntimeException 서브클래스 throw

RULE: 프로그래밍 오류에는 unchecked exception
WHEN: 전제조건 위반, 버그
PATTERN: `IllegalArgumentException`, `IllegalStateException`, `NullPointerException`

RULE: 예외 메시지에 실패 정보 포함
WHEN: 예외 생성 시
PATTERN: `throw new IllegalArgumentException("quantity must be positive, but was: " + quantity)`

RULE: try-with-resources 사용
WHEN: `Closeable`/`AutoCloseable` 리소스 사용 시
PATTERN:
```java
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement(sql)) {
    // ...
}
```

---

## 아키텍처 (Clean Architecture)

RULE: 의존성 방향 — 안쪽만
WHEN: 레이어 간 의존성 설계
PATTERN: `Controller → Service → Domain/Repository Interface`
- Controller는 Service를 안다
- Service는 Domain과 Repository 인터페이스를 안다
- Domain은 아무것도 모른다
- Repository 구현체는 Service가 아닌 인터페이스에 의존

EXCEPTION: 없음 — 의존성 방향 위반은 허용하지 않는다

RULE: 인터페이스는 사용자(호출자) 패키지에 위치
WHEN: Repository, Port 인터페이스 정의
PATTERN: `domain.repository.OrderRepository` (domain 패키지가 소유), 구현체는 `infrastructure.persistence`

RULE: 도메인 로직은 엔티티/서비스에 — DB/프레임워크 코드 금지
WHEN: 비즈니스 로직 위치 결정
PATTERN: 도메인 규칙은 `Order.cancel()` 처럼 도메인 객체 메서드로
EXCEPTION: 쿼리 최적화를 위해 불가피한 경우 별도 Query Service로 분리

---

## 안정성 패턴 (Release It!)

RULE: 외부 서비스 호출에 Timeout 필수
WHEN: HTTP 클라이언트, DB, 메시지 브로커 등 모든 외부 의존성
PATTERN:
```java
// Spring RestClient
RestClient.builder()
    .requestInterceptor(new TimeoutInterceptor(Duration.ofSeconds(3)))
    .build();
```
EXCEPTION: 없음 — Timeout 없는 외부 호출은 허용하지 않는다

RULE: 외부 서비스에 Circuit Breaker 적용
WHEN: 반복적으로 실패할 수 있는 외부 의존성
PATTERN: Resilience4j `@CircuitBreaker`
```java
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
public PaymentResult charge(Order order) { ... }
private PaymentResult paymentFallback(Order order, Throwable t) {
    return PaymentResult.deferred(order);
}
```

RULE: Bulkhead로 리소스 격리
WHEN: 여러 외부 서비스 중 하나의 장애가 전체를 마비시킬 수 있을 때
PATTERN: 서비스별 독립 스레드 풀 또는 Semaphore

RULE: 입력 검증 — 경계에서 한 번
WHEN: 외부 입력 처리 (HTTP 요청, 메시지 큐)
PATTERN: Controller 레이어에서 `@Valid` 어노테이션, Bean Validation
```java
public record CreateOrderRequest(
    @NotBlank String item,
    @Positive int quantity
) {}
```

---

## 리팩토링 트리거 (Refactoring — Fowler)

다음 상황이면 즉시 리팩토링한다:

RULE: 메서드 추출 (Extract Method)
WHEN: 메서드가 30줄 초과, 또는 주석이 필요한 코드 블록 존재
PATTERN: 주석 내용이 메서드 이름이 된다

RULE: 조건부 다형성으로 교체 (Replace Conditional with Polymorphism)
WHEN: 동일 타입을 기준으로 하는 `if/else` 또는 `switch`가 여러 곳에 반복
PATTERN: 전략 패턴 또는 sealed class + pattern matching

RULE: 임시 변수 → 메서드 추출 (Replace Temp with Query)
WHEN: 임시 변수가 한 번만 쓰이는 계산 결과
PATTERN: 메서드로 추출하여 의미 부여

RULE: 매직 넘버 → 상수 (Replace Magic Number with Symbolic Constant)
WHEN: 의미 없는 숫자 리터럴
PATTERN: `private static final int MAX_RETRY = 3;`

---

## 디자인 패턴 적용 (GoF)

RULE: 알고리즘 교체 가능성 → Strategy
WHEN: 런타임에 알고리즘/동작을 교체해야 할 때
PATTERN:
```java
interface DiscountStrategy { int calculate(Order order); }
class PercentageDiscount implements DiscountStrategy { ... }
class FixedDiscount implements DiscountStrategy { ... }
```

RULE: 동일 객체 타입 반복 생성 → Factory Method / Abstract Factory
WHEN: 조건에 따라 구현체가 달라지는 객체 생성
PATTERN:
```java
interface NotificationFactory { Notification create(String type); }
```

RULE: 이벤트 발행/구독 → Observer
WHEN: 상태 변경을 여러 객체에 알려야 할 때
PATTERN: Spring `ApplicationEventPublisher` 사용
```java
publisher.publishEvent(new OrderPlacedEvent(order.id()));
```

RULE: 래핑으로 기능 추가 → Decorator
WHEN: 상속 없이 기능을 동적으로 추가
PATTERN: 동일 인터페이스를 구현하면서 대상 객체를 감쌈

RULE: 복잡한 서브시스템 단순화 → Facade
WHEN: 여러 서비스/컴포넌트를 조합하는 복잡한 작업
PATTERN: 하나의 단순한 메서드로 복잡도를 숨김

---

## 네이밍 (Clean Code)

RULE: 의도를 드러내는 이름
```
// BAD
int d; List<int[]> list1;
// GOOD
int elapsedTimeInDays; List<Cell> flaggedCells;
```

RULE: 불리언 변수/메서드는 `is`/`has`/`can` 접두사
PATTERN: `isActive`, `hasPermission`, `canCancel()`

RULE: 메서드 이름은 동사+명사
PATTERN: `createOrder()`, `findByEmail()`, `calculateTotal()`

RULE: 매개변수 2개 이하가 이상적, 3개부터 객체로 묶기
WHEN: 메서드 매개변수가 3개 이상
PATTERN: 파라미터 객체 도입 (Introduce Parameter Object)
