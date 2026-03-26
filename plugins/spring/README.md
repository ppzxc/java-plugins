# spring

A Spring Boot 4 conventions skill for Claude Code.

## Contents

1. **Core Conventions** — Constructor injection, `@RestController`, `@Transactional` placement, ProblemDetail, API versioning, Actuator
2. **REST Controller Pattern** — Standard controller structure with constructor injection
3. **Error Handling** — RFC 9457 ProblemDetail with `@RestControllerAdvice`
4. **Testing** — `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` + Testcontainers patterns
5. **Review Checklist** — Spring Boot / API quality checks

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install spring
```

## Usage

Invoke as a slash command in Claude Code:

```
/spring
```

Also activates automatically when writing, testing, or reviewing Spring Boot 4 code — controllers, services, repositories, security configuration, or Spring test annotations.
