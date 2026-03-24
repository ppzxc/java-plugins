---
name: java-tester
description: >
  Use when applying TDD or writing tests.
  Trigger for explicit TDD workflow requests ("write this test-first"),
  *Test.java or *IT.java file creation or modification,
  test strategy decisions, or coverage analysis.
  NOT for writing production code (java-coder handles that).
---

# Java Tester

TDD workflow and test writing guide for Java 25 + Spring Boot 4.

## TDD Workflow

**Red → Green → Refactor. No new production code without a failing test.**

1. **Red** — Write the smallest failing test that describes one desired behavior
2. **Green** — Write the minimal production code to make the test pass (no more)
3. **Refactor** — Improve structure while keeping all tests green

Repeat per behavior unit. Commit after each green phase.

→ See `../java-coder/references/tdd-and-legacy.md` for deep-dive on test strategy and legacy code techniques

---

## Test Type Strategy

| Type | Scope | Annotation | Location | Share |
|------|-------|-----------|----------|-------|
| No-Docker / Slice | Controller slice, Repository slice, pure unit | `@WebMvcTest`, `@DataJpaTest`, `@ExtendWith(MockitoExtension.class)` | `src/test/java` | ~70% |
| Integration | Full Spring context + real infra | `@SpringBootTest` + Testcontainers | `src/integrationTest/` | ~25% |
| E2E | Critical user flows only | `@SpringBootTest` (HTTP client) | `src/integrationTest/` | ~5% |

> "No-Docker" = fast, no Testcontainers; includes Spring slice context (`@WebMvcTest`, `@DataJpaTest`).

---

## Coverage Checklist

After writing any new class:

- [ ] New **Controller** → `@WebMvcTest` slice test + at least 1 integration test
- [ ] New **Service** → `@ExtendWith(MockitoExtension.class)` unit test
- [ ] New **Repository** with custom query → `@DataJpaTest` or integration test
- [ ] Naming: `methodName_scenario_expectedBehavior()`
- [ ] Minimum: happy path + at least 1 error/edge case

---

## Test Naming

Pattern: `methodName_scenario_expectedBehavior`

```java
@Test
void findById_whenUnauthorized_throwsAccessDeniedException() { ... }

@Test
void createOrder_withValidRequest_returns201WithLocation() { ... }

@Test
void processPayment_whenGatewayTimeout_throwsPaymentException() { ... }
```

---

## @WebMvcTest Pattern (Controller Slice)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;  // stub the service layer only

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

→ See `../java-coder/references/spring-boot4-conventions.md` for Spring test annotation reference

---

## @DataJpaTest Pattern (Repository Slice)

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired OrderRepository repository;

    @Test
    void findByCustomerId_whenOrdersExist_returnsAll() {
        var customerId = CustomerId.of(UUID.randomUUID());
        repository.save(OrderFixture.create(customerId));

        var result = repository.findByCustomerId(customerId);

        assertThat(result).hasSize(1);
    }

    @Test
    void findByCustomerId_whenNoOrders_returnsEmpty() {
        var result = repository.findByCustomerId(CustomerId.of(UUID.randomUUID()));

        assertThat(result).isEmpty();
    }
}
```

---

## @SpringBootTest Pattern (Integration)

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

    @Autowired TestRestTemplate restTemplate;

    @Test
    void createOrder_fullFlow_returns201WithLocation() {
        var response = restTemplate.postForEntity("/api/orders", request, Void.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getHeaders().getLocation()).isNotNull();
    }
}
```

---

## Test Doubles (Mockito)

Mock **only at architectural boundaries** — never mock what you own within the same layer.

```java
// ✅ Correct: Service mocks Repository (crosses infra boundary)
@MockBean OrderRepository repository;

// ✅ Correct: Service mocks external EmailClient (crosses external boundary)
@MockBean EmailClient emailClient;

// ❌ Wrong: Don't mock domain objects — use real instances
// OrderDomainService service = mock(OrderDomainService.class);
```

Rule: if you own the code and it has no external I/O, use the real implementation.

---

## Legacy Code Testing

When adding tests to untested production code:

1. **Find a seam** — a place where you can change behavior without editing existing code
2. **Characterization test** — capture the current behavior first (even if buggy)
3. **Extract and isolate** — use Extract Method to create a testable unit
4. **Then fix** — change behavior under test coverage

→ See `../java-coder/references/tdd-and-legacy.md` for seam types and techniques

---

## Reference File Guide

| File | When to Open |
|------|-------------|
| `../java-coder/references/tdd-and-legacy.md` | TDD deep-dive, legacy code seams, test doubles |
| `../java-coder/references/spring-boot4-conventions.md` | Spring test annotations, slice test configuration |
