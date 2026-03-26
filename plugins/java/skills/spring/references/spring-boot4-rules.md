# Spring Boot 4 Rules
> Spring Boot 4.x / Spring Framework 7.x / Jakarta EE 11 기준
> Format: RULE → WHEN → PATTERN → EXCEPTION

---

## 핵심 컨벤션

RULE: 생성자 주입만 사용 (`@Autowired` 필드 주입 금지)
WHEN: 모든 의존성 주입
PATTERN:
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    // Spring이 자동으로 생성자 주입 (Lombok @RequiredArgsConstructor 가능)
    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
}
```
EXCEPTION: 없음

RULE: `@Transactional`은 Service 레이어
WHEN: 트랜잭션 경계 설정
PATTERN:
```java
@Service
@Transactional(readOnly = true)  // 클래스 기본값: 읽기 전용
public class OrderService {
    public Order findById(UUID id) { ... }  // readOnly = true 상속

    @Transactional  // 쓰기 작업만 별도 지정
    public Order create(CreateOrderRequest req) { ... }
}
```
EXCEPTION: Controller, Repository에 `@Transactional` 금지

RULE: null safety — `@NonNull`/`@Nullable` 어노테이션 사용 (Spring Framework 7)
WHEN: public API 매개변수/반환값
PATTERN:
```java
import org.springframework.lang.NonNull;
import org.springframework.lang.Nullable;

public @NonNull Order findById(@NonNull UUID id) { ... }
public @Nullable Order findOrNull(@NonNull UUID id) { ... }
```

---

## REST Controller

RULE: Controller는 얇게 유지 — 비즈니스 로직 금지
WHEN: Controller 메서드 작성
PATTERN:
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @GetMapping("/{id}")
    public OrderResponse findById(@PathVariable UUID id) {
        return orderService.findById(id);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void cancel(@PathVariable UUID id) {
        orderService.cancel(id);
    }
}
```

RULE: 생성 응답은 `201 Created + Location 헤더`
WHEN: POST로 리소스 생성 시
PATTERN:
```java
@PostMapping
public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
    OrderResponse order = orderService.create(req);
    URI location = URI.create("/api/v1/orders/" + order.id());
    return ResponseEntity.created(location).body(order);
}
```

RULE: API 버전관리는 URL 경로 방식 (`/api/v1/`)
WHEN: REST API 버전 관리
PATTERN: `/api/v1/orders`, `/api/v2/orders`
EXCEPTION: 헤더 버전관리는 클라이언트 구현 복잡도 증가로 지양

---

## 에러 처리 (RFC 9457 ProblemDetail)

RULE: 예외 처리는 `@RestControllerAdvice` + `ProblemDetail` 사용
WHEN: HTTP 에러 응답 포맷
PATTERN:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Order Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation Error");
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> Objects.requireNonNullElse(fe.getDefaultMessage(), "invalid")));
        problem.setProperty("errors", errors);
        return problem;
    }
}
```

RULE: 도메인 예외는 명시적 클래스로 정의
WHEN: 비즈니스 규칙 위반
PATTERN:
```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(UUID id) {
        super("Order not found: " + id);
    }
}
public class OrderAlreadyConfirmedException extends RuntimeException { ... }
```

---

## 요청 검증

RULE: DTO에 Bean Validation 어노테이션 적용
WHEN: Controller 입력 검증
PATTERN:
```java
public record CreateOrderRequest(
    @NotBlank(message = "item must not be blank")
    String item,

    @Positive(message = "quantity must be positive")
    int quantity,

    @Valid  // 중첩 객체 검증
    @NotNull
    ShippingAddress address
) {}
```

RULE: Controller 메서드에 `@Valid` 어노테이션
WHEN: 요청 객체 검증 활성화
PATTERN: `@Valid @RequestBody CreateOrderRequest request`

---

## 테스트

RULE: Controller 테스트는 `@WebMvcTest`
WHEN: Controller 레이어 격리 테스트
PATTERN:
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;  // 협력 빈은 MockBean

    @Test
    void create_validRequest_returns201() throws Exception {
        given(orderService.create(any())).willReturn(new OrderResponse(UUID.randomUUID(), "item"));

        mockMvc.perform(post("/api/v1/orders")
                .contentType(APPLICATION_JSON)
                .content("""{"item":"book","quantity":1}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.item").value("book"));
    }
}
```

RULE: Repository 테스트는 `@DataJpaTest`
WHEN: JPA Repository 격리 테스트
PATTERN:
```java
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository orderRepository;
    // 실제 H2(또는 Testcontainers) DB 사용

    @Test
    void findByCustomerId_returnsOnlyCustomerOrders() {
        // ...
    }
}
```

RULE: 통합 테스트는 `@SpringBootTest` + Testcontainers
WHEN: 전체 스택 통합 테스트
PATTERN:
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
}
```

RULE: `@Mock` vs `@MockBean`
WHEN: 테스트 유형에 따라 선택
PATTERN:
- `@Mock` (Mockito 순수): 스프링 컨텍스트 없는 단위 테스트
- `@MockBean` (Spring): `@WebMvcTest`, `@DataJpaTest` 등 스프링 컨텍스트 내 Mock

---

## 데이터 접근

RULE: `@Repository` 인터페이스는 `JpaRepository` 확장
WHEN: Spring Data JPA 사용
PATTERN:
```java
public interface OrderRepository extends JpaRepository<Order, UUID> {
    List<Order> findByCustomerId(UUID customerId);

    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.createdAt > :since")
    List<Order> findRecentByStatus(
        @Param("status") OrderStatus status,
        @Param("since") LocalDateTime since);
}
```

RULE: N+1 문제 방지
WHEN: 연관 엔티티를 함께 로딩할 때
PATTERN:
```java
// Fetch Join
@Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
Optional<Order> findWithLines(@Param("id") UUID id);

// EntityGraph
@EntityGraph(attributePaths = {"lines", "customer"})
Optional<Order> findDetailById(UUID id);
```

---

## 설정 및 프로퍼티

RULE: 환경별 설정은 `application-{profile}.yml`
WHEN: 개발/테스트/운영 환경 구분
PATTERN:
```
application.yml          # 공통 설정
application-dev.yml      # 개발 환경
application-prod.yml     # 운영 환경 (시크릿은 환경변수)
application-test.yml     # 테스트 환경
```

RULE: 커스텀 프로퍼티는 `@ConfigurationProperties`
WHEN: 관련 설정 묶음
PATTERN:
```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(
    String apiUrl,
    String apiKey,
    Duration timeout
) {}

// @SpringBootApplication에 추가
@EnableConfigurationProperties(PaymentProperties.class)
```

---

## Observability

RULE: 헬스체크와 메트릭은 Actuator
WHEN: 운영 환경 모니터링
PATTERN:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info
  endpoint:
    health:
      show-details: when-authorized
```

RULE: 분산 추적 ID는 로그에 포함
WHEN: 마이크로서비스 또는 멀티 스레드 환경
PATTERN: Micrometer Tracing + 로그 패턴에 `%X{traceId}` 추가
