# Design Spec: java-coder Plugin Skill Split

**Date**: 2026-03-24
**Status**: Approved
**Scope**: Split `java-coder` single skill into five focused skills — `java-coder`, `java-reviewer`, `java-tester`, `java-25`, `spring`

---

## Context

The existing `java-coder` skill bundles code writing, code review, and test guidance into a single SKILL.md. This causes two problems:

1. **Trigger ambiguity** — the skill activates in too many contexts, reducing focus
2. **Content overload** — one skill covers ~350 lines mixing unrelated workflows

**Goal**: Separate into three independent, purpose-focused skills with distinct triggers and content.

---

## Approach: Complete Independent Separation

Each skill is self-contained. `java-coder` contains a cross-reference hint to `java-tester` at Step 3, but no TDD content. No content is duplicated across skills.

---

## Trigger Conflict Resolution

When multiple skills could match:
- `java-coder` is the default for any `.java` file work (new code, modification, refactoring)
- `java-tester` activates **only** when the user explicitly requests TDD guidance, or the target file is `*Test.java` / `*IT.java`
- `java-reviewer` activates **only** when the user explicitly requests a review, audit, or PR check — never during code generation

---

## Skill 1: `java-coder` (Writing & Designing)

### Trigger
```
Use when writing, designing, or refactoring Java code.
Trigger for .java file creation or modification, REST API design,
Spring Boot configuration, JPA entity modeling, or domain modeling
involving Java 25, Spring Boot 4.
NOT for test file writing, code review, or quality audits.
```

### Content (retained from current SKILL.md)
- Core Principles
- Code Writing Workflow:
  - Step 1: Design (SOLID, DP, DDD)
  - Step 2: Write Clean Code (CC, PP, EJ)
  - Step 3: **Replaced with** `→ Activate java-tester skill for TDD workflow`
  - Step 4: Refactor (RF, LEG)
  - Step 5: Harden for Production (RI) — **writing-time prompt** (what to apply while coding)
- Formatting & Naming
- Java 25 Feature Usage
- Spring Boot 4 Conventions
- Effective Java Essentials
- DDD in Practice
- Stability Patterns — writing-time guidance (how to implement timeouts, circuit breakers, etc.)
- Error Handling / Logging

**Note on Step 5 vs. java-reviewer stability checklist**: Step 5 in `java-coder` is a writing-time prompt (what to implement). The stability checklist in `java-reviewer` is a review-time verification (was it implemented correctly). Both exist independently; neither duplicates the other.

### Removed from java-coder
- TDD / Test coverage checklist (→ `java-tester`)
- Java-specific Checklist (5 categories) (→ `java-reviewer`)
- Test Conventions section (→ `java-tester`)

### References
- `references/clean-and-pragmatic.md`
- `references/design-and-solid.md`
- `references/refactoring-catalog.md`
- `references/effective-java.md`
- `references/domain-driven-design.md`
- `references/java25-features.md`
- `references/spring-boot4-conventions.md`
- `references/release-it-stability.md`

> `tdd-and-legacy.md` is **removed** from java-coder references — it has no content consumer after Step 3 is replaced with a link.

---

## Skill 2: `java-reviewer` (Code Review)

### Trigger
```
Use when reviewing Java code, auditing code quality, or checking
architecture compliance. Trigger on explicit review requests, PR review
commands (/review, "review this code"), or multi-file quality audits.
"PR review" means any code diff or multi-file review, not GitHub-specific tools.
NOT for generating new code or writing tests.
```

### Content (new skill)
- **Review Workflow**: Detect issue → Categorize → Guide refactoring technique → Reference
- **Code Quality Checklist** (from java-coder checklist):
  - No `synchronized` → `ReentrantLock`; see `references/release-it-stability.md`
  - No `ThreadLocal` → `ScopedValue`; see `references/release-it-stability.md`
  - No null returns → `Optional` or empty collections; see `references/effective-java.md`
  - No inner DTO classes → each DTO in own file; see `references/clean-and-pragmatic.md`
  - Records for DTOs and VOs; see `references/effective-java.md`
- **DDD / Architecture Checklist**:
  - Correct layer placement; see `references/domain-driven-design.md`
  - No upward dependency violations
  - Aggregate root access only
  - No Anemic Domain Model
- **Effective Java Checklist**:
  - Static factory or Builder (≥4 params)
  - Immutable Value Objects
  - `Optional` as return type only
  - `enum` over int constants
- **Stability / Production Readiness Checklist** (review-time verification):
  - Timeout on all external calls (HTTP/SMTP/Redis/external DB)
  - Circuit Breaker on unstable dependencies
  - Fallback defined for Circuit Breaker
  - Retry on idempotent operations only + exponential backoff
  - Redis/cache fallback for auth path
  - Input validation at controller/filter boundary
- **Spring Boot / API Checklist**:
  - Constructor injection only (no `@Autowired` on fields)
  - `@Transactional` on Service, not Repository
  - `ProblemDetail` for errors (`application/problem+json`)
  - API versioning applied consistently
  - `201 Created` + `Location` header on resource creation

