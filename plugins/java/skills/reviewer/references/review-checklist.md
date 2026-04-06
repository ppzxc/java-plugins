# Java Code Review Checklist
> Sources: Effective Java, Clean Code, Clean Architecture, Refactoring, DDD
> 리뷰 관점의 체크리스트. "이것이 있으면 지적, 이것이 없으면 제안"
> Format: CHECK → WHY → FIX

---

## 동시성 안전성 (Critical)
> 상세 규칙 → `jdk25-rules.md` 참조

CHECK: `ThreadLocal` 사용 (애플리케이션 도메인 코드)
WHY: Virtual Thread 환경에서 누수 위험
FIX: `ScopedValue` 마이그레이션 권장 (JDK 24+ 확정)
NOTE: 프레임워크 내부(Spring Security, Logback MDC 등) ThreadLocal은 정상 — 건드리지 말 것

CHECK: 공유 가변 상태 (non-final 필드를 여러 스레드가 접근)
WHY: 데이터 경쟁 조건
FIX: `final` 필드, `AtomicXxx`, `ConcurrentHashMap`, 불변 객체

CHECK: Virtual Thread 환경에서 무거운 Native/JNI 호출 in `synchronized`
WHY: JEP 491 이후에도 Native 호출 시에는 여전히 Carrier Thread Pinning 발생 가능
FIX: `ReentrantLock` 교체 검토

---

## Null & 에러 안전성 (Critical)

CHECK: 비즈니스 실패에 Exception 사용 (예: 잔액 부족 등)
WHY: 제어 흐름을 숨기고 컴파일 타임에 처리를 강제할 수 없음 (VIBE 코딩 위반)
FIX: `Sealed Interface` 기반 **Result Pattern** 반환으로 교체

CHECK: 메서드가 `null` 반환
WHY: `NullPointerException` 위험, 호출자가 null 체크를 강요받음
FIX: `Optional<T>` 반환, 빈 컬렉션 반환

CHECK: `Optional.get()` 호출 (isPresent 체크 없이)
WHY: `NoSuchElementException` 위험
FIX: `orElse()`, `orElseGet()`, `orElseThrow()`, `ifPresent()` 사용

---

## 객체 설계 및 DDD (Strict DDD)

CHECK: 도메인 계층(Entity, VO, Aggregate)에 Lombok 어노테이션 사용
WHY: 캡슐화 파괴, 불완전한 객체 생성 허용, 도메인 순수성 오염
FIX: Lombok 제거, 수동 생성자/내부 Builder/팩토리 메서드 구현

CHECK: 원시 타입 집착 (Primitive Obsession)
WHY: 비즈니스 의미 누락, 잘못된 값 할당 위험
FIX: `record` 기반 Value Object(VO) 도입 (Email, Money 등)

CHECK: 빈약한 도메인 모델 (Anemic Domain Model)
WHY: 비즈니스 로직이 서비스로 유출되어 객체 지향 원칙 위반
FIX: 데이터 조작 로직을 도메인 객체 내부 메서드(`order.cancel()` 등)로 이동

CHECK: DTO가 일반 class로 구현됨 (record 미사용)
WHY: record가 더 간결하고 불변 보장
FIX: `record`로 교체 (→ `coding-rules.md` DTO 규칙 참조)

CHECK: 생성자 매개변수가 4개 이상인데 Builder 미사용
WHY: 호출자 실수 가능성 높음
FIX: 수동 static inner Builder 도입 (Lombok 금지)

---

## 아키텍처 (Clean Architecture)

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

---

## 안정성 (Release It!)
> 상세 규칙 → `coding-rules.md` 참조

CHECK: 외부 서비스 호출에 Timeout 미설정
WHY: 외부 서비스 지연 시 스레드 무한 점유, 연쇄 장애
FIX: 모든 외부 HTTP/DB/메시지 호출에 Timeout 필수

CHECK: third-party API/외부 SaaS 호출에 Circuit Breaker 없음
WHY: 부분 장애가 전체 장애로 확산
FIX: Circuit Breaker + Fallback 메서드 추가 (내부 마이크로서비스는 Timeout만 필수)

---

## Effective Java 추가 체크

CHECK: `equals()` 오버라이드 시 `hashCode()` 미오버라이드
WHY: HashSet, HashMap에서 동작 오류
FIX: 항상 함께 오버라이드

CHECK: 제네릭 타입에 raw type 사용 (`List` 대신 `List<String>`)
WHY: 컴파일 타임 타입 안전성 상실
FIX: 타입 매개변수 명시

CHECK: 문자열 연결 (`+`)을 반복문 안에서 사용
WHY: O(n²) 성능
FIX: `StringBuilder` 또는 `String.join()`, Stream `Collectors.joining()`
