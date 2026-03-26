---
name: reviewer
description: >
  Use when reviewing Java code, auditing code quality, or checking architecture
  compliance. Trigger on explicit review requests, PR review commands (/review,
  "review this code"), or multi-file quality audits. NOT for generating new code
  or writing tests.
user_invocable: true
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
   → 동시성 안전성 (synchronized, ThreadLocal)
   → Null 안전성 (null 반환, Optional 오용)
   → 아키텍처 위반 (의존성 방향, 도메인 오염)
   → 외부 호출에 Timeout/Circuit Breaker 없음

3. Major Issues — 수정 권장
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
│   → Critical: ReentrantLock으로 교체
│
├─ ThreadLocal 사용?
│   → Critical: ScopedValue 마이그레이션 검토
│
├─ 공유 가변 필드 (non-final)?
│   → Critical: AtomicXxx, ConcurrentHashMap, 불변화
│
└─ Virtual Thread 환경에서 blocking 연산 in synchronized?
    → Critical: 즉시 수정
```

### 아키텍처를 확인한다면?

```
의존성 방향 확인:
Controller → Service → Domain ← Repository Interface
                         ↑
              Infrastructure (Repository 구현체)

위반 패턴:
├─ Controller에 비즈니스 로직 → Service로 이동
├─ Service가 다른 Service를 직접 의존 (순환) → 이벤트/레이어 분리
├─ Domain Entity에 @Autowired → 제거
├─ Repository 구현체가 domain 패키지에 → infrastructure로 이동
└─ Aggregate 내부 Entity를 외부에서 직접 수정 → Root 통해서만
```

### 외부 호출이 있다면?

```
각 외부 호출 확인:
├─ Timeout 설정 있음? (없으면 Critical)
├─ Circuit Breaker 있음? (없으면 Major)
├─ Fallback 있음? (없으면 Major)
└─ 입력 검증 있음? (없으면 Major)
```

## 피드백 형식

```
[Critical] synchronized 사용 → Virtual Thread pin 위험
  현재: synchronized (this) { ... }
  수정: ReentrantLock 사용
  참고: review-checklist.md — 동시성 안전성

[Major] null 반환 → NPE 위험
  현재: return null;
  수정: return Optional.empty(); 또는 빈 컬렉션

[Minor] 메서드명 개선 제안
  현재: void doThing()
  제안: void processPayment()
```

## Quick Rules

리뷰 시 항상 확인:
1. `synchronized` → ReentrantLock?
2. `ThreadLocal` → ScopedValue?
3. `null` 반환 → Optional?
4. DTO가 class → record?
5. 외부 호출에 Timeout?
6. 의존성 방향이 안쪽?
7. 비즈니스 로직이 Controller에 있지 않음?

## Delegation

| 상황 | 위임 스킬 |
|------|----------|
| Spring Boot 패턴 리뷰 | `java:spring` |
| 새 코드 작성 필요 | `java:coder` |
| 테스트 코드 리뷰 | `java:tester` |

## References
- `references/review-checklist.md` — 동시성, Null, 아키텍처, Effective Java, 안정성, Code Smell 체크리스트
