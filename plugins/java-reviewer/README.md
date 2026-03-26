# java-reviewer

A Java code review checklist skill for Claude Code.

## Contents

1. **Code Quality** — Virtual Thread safety, null handling, DTO structure, record usage
2. **DDD / Architecture** — Layer placement, dependency direction, aggregate rules, domain behavior
3. **Effective Java** — Static factory/Builder, immutability, Optional usage, enum constants
4. **Stability / Production Readiness** — Timeouts, Circuit Breaker, Retry, input validation
5. **Spring Boot / API** — Constructor injection, ProblemDetail, API versioning, transaction rules

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-reviewer
```

## Usage

Invoke as a slash command in Claude Code:

```
/java-reviewer
```

Also activates automatically when reviewing Java code, auditing code quality, or checking architecture compliance.
