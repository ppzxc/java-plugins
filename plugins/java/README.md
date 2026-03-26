# java

Java skills for Claude Code — production-grade Java patterns, code review, TDD, Java 25 features, and Spring Boot 4 conventions.

## Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `java:coder` | Writing, designing, or refactoring Java code | Production-grade Java patterns based on 11 books (Clean Code, SOLID, DDD, Effective Java, Release It!, and more) |
| `java:reviewer` | Reviewing Java code, auditing code quality | Code review checklists — Code Quality, DDD/Architecture, Effective Java, Stability, Spring Boot/API |
| `java:tester` | TDD workflow, writing tests | Red-Green-Refactor cycle, test type strategy, Mockito, Testcontainers, legacy code testing |
| `java:jdk25` | Java 25 feature usage, Virtual Thread safety | Records, sealed classes, pattern matching, virtual threads, ScopedValue, structured concurrency |
| `java:spring` | Spring Boot 4 code, tests, or review | Constructor injection, ProblemDetail, REST patterns, test annotations, security, observability |

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java
```

## Usage

Skills activate automatically based on context, or invoke directly:

```
/java:coder
/java:reviewer
/java:tester
/java:jdk25
/java:spring
```