### References
- `../java-coder/references/clean-and-pragmatic.md`
- `../java-coder/references/design-and-solid.md`
- `../java-coder/references/refactoring-catalog.md`
- `../java-coder/references/effective-java.md`
- `../java-coder/references/domain-driven-design.md`
- `../java-coder/references/spring-boot4-conventions.md`
- `../java-coder/references/release-it-stability.md`

---

## Skill 3: `java-tester` (Testing)

### Trigger
```
Use when applying TDD or writing tests.
Trigger for explicit TDD workflow requests ("write this test-first"),
*Test.java or *IT.java file creation or modification,
test strategy decisions, or coverage analysis.
NOT for writing production code (java-coder handles that).
```

### Content (new skill — extracted + expanded from java-coder)
- **TDD Red-Green-Refactor Workflow** (from Step 3)
- **Test Type Strategy** (no-Docker vs Docker boundary):
  - No-Docker/Slice tests (70%): `@WebMvcTest`, `@DataJpaTest`, Mockito at boundaries
    - "No-Docker" = fast, no Testcontainers; includes Spring slice context
  - Integration tests (25%): `@SpringBootTest` + Testcontainers
  - E2E (5%): Critical flows only
- **Coverage Checklist** (from java-coder):
  - New Controller → `@WebMvcTest` unit test + integration test
  - New Service → `@ExtendWith(MockitoExtension.class)` unit test
  - New Repository (custom query) → `@DataJpaTest` or integration test
  - Naming: `methodName_scenario_expectedBehavior()`
  - Minimum: happy path + 1 error case
- **Test Conventions**:
  - `src/test/java` — no-Docker tests
  - `src/integrationTest/` — Testcontainers integration tests
  - Naming pattern: `methodName_scenario_expectedBehavior`
- **@WebMvcTest Pattern**: mock MVC layer only, stub Service with `@MockBean`
- **@DataJpaTest Pattern**: real DB slice (H2 or Testcontainers), no Web layer
- **@SpringBootTest Pattern**: full context, Testcontainers for infrastructure
- **Test Doubles (Mockito)**: mock at architectural boundaries only; use real implementations within same layer
- **Legacy Code Testing**: seam-based techniques from `references/tdd-and-legacy.md`

### References
- `../java-coder/references/tdd-and-legacy.md`
- `../java-coder/references/spring-boot4-conventions.md` (test annotations)

---

## File Structure After Implementation

```
plugins/java-coder/
  skills/
    java-25/
      SKILL.md           ← new file (Java 25 features, Virtual Thread safety)
    java-coder/
      SKILL.md           ← modified (cross-references to java-25, spring, java-tester, java-reviewer)
      references/        ← unchanged (9 files, shared via relative path by all 5 skills)
        clean-and-pragmatic.md
        design-and-solid.md
        refactoring-catalog.md
        effective-java.md
        domain-driven-design.md
        java25-features.md
        spring-boot4-conventions.md
        release-it-stability.md
        tdd-and-legacy.md
    java-reviewer/
      SKILL.md           ← new file (code review checklists, references via ../java-coder/references/)
    java-tester/
      SKILL.md           ← new file (TDD workflow, references via ../java-coder/references/)
    spring/
      SKILL.md           ← new file (Spring Boot 4 conventions, test patterns, review checklist)
  .claude-plugin/
    plugin.json          ← unchanged (skills auto-discovered from directory structure)
```

**Reference path resolution**: All skills reference shared files using `../java-coder/references/<file>.md`. No file copying or symlinking required. The Claude Code skill runtime resolves these as standard relative filesystem paths.

**Settings file**: `.claude/settings.json` does NOT need manual updates — skills are auto-discovered from the `plugins/java-coder/skills/` directory structure.

---

## Verification

1. `java-coder` SKILL.md no longer contains checklist categories, TDD workflow, Java 25 feature table, or Spring Boot conventions
2. `java-coder` SKILL.md Step 3 reads `→ Activate java-tester skill for TDD workflow`
3. `java-25` SKILL.md contains Java 25 feature reference, Virtual Thread safety rules, records, sealed classes, ScopedValue
4. `spring` SKILL.md contains Spring Boot 4 conventions, REST controller pattern, ProblemDetail error handling, test patterns (@WebMvcTest, @DataJpaTest, @SpringBootTest), review checklist
5. `java-reviewer` SKILL.md contains 4 inline checklist categories + cross-reference to `spring` skill for Spring Boot / API checklist
6. `java-tester` SKILL.md contains TDD Red-Green-Refactor + coverage checklist + cross-reference to `spring` skill for test patterns
7. Each SKILL.md `description` frontmatter satisfies non-overlapping trigger conditions
8. All reference paths use `../java-coder/references/` prefix
9. No content is duplicated across skills (each skill either owns the content or cross-references another skill)
