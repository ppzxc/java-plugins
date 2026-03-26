# Java Plugins 전면 재설계

## Context

현재 java-plugins는 5개 스킬(coder, reviewer, tester, jdk25, spring)과 18개 참조 파일(7,398줄)로 구성되어 있다. 세 가지 근본적 문제가 있다:

1. **컨텍스트 낭비**: 참조 파일이 스킬 간 중복 복사되어 토큰을 과도하게 소비
2. **참조 내용 비효율**: 책 내용을 백과사전식으로 나열하여 Claude가 이미 아는 내용을 반복
3. **스킬 분할 개선 여지**: jdk25가 독립 스킬일 필요가 없고, 결정 트리 형식이 부재

이 재설계는 스킬 구조, 참조 파일, SKILL.md 형식을 모두 처음부터 다시 만든다.

---

## 설계 결정

### 1. 스킬 구조: 5개 → 4개

| 스킬 | 역할 | 변경 |
|------|------|------|
| **java:coder** | Java 코드 작성 + JDK 25 + DDD 핵심 | jdk25 흡수, DDD 핵심 포함 |
| **java:tester** | 테스트 작성 + TDD | 유지 (독립) |
| **java:reviewer** | 코드 리뷰 + 품질 감사 | 유지 (독립) |
| **java:spring** | Spring Boot 4 전용 | 유지 (독립) |

**삭제**: `java:jdk25` — JDK 기능은 코딩 시 항상 필요한 기본 지식이므로 coder에 통합

### 2. 참조 소스: 11권 + 2규격 → 10권 + 2규격

#### 채택 (10권)

**Tier 1 — Claude가 가장 정확히 아는 책 (규칙 추출 확신도 최상)**
1. Effective Java (Joshua Bloch)
2. Refactoring (Martin Fowler)
3. Design Patterns / GoF (Gamma 외)
4. Java Concurrency in Practice (Brian Goetz)
5. Modern Java in Action (Urma 외)

**Tier 2 — 높은 확신**
6. Clean Code (Robert C. Martin)
7. TDD (Kent Beck)
8. Unit Testing Principles, Practices, and Patterns (Vladimir Khorikov)
9. Clean Architecture (Robert C. Martin)
10. Release It! (Michael Nygard)

**추가 규격**
- JDK 25 최신 규격 (coder 참조)
- Spring Boot 4 / Spring Framework 7 / Jakarta EE 11 최신 규격 (spring 참조)

#### 제거

| 책 | 제거 이유 |
|----|----------|
| Code Complete | 너무 일반적, Claude가 이미 자연스럽게 따름 |
| Pragmatic Programmer | 철학적, 구체적 규칙이 아님 |
| DDD (Eric Evans) | 독립 참조 제거. 핵심(Aggregate/Entity/VO)만 coder에 포함 |
| Implementing DDD | DDD와 중복 |

#### 추가

| 책 | 추가 이유 |
|----|----------|
| Java Concurrency in Practice | 동시성/스레드 안전성 규칙이 구체적. Virtual Thread와 연결 |
| Modern Java in Action | Streams, Lambda, Optional 등 현대 Java 패턴 |
| Unit Testing Principles | Mock 전략, 테스트 분류가 tester 스킬에 최적 |

### 3. 참조 파일 전략

#### 원칙: AI 최적화 규칙 형식

책 내용을 그대로 옮기지 않는다. **Claude가 일관되게 따르지 않는 구체적 규칙**만 추출하여 아래 형식으로 작성한다:

```
RULE: [구체적 규칙]
WHEN: [적용 조건/상황]
PATTERN: [코드 패턴 또는 적용 방법]
EXCEPTION: [예외 상황]
```

예시:
```
RULE: public 생성자 대신 static factory 메서드 사용
WHEN: 2개 이상 생성자가 필요하거나, 반환 타입이 인터페이스일 때
PATTERN: of(), from(), valueOf(), getInstance()
EXCEPTION: Spring @Component 등록이 필요한 클래스는 public 생성자 유지
```

#### 스킬별 전용 참조 (중복 제거)

