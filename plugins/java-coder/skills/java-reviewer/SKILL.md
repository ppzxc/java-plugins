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
