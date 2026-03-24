# java-coder

A production-grade Java patterns reference skill for Claude Code.

Based on eleven canonical books:

- *Clean Code* (Robert C. Martin)
- *Agile Software Development: Principles, Patterns, and Practices* (Robert C. Martin)
- *The Pragmatic Programmer* (Hunt & Thomas)
- *Code Complete* (Steve McConnell)
- *Refactoring* (Martin Fowler)
- *Design Patterns: Elements of Reusable Object-Oriented Software* (Gamma et al.)
- *Test-Driven Development* (Kent Beck)
- *Working Effectively with Legacy Code* (Michael Feathers)
- *Effective Java* (Joshua Bloch)
- *Domain-Driven Design* (Eric Evans)
- *Release It!* (Michael Nygard)

## Contents

1. **Design** — SOLID principles, GoF patterns, DDD tactical patterns
2. **Clean Code** — Naming, function size, null safety, error handling
3. **Java 25 Features** — Records, sealed classes, pattern matching, virtual threads
4. **Spring Boot 4 Conventions** — Constructor injection, ProblemDetail, transaction management
5. **Effective Java** — Static factories, builders, immutability, Optional, enums
6. **DDD in Practice** — Aggregate, Value Object, Domain Event, module mapping
7. **TDD & Testing** — Red-Green-Refactor, @WebMvcTest, @DataJpaTest, Testcontainers
8. **Stability Patterns** — Timeout, Circuit Breaker, Bulkhead, Retry, virtual thread safety

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-plugins
```

## Usage

Invoke as a slash command in Claude Code:

```
/java-coder
```

Also activates automatically when writing, reviewing, or refactoring Java code.
