---
description: >
  Use when reviewing Java code, auditing code quality, or checking architecture
  compliance. Trigger on explicit review requests, PR review commands (/review,
  "review this code"), or multi-file quality audits. NOT for generating new code
  or writing tests.
user-invocable: true
version: 0.0.3
---

# java:reviewer

## Trigger
- 명시적 리뷰 요청 ("리뷰해줘", "review this", `/review`)
- PR 코드 리뷰
- 아키텍처 준수 감사
- 코드 품질 점검

## Decision Tree

### 리뷰 절차

```
1. Scan — 전체 파일 훑기
   → 구조 파악, 레이어 위치, 의존 방향 확인

2. Critical Issues 먼저 — 즉시 수정 필요
   → 도메인 내 Lombok 사용 금지 (@Builder, @Data 등)
   → 동시성 안전성 (ScopedValue, synchronized Pinning 여부)
   → Null/에러 안전성 (비즈니스 에러에 Exception 사용 여부 -> Result 패턴 권장)
   → 아키텍처 위반 (의존성 방향, 도메인 오염, 빈약한 도메인 모델)
   → 외부 호출에 Timeout/Circuit Breaker 없음

3. Major Issues — 수정 권장
   → 원시 타입 집착 (Primitive Obsession)
   → Effective Java 위반 (equals/hashCode, static factory 등)
   → 코드 품질 (메서드 길이, 중복 코드, 매직 넘버)
   → 리팩토링 필요 Code Smell

4. Minor Issues — 제안
   → 네이밍 개선
   → 불필요한 복잡도

5. 카테고리별 피드백 작성
   → Critical → Major → Minor 순서
   → 각 지적에 수정 방법 제시
```

### 동시성 코드가 있다면?

```
확인 항목:
├─ synchronized 블록/메서드 사용?
│   → OK (JDK 24+ Pinning 해결됨). 단, Timeout 필요 시 ReentrantLock 제안.
│
├─ ThreadLocal 사용?
│   → Critical: ScopedValue 마이그레이션 필수
│
├─ 공유 가변 필드 (non-final)?
│   → Critical: AtomicXxx, ConcurrentHashMap, 불변화
│
└─ Virtual Thread 환경에서 무거운 Native/JNI 호출 in synchronized?
    → Critical: Pinning 위험 여전함. ReentrantLock 교체.
```

### 아키텍처를 확인한다면?

```
의존성 방향 확인:
Controller → Service → Domain ← Repository Interface
                         ↑
              Infrastructure (Repository 구현체)

위반 패턴:
├─ 도메인 레이어에 Lombok 사용 → Critical: 캡슐화 위반, 직접 구현/Record 전환
├─ 비즈니스 실패에 Exception 던짐 → Critical: Result 패턴(Sealed Class) 전환
├─ 빈약한 도메인 모델 (Anemic Model) → Critical: 비즈니스 로직을 도메인 객체로 이동
├─ Service가 다른 Service를 직접 의존 (순환) → 이벤트/레이어 분리
├─ Repository 구현체가 domain 패키지에 → infrastructure로 이동
└─ Aggregate 내부 Entity를 외부에서 직접 수정 → Root 통해서만
```

## 피드백 형식

```
[Critical] 도메인 내 Lombok 사용 금지
  현재: @Builder public class Order { ... }
  수정: 수동 static inner Builder 구현 또는 Record 전환
  참고: ddd-essentials.md — 도메인 계층 순수성

[Critical] 비즈니스 실패에 Exception 사용
  현재: throw new InsufficientFundsException();
  수정: Sealed Class 기반 Result 패턴 반환
  참고: coding-rules.md — 에러 및 예외 처리

[Major] 원시 타입 집착 (Primitive Obsession)
  현재: String email
  수정: Email(record) Value Object 도입
```

## Quick Rules

리뷰 시 항상 확인:
1. 도메인에 Lombok 사용 금지?
2. 비즈니스 실패를 Result(Sealed Class)로 반환?
3. `synchronized` 사용 (OK) vs `ScopedValue` 사용?
4. 빈약한 도메인 모델 방지 (Behavior in Entity)?
5. 원시 타입 집착 방지 (Value Objects)?
6. 의존성 방향이 안쪽?

## Delegation

| 상황 | 위임 스킬 |
|------|----------|
| Spring Boot 패턴 리뷰 | `java:spring` |
| 새 코드 작성 필요 | `java:coder` |
| 테스트 코드 리뷰 | `java:tester` |

## References
- `references/review-checklist.md` — 동시성, Null/Result, 아키텍처, DDD, Effective Java 체크리스트
