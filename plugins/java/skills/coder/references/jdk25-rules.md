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

RULE: Virtual Thread에서 `synchronized` 사용 기본 (JDK 24+ Pinning 해결됨)
WHEN: Virtual Thread 환경
WHY: JEP 491을 통해 `synchronized` 블록 내부에서 블로킹 작업이 발생해도 더 이상 carrier thread를 pin하지 않음. 코드가 간결해짐.
PATTERN: 순수 자바 키워드 사용
```java
public synchronized void process() { ... }
```
EXCEPTION: Timeout 설정이나 Condition 제어 등 고도화된 락 제어가 필요한 경우에만 `ReentrantLock` 사용

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
EXCEPTION: 프레임워크 내부(Spring Security SecurityContext, Logback MDC, 트랜잭션 동기화 매니저 등)는
ThreadLocal을 계속 사용한다. 애플리케이션 도메인 코드에서만 ScopedValue로 전환하라.

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

## Structured Concurrency (JDK 25 Preview — JEP 505, --enable-preview 필요)

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

> NOTE: ScopedValue는 JDK 24에서 확정된 기능이다 (JEP 487). 별도 플래그 불필요.
> StructuredTaskScope는 JDK 25에서도 Preview 상태이므로 `--enable-preview` 필요 (JEP 505).

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

## Stable Values (JDK 25 Preview — JEP 502, --enable-preview 필요)

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

---

## Stream Gatherers (JDK 25 — JEP 485)

RULE: 커스텀 중간 연산이 필요할 때 Stream Gatherer 사용
WHEN: 기존 Stream API(map, filter, flatMap 등)로 표현하기 어려운 상태 기반 중간 연산
PATTERN:
```java
// 슬라이딩 윈도우 (내장 Gatherer 활용)
List<List<Integer>> windows = IntStream.rangeClosed(1, 10)
    .boxed()
    .gather(Gatherers.windowSliding(3))
    .toList();

// 커스텀 Gatherer: 누적 합계
Gatherer<Integer, ?, Integer> runningTotal = Gatherer.of(
    () -> new int[]{0},
    (state, element, downstream) -> {
        state[0] += element;
        return downstream.push(state[0]);
    }
);

List<Integer> totals = List.of(1, 2, 3, 4, 5)
    .stream()
    .gather(runningTotal)
    .toList();
```
EXCEPTION: 단순 변환/필터는 기존 map/filter 사용 (Gatherer는 상태가 필요할 때만)

---

## Flexible Constructor Bodies (JDK 25 — JEP 513)

RULE: 생성자에서 super() 호출 전 코드 실행이 필요할 때 활용
WHEN: 부모 클래스 생성자 호출 전 인수 검증, 변환 또는 준비 작업이 필요한 경우
PATTERN:
```java
public class PositiveNumber extends Number {
    public PositiveNumber(int value) {
        // JDK 25 이전: super() 호출 전 코드 불가
        // JDK 25 이후: super() 호출 전 변수 선언 및 검증 가능
        if (value <= 0) throw new IllegalArgumentException("must be positive: " + value);
        super(value);
    }
}
```
EXCEPTION: 부모 생성자 인수가 단순한 경우 — 이전 방식(super 먼저) 유지가 더 명확
