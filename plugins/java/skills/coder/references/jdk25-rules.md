# JDK 25 Rules
> Java 25 LTS 기준. Virtual Thread 안전성과 신규 언어 기능 적용 규칙.
> Format: RULE → WHEN → PATTERN → EXCEPTION

---

## Records

RULE: 데이터 운반 객체는 record 사용
WHEN: DTO, 이벤트 페이로드, 설정값 홀더
PATTERN:
```java
public record Money(BigDecimal amount, Currency currency) {
    // 컴팩트 생성자로 검증
    public Money {
        Objects.requireNonNull(amount);
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("amount must be non-negative");
    }
}
```
EXCEPTION: JPA `@Entity` (프록시 생성 불가), 가변 상태 필요, 상속 필요

RULE: record는 상속 불가 — 공통 동작은 인터페이스로
PATTERN:
```java
interface Identifiable { UUID id(); }
public record Order(UUID id, String item) implements Identifiable {}
```

---

## Sealed Classes + Pattern Matching

RULE: 닫힌 타입 계층에 sealed class 사용
WHEN: 정해진 수의 서브타입만 존재하는 계층 (Result, 도메인 상태 등)
PATTERN:
```java
public sealed interface PaymentResult
    permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {
    record Success(String transactionId) implements PaymentResult {}
    record Failure(String reason) implements PaymentResult {}
    record Pending(String referenceId) implements PaymentResult {}
}
```

RULE: sealed class는 반드시 exhaustive switch로 처리
WHEN: sealed 타입을 분기 처리할 때
PATTERN:
```java
String message = switch (result) {
    case PaymentResult.Success s -> "결제 완료: " + s.transactionId();
    case PaymentResult.Failure f -> "결제 실패: " + f.reason();
    case PaymentResult.Pending p -> "처리 중: " + p.referenceId();
    // default 불필요 — 컴파일러가 exhaustiveness 보장
};
```
EXCEPTION: 없음 — default 추가는 새 서브타입 추가 시 컴파일 오류를 숨긴다

---

## Pattern Matching

RULE: `instanceof` 타입 확인 시 pattern matching 사용
WHEN: 타입 캐스트가 필요한 모든 경우
PATTERN:
```java
// GOOD
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}
// BAD
if (shape instanceof Circle) {
    Circle c = (Circle) shape; // 불필요한 캐스트
}
```

RULE: switch pattern matching으로 복잡한 분기 단순화
WHEN: 여러 타입에 따른 분기
PATTERN:
```java
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> t.base() * t.height() / 2;
};
```

RULE: Guarded patterns으로 조건 추가
PATTERN:
```java
String category = switch (order) {
    case Order o when o.total() > 100_000 -> "VIP";
    case Order o when o.total() > 10_000  -> "Premium";
    case Order o                           -> "Standard";
};
```

---

## Virtual Threads (Project Loom)

RULE: Virtual Thread에서 `synchronized` 블록 사용 금지
WHEN: Virtual Thread 환경 (Spring Boot 3.2+ with virtual threads enabled)
WHY: `synchronized`는 carrier thread를 pin하여 Virtual Thread의 확장성을 무효화
PATTERN: `ReentrantLock` 으로 교체
```java
// BAD — Virtual Thread pin 발생
synchronized (this) { ... }
// GOOD
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { ... } finally { lock.unlock(); }
```
EXCEPTION: JVM 내부 라이브러리 코드 (직접 제어 불가)

RULE: Virtual Thread에서 `ThreadLocal` 사용 금지
WHEN: Virtual Thread 환경
WHY: 수백만 개의 Virtual Thread가 생성될 수 있어 `ThreadLocal` 누수 위험
PATTERN: `ScopedValue` 사용
```java
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

// 설정
ScopedValue.where(CONTEXT, new RequestContext(userId))
           .run(() -> handleRequest());

// 읽기
RequestContext ctx = CONTEXT.get();
```

RULE: Virtual Thread는 I/O 바운드 작업에만 사용
WHEN: 작업 유형 구분
PATTERN:
```java
// I/O 바운드: Virtual Thread
var ioExecutor = Executors.newVirtualThreadPerTaskExecutor();

// CPU 바운드: Platform Thread Pool
var cpuExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors());
```

---

## Structured Concurrency (JDK 21+)

RULE: 관련 작업 묶음에 StructuredTaskScope 사용
WHEN: 여러 비동기 작업을 함께 관리해야 할 때
PATTERN:
```java
// 모두 성공해야 할 때
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask  = scope.fork(() -> fetchUser(userId));
    var orderTask = scope.fork(() -> fetchOrders(userId));
    scope.join().throwIfFailed();
    return new UserDashboard(userTask.get(), orderTask.get());
}

// 하나만 성공하면 될 때
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<Response>()) {
    scope.fork(() -> callPrimary());
    scope.fork(() -> callFallback());
    return scope.join().result();
}
```

---

## ScopedValue

RULE: 요청 컨텍스트 전달에 ScopedValue 사용
WHEN: HTTP 요청 컨텍스트, 트랜잭션 ID, 사용자 정보를 호출 체인에 전달
PATTERN:
```java
public class RequestContextHolder {
    public static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();
    public static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
}

// 필터/인터셉터에서 설정
ScopedValue.where(TRACE_ID, UUID.randomUUID().toString())
           .where(CURRENT_USER, authenticatedUser)
           .run(() -> chain.doFilter(request, response));

// 서비스에서 읽기
String traceId = RequestContextHolder.TRACE_ID.get();
```

---

## Text Blocks

RULE: 멀티라인 문자열에 Text Block 사용 (JDK 15+)
WHEN: JSON, SQL, HTML 등 멀티라인 문자열
PATTERN:
```java
String query = """
        SELECT u.id, u.name
        FROM users u
        WHERE u.active = true
          AND u.created_at > :since
        """;

String json = """
        {
          "name": "%s",
          "amount": %d
        }
        """.formatted(name, amount);
```

---

## Sequenced Collections (JDK 21+)

RULE: 순서가 있는 컬렉션 첫/마지막 원소 접근
WHEN: `List`, `LinkedHashSet`, `LinkedHashMap` 등
PATTERN:
```java
list.getFirst(); // 0번 인덱스
list.getLast();  // 마지막 인덱스
list.reversed(); // 역순 뷰 (새 컬렉션 아님)
```
EXCEPTION: 없음 — `list.get(0)`, `list.get(list.size()-1)` 대신 사용

---

## Stable Values (JDK 25)

RULE: 지연 초기화 싱글톤에 StableValue 사용
WHEN: 초기화 비용이 높고 지연이 필요한 불변 값
PATTERN:
```java
private final StableValue<Config> config = StableValue.of();

public Config getConfig() {
    return config.orElseSet(this::loadConfig);
}
```
EXCEPTION: 스프링 빈은 컨테이너가 관리하므로 불필요
