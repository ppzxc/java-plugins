---
description: >
  Use when writing, testing, or reviewing Spring Boot 4 code — controllers,
  services, repositories, security configuration, Spring test annotations
  (@WebMvcTest, @DataJpaTest, @SpringBootTest), ProblemDetail error handling,
  API versioning, or any Spring Framework 7 / Jakarta EE 11 patterns.
user-invocable: true
version: 0.0.4
---

# java:spring

## Trigger
- Spring Boot 4 관련 코드 작업 전반
- `@RestController`, `@Service`, `@Repository`, `@Component` 파일
- Spring 테스트 어노테이션 (`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`)
- ProblemDetail 에러 처리
- Spring Security 설정
- Spring Data JPA

## Decision Tree

### 어떤 레이어 코드를 작성하는가?

```
레이어 선택:
│
├─ HTTP 요청/응답 처리 → @RestController
│   → @RequestMapping("/api/v1/resource")
│   → 비즈니스 로직 금지 (Service로 위임)
│   → @Valid 입력 검증
│   → 201 Created + Location 헤더 (생성 시)
│
├─ 비즈니스 로직 조율 → @Service
│   → 생성자 주입만
│   → @Transactional(readOnly = true) 클래스 기본값
│   → 쓰기 메서드만 @Transactional 추가
│
├─ 데이터 접근 → @Repository + JpaRepository
│   → 커스텀 쿼리는 @Query 또는 QueryDSL
│   → N+1 주의 (Fetch Join, EntityGraph)
│
└─ 외부 HTTP 클라이언트 → RestClient (Spring 6.1+)
    → Timeout 설정 필수
    → Circuit Breaker 적용
```

### 에러 응답을 만들어야 한다면?

```
에러 처리 흐름:
│
├─ 도메인 예외 정의
│   → extends RuntimeException
│   → 명확한 이름: OrderNotFoundException, PaymentFailedException
│
├─ @RestControllerAdvice에서 처리
│   → ProblemDetail.forStatusAndDetail(HttpStatus, message)
│   → problem.setTitle(), setType() 설정
│
└─ 검증 오류 (@Valid 실패)
    → MethodArgumentNotValidException 처리
    → errors 필드에 field→message 맵 추가
```

### 테스트를 작성한다면?

```
무엇을 테스트하는가?
│
├─ Controller (HTTP 레이어만)
│   → @WebMvcTest(MyController.class)
│   → @MockBean으로 Service 교체
│   → MockMvc로 요청/응답 검증
│
├─ Repository (DB 쿼리)
│   → @DataJpaTest
│   → H2 인메모리 또는 @Testcontainers
│   → 실제 쿼리 실행 검증
│
└─ 전체 스택 통합
    → @SpringBootTest(webEnvironment = RANDOM_PORT)
    → @Testcontainers + PostgreSQL 컨테이너
    → RestAssured 또는 TestRestTemplate
```

### `@Mock` vs `@MockBean`?

```
스프링 컨텍스트가 필요한가?
│
├─ 필요 없음 (단위 테스트)
│   → @Mock (Mockito)
│   → @ExtendWith(MockitoExtension.class)
│
└─ 필요 (슬라이스/통합 테스트)
    → @MockBean (Spring Test)
    → @WebMvcTest, @DataJpaTest 안에서 사용
```

### JPA 관계에서 N+1이 발생한다면?

```
해결 전략:
│
├─ 단건 조회 → Fetch Join
│   → @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
│
├─ 목록 조회 (페이징 포함) → @EntityGraph
│   → @EntityGraph(attributePaths = {"lines"})
│
└─ 복잡한 조회 → QueryDSL + Projection DTO
    → record로 Projection 정의
```

## Quick Rules

DO:
- 생성자 주입만 (필드 주입 금지)
- `@Transactional(readOnly = true)` 클래스 기본값
- 에러 응답 → ProblemDetail (RFC 9457)
- Controller → `@Valid` 입력 검증
- Controller 테스트 → `@WebMvcTest`
- Repository 테스트 → `@DataJpaTest`
- 생성 응답 → `201 Created + Location 헤더`

DON'T:
- `@Autowired` 필드 주입
- Controller에 비즈니스 로직
- Repository에 `@Transactional`
- @Mock 대신 @MockBean (스프링 컨텍스트 없는 테스트에서)
- 연관 엔티티 EAGER 로딩 기본값

## Delegation

| 상황 | 위임 스킬 |
|------|----------|
| 순수 Java 코드 작성 | `java:coder` |
| 테스트 전략/TDD | `java:tester` |
| 코드 리뷰 | `java:reviewer` |

## References
- `references/spring-boot4-rules.md` — Controller, Service, Repository, 에러 처리, 테스트, 설정, Observability
