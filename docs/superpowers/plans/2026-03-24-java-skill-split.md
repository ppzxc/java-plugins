# Java Skill Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the monolithic `java-coder` SKILL.md into three focused skills: `java-coder` (writing), `java-reviewer` (review), `java-tester` (testing).

**Architecture:** Three independent SKILL.md files. `java-coder` is modified to remove test/review content. `java-reviewer` and `java-tester` are new files in sibling directories. All three share the existing `references/` directory via relative paths (`../java-coder/references/`).

**Tech Stack:** Markdown only — no build tools or test frameworks. Verification is done with `grep`.

---

## File Map

| Action | Path |
|--------|------|
| Modify | `plugins/java-coder/skills/java-coder/SKILL.md` |
| Create | `plugins/java-coder/skills/java-reviewer/SKILL.md` |
| Create | `plugins/java-coder/skills/java-tester/SKILL.md` |
| Unchanged | `plugins/java-coder/skills/java-coder/references/` (all 9 files — including `tdd-and-legacy.md`, which is kept on disk but removed from `java-coder`'s Reference File Guide table) |
| Unchanged | `plugins/java-coder/.claude-plugin/plugin.json` |

---

## Task 1: Modify `java-coder` SKILL.md

Remove test/review content. Four targeted edits to the existing file.

**Files:**
- Modify: `plugins/java-coder/skills/java-coder/SKILL.md`

---

- [ ] **Step 1: Verify current state — confirm sections to remove exist**

```bash
grep -n "reviewing\|test writing\|tdd-and-legacy\|커버리지 검증\|Test Conventions\|Java-Specific Checklist" \
  plugins/java-coder/skills/java-coder/SKILL.md
```

Expected: lines found for each keyword (confirms content that will be removed).

---

- [ ] **Step 2: Update frontmatter `description` — remove "reviewing" and "test writing"**

In `plugins/java-coder/skills/java-coder/SKILL.md`, replace the `description` block in the frontmatter:

**Old:**
```yaml
description: >
  Use when writing, reviewing, refactoring, or designing Java code.
  Trigger for any .java file creation or modification, REST API design, Spring Boot configuration,
  JPA entity modeling, test writing, or code review involving Java 25, Spring Boot 4, or
  domain modeling patterns.
```

**New:**
```yaml
description: >
  Use when writing, designing, or refactoring Java code.
  Trigger for .java file creation or modification, REST API design, Spring Boot configuration,
  JPA entity modeling, or domain modeling involving Java 25, Spring Boot 4.
  NOT for test file writing, code review, or quality audits.
```

---

- [ ] **Step 3: Replace Step 3 (TDD) with a cross-reference link**

Replace the following verbatim block (preserve the blank line before `### Step 4`):

**Old** (exact match — replace entire block up to and including the last line before the blank line before `### Step 4`):
```
### Step 3: Write Tests First (TDD)

Red → Green → Refactor. No new code without a failing test.

- **Unit tests** (70%): `@WebMvcTest`, `@DataJpaTest`, Mockito for boundaries
- **Integration tests** (25%): `@SpringBootTest` + Testcontainers
- **E2E** (5%): Critical flows only
- Test names: `methodName_scenario_expectedBehavior()`

**커버리지 검증** (새 기능/클래스 작성 후):
- [ ] 새 Controller → `@WebMvcTest` 단위 테스트 + 통합 테스트 파일 존재 확인
- [ ] 새 Service → `@ExtendWith(MockitoExtension.class)` 단위 테스트 파일 존재 확인
- [ ] 새 Repository (커스텀 쿼리) → `@DataJpaTest` 또는 통합 테스트 존재 확인
- [ ] 테스트 네이밍: `methodName_scenario_expectedBehavior()` 패턴 준수
- [ ] 최소: happy path + 주요 에러 케이스 1개 이상

→ See `references/tdd-and-legacy.md` for test strategy and legacy code techniques
```

**New** (keep one blank line after, so `### Step 4` follows normally):
```
### Step 3: Write Tests First (TDD)

→ Activate **java-tester** skill for the full TDD workflow, coverage checklist, and test patterns.
```

---

- [ ] **Step 4: Remove the `## Test Conventions` section**

Remove the entire block below. The section starts at `## Test Conventions` and ends at the `---` separator. The blank line between `---` and `## Error Handling / Logging` must be preserved.

**Block to delete** (starts at `## Test Conventions`, ends at and includes `---`):

```
## Test Conventions

```
src/test/java          — Unit tests (@WebMvcTest, @DataJpaTest) — no Docker
src/integrationTest/   — Integration tests (@SpringBootTest + Testcontainers)
```

Test naming: `methodName_scenario_expectedBehavior`

```java
@Test
void findById_whenUnauthorized_throwsAccessDeniedException() { ... }
```

---
```

After deletion, `## Error Handling / Logging` should follow with one blank line gap.

---

- [ ] **Step 5: Remove the `## Java-Specific Checklist` section**

Remove the entire section from `## Java-Specific Checklist` through its closing `---` separator. This section has five sub-sections: Code Quality, DDD / Architecture, Effective Java, Stability / Production Readiness, Spring Boot / API.

---

- [ ] **Step 6: Remove `tdd-and-legacy.md` row from the Reference File Guide table**

In the `## Reference File Guide` table at the bottom of the file, remove the row:

```
| `references/tdd-and-legacy.md` | Test strategy, legacy code seams, test doubles |
```

---

- [ ] **Step 7: Verify removals**

```bash
grep -n "reviewing\|test writing\|tdd-and-legacy\|커버리지 검증\|Test Conventions\|Java-Specific Checklist" \
  plugins/java-coder/skills/java-coder/SKILL.md
```

Expected: **no output** (all removed content is gone).

> Note: `ReentrantLock` and `Circuit Breaker` legitimately remain in the retained `## Stability Patterns` section — do NOT include them in this negative check.

```bash
grep -n "java-tester\|Activate java-tester" plugins/java-coder/skills/java-coder/SKILL.md
```

Expected: Step 3 line found.

---

- [ ] **Step 8: Commit**

```bash
git add plugins/java-coder/skills/java-coder/SKILL.md
git commit -m "refactor(skill): remove test/review content from java-coder skill

Extracted TDD step, test conventions, and Java-specific checklist
to dedicated java-tester and java-reviewer skills."
```

---

## Task 2: Create `java-reviewer` SKILL.md

New skill file. Review workflow + 5 checklists + refactoring references.

**Files:**
- Create: `plugins/java-coder/skills/java-reviewer/SKILL.md`

---

- [ ] **Step 1: Verify directory doesn't exist yet**

```bash
ls plugins/java-coder/skills/
```

Expected: only `java-coder/` listed (no `java-reviewer/` yet).

---

- [ ] **Step 2: Create the file**

Create `plugins/java-coder/skills/java-reviewer/SKILL.md` with the following content:

```markdown
---
name: java-reviewer
description: >
  Use when reviewing Java code, auditing code quality, or checking
  architecture compliance. Trigger on explicit review requests, PR review
  commands (/review, "review this code"), or multi-file quality audits.
  "PR review" means any code diff or multi-file review, not GitHub-specific tools.
  NOT for generating new code or writing tests.
---

# Java Reviewer

Review Java code for quality, architecture compliance, and production readiness.

## Review Workflow

1. **Scan** — Read the full diff or file set before commenting
2. **Categorize** — Group issues by category (Code Quality / DDD / Effective Java / Stability / Spring Boot)
3. **Guide** — For each issue: state the problem, cite the rule, suggest the refactoring technique
4. **Reference** — Point to the relevant reference file for deep-dive

---

## Code Quality

### Checklist
- [ ] No `synchronized` → use `ReentrantLock` instead
- [ ] No `ThreadLocal` → use `ScopedValue` instead
- [ ] No null returns → `Optional` or empty collections
- [ ] No inner class/record → each DTO in its own file
- [ ] Records used for DTOs and Value Objects

**When found:** See `../java-coder/references/release-it-stability.md` (Virtual Thread safety) and `../java-coder/references/clean-and-pragmatic.md` (structure rules)

**Refactoring techniques:**
- `synchronized` → Replace with `ReentrantLock`; check for `ThreadLocal` in same class
- Inner DTO → Extract Class (move to its own file)

---

## DDD / Architecture

### Checklist
- [ ] Correct layer placement (domain / application / infrastructure / presentation)
- [ ] No upward dependency violations (e.g., domain → infrastructure is forbidden)
- [ ] Aggregate accessed through root only — no direct child entity mutation
- [ ] Domain objects carry behavior (no Anemic Domain Model)

**When found:** See `../java-coder/references/domain-driven-design.md`

**Refactoring techniques:**
- Layer violation → Extract Interface (keep interface in domain, move impl to infrastructure)
- Anemic model → Move Method (push behavior into the domain object)
- Direct child access → Add Method on Aggregate Root

---

## Effective Java

### Checklist
- [ ] Static factory or Builder used where appropriate (≥4 params or optional fields)
- [ ] Value Objects are immutable (`record` or `final` fields, no setters)
- [ ] `Optional` used only as return type — never as field or parameter
- [ ] `enum` used instead of int constants

**When found:** See `../java-coder/references/effective-java.md`

**Refactoring techniques:**
- Constructor with ≥4 params → Introduce Builder
- Mutable VO → Replace with `record` or make fields `final`
- Int constants → Replace Type Code with Enum

---

## Stability / Production Readiness

### Checklist (required when external calls present)
- [ ] **All** external calls (HTTP/SMTP/Redis/external DB) have explicit `connectTimeout` + `readTimeout`
- [ ] Unstable dependencies have Circuit Breaker (`@CircuitBreaker` or programmatic)
- [ ] Circuit Breaker has fallback method (degraded mode)
- [ ] Retry applies to idempotent operations only (GET, PUT) + exponential backoff
- [ ] Redis/cache failure has auth-path fallback
- [ ] Input validated at system boundary (controller/filter)

**When found:** See `../java-coder/references/release-it-stability.md`

**Refactoring techniques:**
- Missing timeout → add `connectTimeout`/`readTimeout` to client config
- Missing Circuit Breaker → add `@CircuitBreaker(name = "...", fallbackMethod = "...")`
- Retry on POST → remove or add idempotency key first

---

## Spring Boot / API

### Checklist
- [ ] Constructor injection only — no `@Autowired` on fields
- [ ] `@Transactional` on Service layer, not Repository interface
- [ ] Error responses use `ProblemDetail` + `application/problem+json` (RFC 9457)
- [ ] API versioning applied consistently across all endpoints
- [ ] `201 Created` + `Location` header on resource creation

**When found:** See `../java-coder/references/spring-boot4-conventions.md`

**Refactoring techniques:**
- Field injection → Extract Constructor (move `@Autowired` fields to constructor params)
- Non-standard error body → Replace with `ProblemDetail.forStatusAndDetail(...)`

---

## Reference File Guide

| File | When to Open |
|------|-------------|
| `../java-coder/references/clean-and-pragmatic.md` | Naming, function size, structure rules |
| `../java-coder/references/design-and-solid.md` | SOLID violations, GoF pattern misuse |
| `../java-coder/references/refactoring-catalog.md` | Specific refactoring technique lookup |
| `../java-coder/references/effective-java.md` | EJ idiom violations |
| `../java-coder/references/domain-driven-design.md` | DDD and architecture issues |
| `../java-coder/references/spring-boot4-conventions.md` | Spring annotation and API issues |
| `../java-coder/references/release-it-stability.md` | Stability and Virtual Thread issues |
```

---

- [ ] **Step 3: Verify the file was created with correct structure**

```bash
grep -n "name:\|description:\|## Review Workflow\|## Code Quality\|## DDD\|## Effective Java\|## Stability\|## Spring Boot\|java-coder/references" \
  plugins/java-coder/skills/java-reviewer/SKILL.md
```

Expected: all 5 checklist sections and reference paths found.

---

- [ ] **Step 4: Commit**

```bash
git add plugins/java-coder/skills/java-reviewer/SKILL.md
git commit -m "feat(skill): add java-reviewer skill

Code review workflow with 5 checklist categories and refactoring
guidance. References shared via ../java-coder/references/ path."
```

---

## Task 3: Create `java-tester` SKILL.md

New skill file. TDD workflow + test patterns + coverage checklist.

**Files:**
- Create: `plugins/java-coder/skills/java-tester/SKILL.md`

---

- [ ] **Step 1: Verify directory doesn't exist yet**

```bash
ls plugins/java-coder/skills/
```

Expected: `java-coder/` and `java-reviewer/` listed (no `java-tester/` yet).

---

- [ ] **Step 2: Create the file**

Create `plugins/java-coder/skills/java-tester/SKILL.md` with the following content:

```markdown
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
```

---

- [ ] **Step 3: Verify the file was created with correct structure**

```bash
grep -n "name:\|description:\|## TDD Workflow\|## Test Type\|## Coverage\|## @WebMvcTest\|## @DataJpaTest\|## @SpringBootTest\|## Test Doubles\|## Legacy\|java-coder/references" \
  plugins/java-coder/skills/java-tester/SKILL.md
```

Expected: all major sections and reference paths found.

---

- [ ] **Step 4: Commit**

```bash
git add plugins/java-coder/skills/java-tester/SKILL.md
git commit -m "feat(skill): add java-tester skill

TDD Red-Green-Refactor workflow, test type strategy, coverage checklist,
and concrete patterns for @WebMvcTest, @DataJpaTest, @SpringBootTest.
References shared via ../java-coder/references/ path."
```

---

## Final Verification

- [ ] **Confirm all three skills exist**

```bash
ls plugins/java-coder/skills/
```

Expected: `java-coder/  java-reviewer/  java-tester/`

- [ ] **Confirm java-coder no longer contains checklist or TDD content**

```bash
grep -c "Java-Specific Checklist\|커버리지 검증\|tdd-and-legacy\|Test Conventions" \
  plugins/java-coder/skills/java-coder/SKILL.md
```

Expected: `0`

- [ ] **Confirm java-reviewer has all 5 checklist categories**

```bash
grep "^## " plugins/java-coder/skills/java-reviewer/SKILL.md
```

Expected: `## Review Workflow`, `## Code Quality`, `## DDD / Architecture`, `## Effective Java`, `## Stability / Production Readiness`, `## Spring Boot / API`, `## Reference File Guide`

- [ ] **Confirm java-tester has TDD workflow and test patterns**

```bash
grep "^## " plugins/java-coder/skills/java-tester/SKILL.md
```

Expected: `## TDD Workflow`, `## Test Type Strategy`, `## Coverage Checklist`, `## Test Naming`, `## @WebMvcTest Pattern`, `## @DataJpaTest Pattern`, `## @SpringBootTest Pattern`, `## Test Doubles`, `## Legacy Code Testing`, `## Reference File Guide`

- [ ] **Confirm reference paths are correct in new skills**

```bash
grep "references/" plugins/java-coder/skills/java-reviewer/SKILL.md | head -3
grep "references/" plugins/java-coder/skills/java-tester/SKILL.md | head -3
```

Expected: all paths start with `../java-coder/references/`