각 스킬은 자신에게 필요한 내용만 담은 전용 참조 파일을 가진다. 동일 참조가 여러 스킬에 복사되지 않는다.

| 스킬 | 참조 파일 | 소스 |
|------|----------|------|
| **java:coder** | `coding-rules.md` (단일 파일, 8권의 규칙을 주제별로 통합) | Effective Java, Refactoring, GoF, Concurrency, Modern Java, Clean Code, Clean Architecture, Release It! |
| | `jdk25-rules.md` | JDK 25 규격 |
| | `ddd-essentials.md` | DDD 핵심 (Aggregate, Entity, Value Object) |
| **java:tester** | `testing-rules.md` | TDD, Unit Testing Principles |
| **java:reviewer** | `review-checklist.md` | Effective Java, Clean Code, Clean Architecture, Refactoring (리뷰 관점) |
| **java:spring** | `spring-boot4-rules.md` | Spring Boot 4 / Framework 7 / Jakarta EE 11 규격 |

### 4. SKILL.md 형식: 결정 트리 중심

현재의 원칙 나열 + 워크플로우 + 체크리스트 혼합을 **결정 트리 중심**으로 변경한다.

#### 구조

```markdown
# 스킬명

## Trigger
[언제 이 스킬이 활성화되는지]

## Decision Tree
[상황 → 판단 → 행동 분기 구조]

## Quick Rules
[DO/DON'T 최소 목록 — Claude가 자주 빠뜨리는 것만]

## Delegation
[다른 스킬로 위임하는 조건]
```

#### 예시 (java:coder Decision Tree)

```
새 클래스 작성?
├── DTO/데이터 운반 → record 사용
├── 도메인 엔티티 → class + 불변 필드 + equals/hashCode
├── 서비스 → 생성자 주입, @Transactional 고려
└── 유틸리티 → final class + private 생성자 + static 메서드

동시성 코드?
├── I/O 바운드 → Virtual Thread (Executors.newVirtualThreadPerTaskExecutor)
├── CPU 바운드 → Platform Thread Pool
├── 공유 상태 → ReentrantLock (synchronized 금지)
└── 스레드 로컬 → ScopedValue (ThreadLocal 금지)

에러 처리?
├── 복구 가능 → checked exception 또는 Optional
├── 프로그래밍 오류 → unchecked exception
├── 외부 서비스 호출 → Timeout + Circuit Breaker + Fallback
└── Spring Controller → ProblemDetail (RFC 9457) → java:spring 위임
```

---

## 디렉토리 구조 (변경 후)

```
java-plugins/
├── plugins/
│   └── java/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           ├── coder/
│           │   ├── SKILL.md              # 결정 트리 중심
│           │   └── references/
│           │       ├── coding-rules.md    # 8권 통합 AI 최적화 규칙
│           │       ├── jdk25-rules.md     # JDK 25 규격
│           │       └── ddd-essentials.md  # DDD 핵심만
│           ├── tester/
│           │   ├── SKILL.md
│           │   └── references/
│           │       └── testing-rules.md   # 2권 통합
│           ├── reviewer/
│           │   ├── SKILL.md
│           │   └── references/
│           │       └── review-checklist.md # 4권 리뷰 관점
│           └── spring/
│               ├── SKILL.md
│               └── references/
│                   └── spring-boot4-rules.md # Spring Boot 4 규격
```

**참조 파일 변화**: 18개(7,398줄) → 6개 (목표: ~3,000줄 이하)

---

## 검증 방법

1. **토큰 절감 확인**: 각 스킬 활성화 시 로딩되는 총 줄 수를 측정하여 현재 대비 50% 이상 감소 확인
2. **트리거 정확도**: 각 스킬의 트리거 조건이 겹치지 않는지 확인
3. **규칙 적용 테스트**: 실제 Java 코드 작성/리뷰/테스트 시나리오에서 스킬이 올바르게 가이드하는지 확인
4. **참조 중복 없음**: 동일 내용이 여러 참조 파일에 존재하지 않는지 확인
