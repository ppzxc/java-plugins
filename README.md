# java-plugins

A production-grade Java reference plugin for Claude Code. AI-optimized rules extracted from 10 canonical books + JDK 25 + Spring Boot 4 specifications.

## Reference Books

**Tier 1** (highest confidence rules):
- *Effective Java* (Joshua Bloch)
- *Refactoring* (Martin Fowler)
- *Design Patterns* / GoF (Gamma et al.)
- *Java Concurrency in Practice* (Brian Goetz)
- *Modern Java in Action* (Urma et al.)

**Tier 2**:
- *Clean Code* (Robert C. Martin)
- *Test-Driven Development* (Kent Beck)
- *Unit Testing Principles, Practices, and Patterns* (Vladimir Khorikov)
- *Clean Architecture* (Robert C. Martin)
- *Release It!* (Michael Nygard)

**Specifications**:
- JDK 25 LTS
- Spring Boot 4 / Spring Framework 7 / Jakarta EE 11

## Skills

| Skill | Description |
|-------|-------------|
| `java:coder` | Java code writing — decision tree format, JDK 25 integration, DDD essentials |
| `java:reviewer` | Code review — concurrency safety, null safety, architecture, stability checklists |
| `java:tester` | TDD workflow — Red-Green-Refactor, Mock strategy, legacy code testing |
| `java:spring` | Spring Boot 4 — REST patterns, ProblemDetail, test annotations, observability |

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java
```

## Usage

Skills activate automatically based on context. You can also invoke directly:

```
/java:coder     # Java code writing and design
/java:reviewer  # Code review and quality audit
/java:tester    # TDD and testing
/java:spring    # Spring Boot 4 conventions
```
