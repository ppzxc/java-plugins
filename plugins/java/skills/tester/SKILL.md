---
description: >
  Use when applying TDD or writing test code. Trigger for *Test.java or *IT.java
  file creation/modification, explicit TDD workflow requests, test strategy decisions,
  or coverage analysis. NOT for writing production code (java:coder handles that).
user-invocable: true
version: 0.0.4
---

# java:tester

## Trigger
- `*Test.java` / `*IT.java` 파일 생성 또는 수정
- TDD 워크플로우 명시적 요청 ("테스트 먼저", "TDD로")
- 테스트 전략 결정
- 커버리지 분석

## Decision Tree

### TDD로 기능을 구현한다면?

```
Red-Green-Refactor 사이클:
│
├─ 1. Red: 실패하는 테스트 먼저 작성
│   → 컴파일 오류도 Red다
│   → 한 번에 하나의 테스트만
│
├─ 2. Green: 최소한의 코드로 통과
│   → 지저분해도 괜찮다, 일단 통과
│
└─ 3. Refactor: 동작 유지하며 정리
    → 모든 테스트가 통과하는 상태 유지
```

### 어떤 테스트 유형을 써야 할까?

```
무엇을 테스트하는가?
│
├─ 도메인 로직 / 비즈니스 규칙
│   → 단위 테스트 (Unit Test)
│   → JUnit 5 + AssertJ
│   → Mock 최소화 (실제 도메인 객체 사용)
│
├─ Spring Controller (HTTP 요청/응답)
│   → @WebMvcTest + MockMvc
│   → java:spring 참조
│
├─ JPA Repository (쿼리 검증)
│   → @DataJpaTest
│   → java:spring 참조
│
├─ 여러 컴포넌트 연동 (DB, 외부 API)
│   → 통합 테스트 (Integration Test)
│   → @SpringBootTest + Testcontainers
│
└─ 핵심 사용자 시나리오 전체
    → E2E 테스트 (최소한으로)
```

### Mock을 써야 할까?

```
의존성이 무엇인가?
│
├─ 외부 서비스 (이메일, SMS, 외부 API)
│   → Mock 사용 (경계에서만)
│   → @Mock (단위), @MockitoBean (스프링 슬라이스, Spring Boot 4+)
│
├─ DB / 영속성
│   → Real DB 선호 (H2 또는 Testcontainers)
│   → Mock Repository는 리팩토링 내성 약함
│
└─ 같은 모듈 내 도메인 객체
    → Mock 금지 — 실제 객체 사용
```

### 테스트가 자꾸 깨진다면?

```
왜 깨지는가?
│
├─ 구현 변경(리팩토링) 때마다 깨짐
│   → 내부 구현을 테스트하고 있는 것
│   → verify(mock.내부메서드()) 제거
│   → 최종 결과(public API)만 검증
│
├─ 다른 테스트에 영향받음
│   → 테스트 간 상태 공유 중
│   → @BeforeEach에서 독립적으로 초기화
│
└─ 순서에 따라 결과가 달라짐
    → 테스트가 실행 순서에 의존
    → 각 테스트가 자신의 데이터를 설정하도록
```

### 레거시 코드에 테스트를 추가한다면?

```
1. 먼저 Characterization Test 작성
   → 현재 동작을 그대로 기록 (옳고 그름 판단 안 함)
   → 테스트가 통과되면 동작이 문서화된 것

2. Seam 찾기
   → Object Seam: 생성자에 의존성 주입 추가
   → Interface Seam: new ConcreteClass() → 인터페이스 주입

3. 안전하게 수정
```

## Quick Rules

DO:
- 테스트 이름: `methodName_scenario_expectedResult`
- Arrange-Act-Assert 구조 유지
- 하나의 테스트 = 하나의 논리적 동작 검증
- AssertJ 사용 (`assertThat()`)
- 실패 테스트를 먼저 작성 (TDD)

DON'T:
- 구현 세부사항 (private 메서드, 내부 호출) 검증
- 도메인 내부 객체 Mock
- 테스트 간 상태 공유
- 단순 getter/setter 테스트 (가치 없음)
- 테스트를 통과시키기 전에 다음 테스트 작성

## 테스트 비율 목표
- **단위 테스트**: 70% (도메인 로직 중심)
- **통합 테스트**: 25% (DB, 외부 연동)
- **E2E 테스트**: 5% (핵심 시나리오)

## Delegation

| 상황 | 위임 스킬 |
|------|----------|
| 프로덕션 코드 작성 | `java:coder` |
| Spring 테스트 어노테이션 사용법 (@WebMvcTest, @DataJpaTest 등) | `java:spring` |
| 순수 도메인 로직 Mock 전략 | 이 스킬에서 제공 |

## References
- `references/testing-rules.md` — TDD 사이클, Mock 전략, AssertJ 패턴, 레거시 테스트
