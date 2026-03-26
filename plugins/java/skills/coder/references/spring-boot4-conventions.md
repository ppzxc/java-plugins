# Spring Boot 4 Coding Conventions — Detailed Guide

This document defines coding conventions for Spring Boot 4.0 (Spring Framework 7, Jakarta EE 11) environments.

## Table of Contents
1. [Platform Baseline](#platform-baseline)
2. [Project Structure](#project-structure)
3. [Null Safety (JSpecify)](#null-safety)
4. [Modular Auto-Configuration](#modular-auto-configuration)
5. [REST API Design](#rest-api-design)
6. [API Versioning](#api-versioning)
7. [HTTP Service Client](#http-service-client)
8. [Configuration Management](#configuration-management)
9. [Security (Spring Security 7)](#security)
10. [Data Access](#data-access)
11. [Observability](#observability)
12. [Testing](#testing)
13. [Error Handling](#error-handling)
14. [Migration Notes](#migration-notes)

---

## Platform Baseline

- **Minimum JDK**: 17 (backward compatibility)
- **Recommended JDK**: 25 (LTS)
- **Spring Framework**: 7.0
- **Jakarta EE**: 11 (Servlet 6.1, JPA 3.2)
- **Hibernate ORM**: 7.0
- **Kotlin** (if used): 2.2+

---

## Project Structure

### Package Layout (Feature-based vs Layer-based)

**Layer-based** (small projects):
```
com.example.myapp/
├── config/
├── controller/
├── service/
├── repository/
├── domain/
├── dto/
└── exception/
```

**Feature-based** (medium to large projects, recommended):
```
com.example.myapp/
├── order/
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   ├── Order.java
│   ├── CreateOrderRequest.java
│   └── OrderResponse.java
├── payment/
│   ├── PaymentController.java
│   └── ...
├── common/
│   ├── config/
│   └── exception/
└── MyAppApplication.java
```

In feature-based structure, packages represent bounded contexts. This pairs well with Spring Modulith.

---

## Null Safety

Spring Boot 4 systematizes null safety with JSpecify annotations.

### Package-Level Default Policy
Declare `@NullMarked` in every package's `package-info.java`:
```java
@NullMarked
package com.example.myapp.order;

import org.jspecify.annotations.NullMarked;
```
This makes all types in the package @NonNull by default.

### Explicit Nullable
Only annotate `@Nullable` where null is permitted:
```java
import org.jspecify.annotations.Nullable;

public interface OrderRepository extends JpaRepository<Order, Long> {
  @Nullable
  Order findByOrderNumber(String orderNumber);
}
```

### IDE Integration
IntelliJ IDEA 2025.3+ natively supports JSpecify annotations and can detect null violations at compile time.

---

## Modular Auto-Configuration

Spring Boot 4 splits auto-configuration into feature-specific modules. Only add the modules you need:

```groovy
// build.gradle.kts
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

For the legacy Spring Boot 3 monolithic jar, use `spring-boot-autoconfigure-classic`, but prefer modular dependencies for new projects.

---

## REST API Design

### Controller Conventions
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

  @GetMapping
  public Page<OrderResponse> listOrders(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    return orderService.listOrders(PageRequest.of(page, size));
  }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public void cancelOrder(@PathVariable long id) {
    orderService.cancelOrder(id);
  }
}
```

**Conventions:**
- Declare base paths with `@RestController` + `@RequestMapping`
- Use HTTP-method-specific annotations (`@GetMapping`, `@PostMapping`, etc.)
- Always annotate request bodies with `@Valid`
- Explicitly specify response status (201 Created, 204 No Content, etc.)
- Keep controllers thin — delegate all business logic to services

### DTOs (Record-based)
```java
public record CreateOrderRequest(
    @NotBlank String itemCode,
    @Positive int quantity,
    @Nullable String couponCode
) {}

public record OrderResponse(
    long id,
    String orderNumber,
    String status,
    BigDecimal totalAmount,
    Instant createdAt
) {
  public static OrderResponse from(Order order) {
    return new OrderResponse(
        order.getId(),
        order.getOrderNumber(),
        order.getStatus().name(),
        order.getTotalAmount(),
        order.getCreatedAt()
    );
  }
}
```

---

## API Versioning

Spring Boot 4 provides built-in API versioning:

```yaml
# application.yml
spring:
  mvc:
    apiversion:
      type: header    # header, query-parameter, or media-type
      header: X-API-Version
```

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

  @GetMapping(value = "/search", version = "1")
  public List<ProductV1Response> searchV1(@RequestParam String query) {
    return productService.searchV1(query);
  }

  @GetMapping(value = "/search", version = "2")
  public Page<ProductV2Response> searchV2(
      @RequestParam String query, Pageable pageable) {
    return productService.searchV2(query, pageable);
  }
}
```

**Conventions:**
- Decide the versioning strategy (header, query param, media type) early and standardize it
- Provide a deprecation period before removing older versions
- Separate DTOs per version (`ProductV1Response`, `ProductV2Response`)

---

## HTTP Service Client

Use Spring Boot 4's declarative HTTP client. Eliminates RestTemplate / WebClient boilerplate.

```java
@HttpExchange(url = "https://payment-api.example.com")
public interface PaymentClient {

  @PostExchange("/payments")
  PaymentResponse createPayment(@RequestBody PaymentRequest request);

  @GetExchange("/payments/{paymentId}")
  PaymentResponse getPayment(@PathVariable String paymentId);

  @GetExchange("/payments")
  List<PaymentResponse> listPayments(
      @RequestParam("orderId") String orderId);
}
```

Auto-configuration handles bean registration — inject directly without manual setup.

---

## Configuration Management

### Record-based ConfigurationProperties
```java
@ConfigurationProperties(prefix = "app.order")
public record OrderProperties(
    int maxRetryCount,
    Duration timeout,
    @DefaultValue("100") int defaultPageSize
) {}
```

```yaml
# application.yml
app:
  order:
    max-retry-count: 3
    timeout: 30s
    default-page-size: 50
```

**Conventions:**
- Use `application.yml` as default (more readable than `.properties`)
- Separate profile-specific config into `application-{profile}.yml`
- Store sensitive values in environment variables or external secret managers
- Write `@ConfigurationProperties` as records for immutability

---

## Security

Based on Spring Security 7 (Spring Boot 4 default).

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
  }
}
```

**Conventions:**
- Use Lambda DSL (deprecated method-chaining style is removed)
- Enable method-level security with `@EnableMethodSecurity`
- Explicitly configure CORS and CSRF policies

---

## Data Access

### JPA Entity
```java
@Entity
@Table(name = "orders")
public class Order {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true, length = 20)
  private String orderNumber;

  @Enumerated(EnumType.STRING)
  @Column(nullable = false, length = 20)
  private OrderStatus status;

  @Column(nullable = false, precision = 12, scale = 2)
  private BigDecimal totalAmount;

  @Column(nullable = false, updatable = false)
  private Instant createdAt;

  protected Order() {} // JPA default constructor

  public Order(String orderNumber, BigDecimal totalAmount) {
    this.orderNumber = orderNumber;
    this.totalAmount = totalAmount;
    this.status = OrderStatus.PENDING;
    this.createdAt = Instant.now();
  }

  // Expose only getters; replace setters with business methods
  public void approve() {
    if (this.status != OrderStatus.PENDING) {
      throw new IllegalStateException("Can only approve orders in PENDING status");
    }
    this.status = OrderStatus.APPROVED;
  }
}
```

**Conventions:**
- Do NOT use records for entities (JPA requires mutable fields and default constructor)
- Expose meaningful business methods instead of setters
- Specify `@Column` constraints explicitly (nullable, length, etc.)
- Make JPA default constructors `protected`
- Implement `equals()` / `hashCode()` based on business key or ID

### Repository
```java
public interface OrderRepository extends JpaRepository<Order, Long> {

  @Nullable
  Order findByOrderNumber(String orderNumber);

  @Query("""
      SELECT o FROM Order o
      WHERE o.status = :status
        AND o.createdAt >= :since
      ORDER BY o.createdAt DESC
      """)
  Page<Order> findRecentByStatus(
      @Param("status") OrderStatus status,
      @Param("since") Instant since,
      Pageable pageable);

  boolean existsByOrderNumber(String orderNumber);
}
```

---

## Observability

Spring Boot 4 ships with Micrometer 1.16 + OpenTelemetry support by default.

```java
@Service
public class OrderService {

  private final OrderRepository orderRepository;

  @Observed(name = "order.create", contextualName = "create-order")
  public OrderResponse createOrder(CreateOrderRequest request) {
    // Micrometer auto-generates timer, counter, and trace
    var order = new Order(generateOrderNumber(), calculateTotal(request));
    orderRepository.save(order);
    return OrderResponse.from(order);
  }
}
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  observations:
    annotations:
      enabled: true
```

**New OpenTelemetry Starter:**
```groovy
implementation("org.springframework.boot:spring-boot-starter-opentelemetry")
```
This single starter configures OTLP-based metric and trace export.

---

## Testing

### Unit Tests
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

  @Mock
  private OrderRepository orderRepository;

  @Mock
  private PaymentClient paymentClient;

  @InjectMocks
  private OrderService orderService;

  @Test
  @DisplayName("Creates an order for a valid request")
  void should_createOrder_when_validRequest() {
    // given
    var request = new CreateOrderRequest("ITEM-001", 2, null);
    given(orderRepository.save(any(Order.class)))
        .willAnswer(invocation -> invocation.getArgument(0));

    // when
    var response = orderService.createOrder(request);

    // then
    assertThat(response).isNotNull();
    assertThat(response.status()).isEqualTo("PENDING");
    then(orderRepository).should().save(any(Order.class));
  }
}
```

### Integration Tests
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderControllerIntegrationTest {

  @Autowired
  private TestRestClient restClient;

  @Test
  @DisplayName("POST /api/orders creates an order")
  void should_createOrder_via_api() {
    var request = new CreateOrderRequest("ITEM-001", 2, null);

    var response = restClient.post()
        .uri("/api/orders")
        .body(request)
        .exchange()
        .expectStatus().isCreated()
        .expectBody(OrderResponse.class)
        .returnResult()
        .getResponseBody();

    assertThat(response).isNotNull();
    assertThat(response.orderNumber()).isNotBlank();
  }
}
```

**Conventions:**
- Use `@SpringBootTest` only for integration tests (it is heavyweight)
- Prefer slice tests: `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`
- Use `@Testcontainers` for real-database tests
- Use AssertJ as the default assertion library

---

## Error Handling

### RFC 7807 ProblemDetail
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(OrderNotFoundException.class)
  public ProblemDetail handleOrderNotFound(OrderNotFoundException ex) {
    var problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("Order not found");
    problem.setProperty("orderId", ex.getOrderId());
    return problem;
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
    var problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.BAD_REQUEST, "Validation failed");
    problem.setTitle("Validation Error");
    var errors = ex.getBindingResult().getFieldErrors().stream()
        .collect(Collectors.toMap(
            FieldError::getField,
            fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "invalid"
        ));
    problem.setProperty("fieldErrors", errors);
    return problem;
  }
}
```

```yaml
# Enable ProblemDetail
spring:
  mvc:
    problemdetail:
      enabled: true
```

---

## Migration Notes

Key changes when migrating from Spring Boot 3 to 4:

- `javax.*` → `jakarta.*` (should already be done in Boot 3)
- `spring-jcl` removed → use Apache Commons Logging directly
- URL pattern matching: suffix pattern and trailing slash matching removed
- Auto-configuration modules split — import paths may change
- `@Autowired` on constructors: omitted by default for single constructors
- Spring Security: Lambda DSL required, deprecated method chaining removed
- Kotlin: 2.2 baseline
