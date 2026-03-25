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

## Spring Test Patterns

→ See **spring** skill for `@WebMvcTest`, `@DataJpaTest`, and `@SpringBootTest` patterns.

---

## Test Doubles (Mockito)

Mock **only at architectural boundaries** — never mock what you own within the same layer.

```java
// ✅ Unit test (no Spring context): use @Mock
@Mock OrderRepository repository;
@Mock EmailClient emailClient;

// ✅ Slice/integration test (Spring context): use @MockBean
@MockBean OrderRepository repository;
@MockBean EmailClient emailClient;

// ❌ Wrong: Don't mock domain objects — use real instances
// OrderDomainService service = mock(OrderDomainService.class);
```

Rule: if you own the code and it has no external I/O, use the real implementation.
`@Mock` for `@ExtendWith(MockitoExtension.class)` tests (no Spring context).
`@MockBean` for `@WebMvcTest` / `@SpringBootTest` tests (Spring context required).

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
| **spring** skill | Spring test annotations, @WebMvcTest, @DataJpaTest, @SpringBootTest patterns |
