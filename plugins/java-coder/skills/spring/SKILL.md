---
name: spring
description: >
  Use when writing, testing, or reviewing Spring Boot 4 code — controllers, services, repositories,
  security configuration, Spring test annotations (@WebMvcTest, @DataJpaTest, @SpringBootTest),
  ProblemDetail error handling, API versioning, or any Spring Framework 7 / Jakarta EE 11 patterns.
---

# Spring Boot 4

Spring Boot 4 (Spring Framework 7, Jakarta EE 11) conventions for writing, testing, and reviewing.

## Core Conventions

- **Constructor injection only** — no `@Autowired` on fields
- `@RestController` + `@RequestMapping` on class; method-level `@GetMapping` etc.
- `@Transactional` on Service layer, never on Repository interface
- `ProblemDetail` (RFC 9457) for all error responses — `application/problem+json`
- `201 Created` + `Location` header on resource creation
- API versioning via header/URI/content negotiation — decide early and standardize
- Health checks via Spring Actuator (`/actuator/health`)

Full reference → `../java-coder/references/spring-boot4-conventions.md`

---

## REST Controller Pattern

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

  private final OrderService orderService;

  public OrderController(OrderService orderService) {
    this.orderService = orderService;
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
    return orderService.createOrder(request);
  }

  @GetMapping("/{id}")
  public OrderResponse getOrder(@PathVariable long id) {
    return orderService.getOrderById(id);
  }
}
```

---

## Error Handling (RFC 9457 ProblemDetail)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(OrderNotFoundException.class)
  public ProblemDetail handleOrderNotFound(OrderNotFoundException ex) {
    var problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setProperty("orderId", ex.getOrderId());
    return problem;
  }
}
```

```yaml
spring:
  mvc:
    problemdetail:
      enabled: true
```

---

## Testing

### Test Type Strategy

| Type | Annotation | Location | Share |
|------|-----------|----------|-------|
| No-Docker / Slice | `@WebMvcTest`, `@DataJpaTest`, `@ExtendWith(MockitoExtension.class)` | `src/test/java` | ~70% |
| Integration | `@SpringBootTest` + Testcontainers | `src/integrationTest/` | ~25% |
| E2E | `@SpringBootTest` (HTTP client) | `src/integrationTest/` | ~5% |

**`@Mock` vs `@MockBean`:**
- `@Mock` for `@ExtendWith(MockitoExtension.class)` tests (no Spring context)
- `@MockBean` for `@WebMvcTest` / `@SpringBootTest` tests (Spring context required)

### @WebMvcTest (Controller Slice)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

  @Autowired MockMvc mockMvc;
  @MockBean OrderService orderService;

  @Test
  void getOrder_whenExists_returns200() throws Exception {
    given(orderService.findById(any())).willReturn(Optional.of(order));

    mockMvc.perform(get("/api/orders/{id}", orderId))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.id").value(orderId.toString()));
  }

  @Test
  void getOrder_whenNotFound_returns404WithProblemDetail() throws Exception {
    given(orderService.findById(any())).willReturn(Optional.empty());

    mockMvc.perform(get("/api/orders/{id}", orderId))
        .andExpect(status().isNotFound())
        .andExpect(content().contentType("application/problem+json"));
  }
}
```

### @DataJpaTest (Repository Slice)

```java
@DataJpaTest
class OrderRepositoryTest {

  @Autowired OrderRepository repository;

  @Test
  void findByCustomerId_whenOrdersExist_returnsAll() {
    repository.save(OrderFixture.create(customerId));
    assertThat(repository.findByCustomerId(customerId)).hasSize(1);
  }

  @Test
  void findByCustomerId_whenNoOrders_returnsEmpty() {
    assertThat(repository.findByCustomerId(CustomerId.of(UUID.randomUUID()))).isEmpty();
  }
}
```

### @SpringBootTest (Integration)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderIntegrationTest {

  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

  @DynamicPropertySource
  static void configure(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired TestRestClient restClient;

  @Test
  void createOrder_fullFlow_returns201WithLocation() {
    var response = restClient.post()
        .uri("/api/orders")
        .body(request)
        .retrieve()
        .toBodilessEntity();

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    assertThat(response.getHeaders().getLocation()).isNotNull();
  }
}
```

---

## Review Checklist (Spring Boot / API)

- [ ] Constructor injection only — no `@Autowired` on fields
- [ ] `@Transactional` on Service layer, not Repository interface
- [ ] Error responses use `ProblemDetail` + `application/problem+json` (RFC 9457)
- [ ] API versioning applied consistently across all endpoints
- [ ] `201 Created` + `Location` header on resource creation

**Refactoring:**
- Field injection → Extract Constructor (move `@Autowired` fields to constructor params)
- Non-standard error body → Replace with `ProblemDetail.forStatusAndDetail(...)`

---

## Reference File Guide

| File | When to Open |
|------|-------------|
| `../java-coder/references/spring-boot4-conventions.md` | Full Spring Boot 4 conventions — project structure, null safety, data access, security, observability, migration notes |
