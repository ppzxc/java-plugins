---
description: >
  Use when writing, designing, or refactoring Java code. Trigger for .java file
  creation or modification, domain modeling, concurrency code, or API design.
  NOT for test file writing (java:tester), code review (java:reviewer), or
  Spring Boot patterns (java:spring).
user-invocable: true
version: 0.0.5
---

# java:coder

## Trigger
- `.java` 파일 생성 또는 수정
- 도메인 모델링 (Entity, Value Object, Aggregate 설계)
- 동시성 코드 작성
- API 설계 (Spring Controller 제외 → `java:spring`)
- 리팩토링 작업

## Decision Tree

### 새 클래스를 만들어야 한다면?

```
어떤 목적의 클래스인가?
│
├─ 데이터 운반 (DTO, Request, Response, Event)
│   → record 사용
│   → 검증 로직은 compact constructor에
│
├─ 도메인 Entity (ID로 구분되는 비즈니스 객체)
│   → class 사용 (record 불가 — 상태 변경 필요)
│   → ID 기반 equals/hashCode
│   → 도메인 로직을 메서드로
│
├─ 도메인 Value Object (값 자체가 동일성)
│   → record 사용 (Money, Address, Coordinate 등)
│   → 불변 유지, 변경 시 새 인스턴스 반환
│
├─ 서비스 (비즈니스 로직 조율)
│   → class, 생성자 주입
│   → @Transactional 고려 (Spring → java:spring)
│
└─ 유틸리티 (stateless 헬퍼)
    → final class + private 생성자 + static 메서드
```

### 생성자 vs static factory vs Builder?

```
매개변수가 몇 개인가?
│
├─ 1~2개이고 타입이 명확
│   → 생성자 또는 static factory (of(a, b))
│
├─ 3개이고 의미가 명확
│   → static factory 권장 (Money.of(amount, currency))
│
└─ 4개 이상 또는 선택적 매개변수 있음
    → 수동 static inner Builder 또는 정적 팩토리 메서드 필수 (Lombok @Builder 전면 금지)
```

### 동시성 코드를 작성한다면?

```
작업 유형은?
│
├─ I/O 바운드 (HTTP 호출, DB 쿼리, 파일 I/O)
│   → Virtual Thread: Executors.newVirtualThreadPerTaskExecutor()
│   → synchronized 허용 (JDK 24+ Pinning 해결됨)
│   → Timeout 등 고급 제어 필요시에만 ReentrantLock
│   → 애플리케이션 코드에서 ThreadLocal 금지 → ScopedValue (JDK 24+ 확정)
│   → Spring Security/MDC 등 프레임워크 내부 ThreadLocal은 건드리지 말 것
│
├─ CPU 바운드 (계산, 이미지 처리)
│   → Platform Thread Pool
│   → Executors.newFixedThreadPool(availableProcessors())
│
└─ 여러 작업을 함께 관리
    → StructuredTaskScope 사용
    → ShutdownOnFailure: 모두 성공해야 할 때
    → ShutdownOnSuccess: 하나만 성공하면 될 때
```

### 에러/null 처리는?

```
작업이 실패하거나 반환값이 없을 수 있다면?
│
├─ 예상 가능한 비즈니스 실패 (잔액 부족 등)
│   → Sealed Class 기반 Result Pattern 반환 (Exception 금지)
├─ 인프라/시스템 에러 (DB 접속 실패 등)
│   → throw Exception (Unchecked)
├─ 단순 단일 값 부재
│   → Optional<T> 반환
└─ 컬렉션 부재
    → 빈 컬렉션 반환 (List.of())
```

### 외부 서비스를 호출한다면?

```
외부(third-party/외부 시스템) 호출 필수 체크:
├─ Timeout 설정 (내부 서비스 포함 모든 외부 호출)
├─ Circuit Breaker 적용 (third-party API, 외부 SaaS 등)
├─ Fallback 구현 (Circuit Breaker 적용 시)
└─ Retry 정책 (멱등성 확인 후, exponential backoff 적용)

내부 마이크로서비스 호출: Timeout 필수, Circuit Breaker는 선택적

이 중 하나라도 없으면 추가한다.
```

### 어떤 패턴을 써야 할지 모르겠다면?

```
상황에 따라:
│
├─ 알고리즘/동작을 런타임에 교체
│   → Strategy 패턴
│
├─ 조건에 따라 다른 구현체 생성
│   → Factory Method 패턴
│
├─ 상태 변경을 여러 객체에 알려야
│   → Observer (Spring: ApplicationEventPublisher)
│
├─ 복잡한 하위 시스템을 단순화
│   → Facade 패턴
│
└─ 타입 계층이 닫혀 있고 exhaustive 처리 필요
    → Sealed class + pattern matching switch
```

## Quick Rules

DO:
- DTO → `record`
- 도메인 에러 → `Sealed Class (Result Pattern)` 반환
- 동시성 제어 → `synchronized` 기본 사용 (단, Timeout 필요시 `ReentrantLock`)
- 애플리케이션 코드의 스레드 로컬 → `ScopedValue` (ThreadLocal 금지, 프레임워크 내부 제외)
- null 반환 금지 → `Optional` 또는 빈 컬렉션
- 의존성은 안쪽 방향 (Controller → Service → Domain)
- 외부 호출 → Timeout + Circuit Breaker + Fallback 필수

DON'T:
- 도메인 모델에 Lombok (`@Builder`, `@Data` 등) 사용
- 비즈니스 규칙 위반을 Exception으로 던짐
- Domain Entity에 비즈니스 무관한 프레임워크 코드
- 같은 타입 조건 if/switch를 여러 곳에 반복
- 메서드 30줄 초과
- 외부 호출에 Timeout 없음
- null 반환

## Delegation

| 상황 | 위임 스킬 |
|------|----------|
| 테스트 코드 작성 | `java:tester` |
| 코드 리뷰 요청 | `java:reviewer` |
| Spring @Service (트랜잭션/DI 관련) / Controller / Repository / Spring 테스트 | `java:spring` |
| 순수 도메인 Service (인프라 의존 없음) | `java:coder` 유지 |

## References
- `references/coding-rules.md` — 객체 생성, 동시성, 예외/Result, 아키텍처, 안정성, 패턴
- `references/jdk25-rules.md` — JDK 25 기능 (record, sealed, pattern matching, virtual thread 등)
- `references/ddd-essentials.md` — DDD 핵심 (Entity, Value Object, Aggregate, Repository)