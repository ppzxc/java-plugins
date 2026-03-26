# Java Code Review Checklist
> Sources: Effective Java, Clean Code, Clean Architecture, Refactoring
> 리뷰 관점의 체크리스트. "이것이 있으면 지적, 이것이 없으면 제안"
> Format: CHECK → WHY → FIX

---

## 동시성 안전성 (Critical)

CHECK: `synchronized` 블록/메서드 사용
WHY: Virtual Thread 환경에서 carrier thread pin 발생, 성능 저하
FIX: `ReentrantLock` 교체
```java
// FIX
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* critical section */ } finally { lock.unlock(); }
```

CHECK: `ThreadLocal` 사용
WHY: Virtual Thread 환경에서 누수 위험, ScopedValue가 더 안전
FIX: `ScopedValue` 마이그레이션 검토

CHECK: 공유 가변 상태 (non-final 필드를 여러 스레드가 접근)
WHY: 데이터 경쟁 조건
FIX: `final` 필드, `AtomicXxx`, `ConcurrentHashMap`, 불변 객체

---

## Null 안전성 (Critical)

CHECK: 메서드가 `null` 반환
WHY: `NullPointerException` 위험, 호출자가 null 체크를 강요받음
FIX: `Optional<T>` 반환, 빈 컬렉션 반환

CHECK: `Optional.get()` 호출 (isPresent 체크 없이)
WHY: `NoSuchElementException` 위험
FIX: `orElse()`, `orElseGet()`, `orElseThrow()`, `ifPresent()` 사용

CHECK: `Optional`을 필드/매개변수로 사용
WHY: Optional은 반환 타입으로만 설계됨
FIX: 필드는 `null` 허용 + 어노테이션, 매개변수는 오버로딩으로 대체

---

## 객체 설계 (Effective Java)

CHECK: DTO가 일반 class로 구현됨 (record 미사용)
WHY: record가 더 간결하고 불변 보장
FIX: `record`로 교체 (JPA Entity, 가변 상태 필요한 경우 제외)

CHECK: 생성자 매개변수가 4개 이상
WHY: 호출자가 인수 순서를 기억해야 함, 오류 가능성 높음
FIX: Builder 패턴 도입

CHECK: `equals()` 오버라이드 시 `hashCode()` 미오버라이드
WHY: HashSet, HashMap에서 동작 오류
FIX: 항상 함께 오버라이드

CHECK: `equals()`가 가변 필드로 구현됨
WHY: HashMap 키, HashSet 원소로 사용 시 버그
FIX: 불변 필드(주로 ID)로만 구현

CHECK: `clone()` 오버라이드 사용
WHY: 깊은 복사 계약 이행이 어렵고 버그 유발
FIX: 복사 생성자 또는 static factory `copy(original)` 사용

---

## 아키텍처 (Clean Architecture)

CHECK: Controller/API 계층에 비즈니스 로직
WHY: 재사용 불가, 테스트 어려움, 계층 분리 위반
FIX: Service 레이어로 이동

CHECK: Service가 다른 Service를 직접 의존 (순환 의존 발생)
WHY: 결합도 증가, 변경 영향 범위 확대
FIX: 이벤트 기반 통신 또는 중간 레이어 추출

CHECK: Domain Entity에 `@Autowired` / Spring 어노테이션
WHY: 도메인이 프레임워크에 의존 — 아키텍처 오염
FIX: 도메인 객체에서 Spring 의존성 제거, Application Service에서 처리

CHECK: Repository 구현체가 도메인 레이어에 위치
WHY: 인프라 코드가 도메인을 오염
FIX: Repository 인터페이스는 도메인에, 구현체는 infrastructure 패키지로

CHECK: Aggregate 내부 Entity에 직접 접근
WHY: Aggregate 일관성 경계 위반
FIX: Aggregate Root 메서드를 통해서만 접근

---

## 코드 품질 (Clean Code)

CHECK: 메서드 길이가 30줄 초과
WHY: 단일 책임 위반, 이해 어려움
FIX: Extract Method 리팩토링

CHECK: 중첩 if/else가 3단계 초과 (Arrow Anti-pattern)
WHY: 가독성 저하
FIX: Guard Clause (Early Return), 조건 로직 추출

CHECK: 매직 넘버/문자열 리터럴
WHY: 의미 불명확, 수정 시 누락 위험
FIX: `private static final` 상수로 추출

CHECK: 주석이 "무엇"을 설명 (코드로 표현 가능한 내용)
WHY: 코드와 주석이 불일치할 수 있음
FIX: 의미 있는 이름으로 코드 자체가 설명하도록. 주석은 "왜"만 허용

CHECK: 같은 if/switch 조건이 여러 곳에 반복
WHY: Open/Closed Principle 위반, 새 타입 추가 시 모두 수정해야 함
FIX: Strategy 패턴, Polymorphism, Sealed class + pattern matching

---

## 안정성 (Release It!)

CHECK: 외부 서비스 호출에 Timeout 미설정
WHY: 외부 서비스 지연 시 스레드 무한 점유, 연쇄 장애
FIX: 모든 외부 HTTP/DB/메시지 호출에 Timeout 필수

CHECK: 외부 서비스 장애 시 Fallback 없음
WHY: 부분 장애가 전체 장애로 확산
FIX: Circuit Breaker + Fallback 메서드 추가

CHECK: 입력 검증이 없거나 Service 레이어에서만 수행
WHY: Controller에서 먼저 차단해야 불필요한 처리 방지
FIX: Controller에서 `@Valid` + Bean Validation 적용

CHECK: Redis/캐시 장애 시 동작 미정의
WHY: 캐시 장애가 전체 서비스 중단으로 이어짐
FIX: try-catch로 캐시 장애 시 DB 폴백 구현

---

## 리팩토링 제안 트리거 (Refactoring — Fowler)

다음이 보이면 리팩토링을 제안한다:

| Code Smell | 리팩토링 기법 |
|-----------|-------------|
| 긴 메서드 (30줄+) | Extract Method |
| 긴 매개변수 목록 (4개+) | Introduce Parameter Object |
| 중복 코드 (동일 로직 2곳+) | Extract Method, Pull Up Method |
| 데이터 클래스 (getter/setter만) | Move Method, 도메인 로직 이동 |
| 타입에 따른 반복 switch/if | Replace Conditional with Polymorphism |
| 임시 변수 (한 번만 사용) | Inline Variable, Replace Temp with Query |
| 주석이 없으면 이해 불가 | Extract Method (주석 내용 = 메서드 이름) |

---

## Effective Java 추가 체크

CHECK: `Comparable` 구현 시 `equals`와 일관성 없음
WHY: `TreeSet`, `TreeMap` 동작 오류
FIX: `compareTo` 결과가 0 ↔ `equals` 결과가 true

CHECK: 제네릭 타입에 raw type 사용 (`List` 대신 `List<String>`)
WHY: 컴파일 타임 타입 안전성 상실
FIX: 타입 매개변수 명시

CHECK: 가변 args (`varargs`)와 제네릭 함께 사용
WHY: heap pollution 경고, 안전하지 않을 수 있음
FIX: `@SafeVarargs` 추가하거나 `List` 매개변수로 대체

CHECK: 문자열 연결 (`+`)을 반복문 안에서 사용
WHY: O(n²) 성능
FIX: `StringBuilder` 또는 `String.join()`, Stream `Collectors.joining()`
