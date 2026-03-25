# java-coder

A production-grade Java patterns plugin for Claude Code containing 5 focused skills.

Based on eleven canonical books: Clean Code, SOLID, Pragmatic Programmer, Code Complete, Refactoring, Design Patterns, TDD, Legacy Code, Effective Java, DDD, Release It!

## Skills

| Skill | Trigger |
|-------|---------|
| **java-coder** | Writing, designing, or refactoring Java code |
| **java-25** | Java 25 features — records, sealed classes, virtual threads, ScopedValue |
| **spring** | Spring Boot 4 — conventions, REST patterns, test annotations, ProblemDetail |
| **java-tester** | TDD workflow, writing tests (*Test.java, *IT.java) |
| **java-reviewer** | Code review, quality audits, architecture compliance |

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-plugins
```

## Usage

Each skill activates automatically based on context:

```
/java-coder     # Code writing and design
/java-25        # Java 25 feature reference
/spring         # Spring Boot 4 conventions
/java-tester    # TDD and testing
/java-reviewer  # Code review
```
