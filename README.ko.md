# java-coder

Production-grade Java 패턴 레퍼런스 스킬입니다.

다음 11권의 책을 기반으로 작성되었습니다:

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

## 포함 내용

1. **Design** — SOLID 원칙, GoF 패턴, DDD tactical 패턴
2. **Clean Code** — 네이밍, 함수 크기, null 안전성, 에러 처리
3. **Java 25 Features** — Records, sealed classes, pattern matching, virtual threads
4. **Spring Boot 4 Conventions** — 생성자 주입, ProblemDetail, 트랜잭션 관리
5. **Effective Java** — Static factory, Builder, 불변성, Optional, enum
6. **DDD in Practice** — Aggregate, Value Object, Domain Event, 모듈 매핑
7. **TDD & Testing** — Red-Green-Refactor, @WebMvcTest, @DataJpaTest, Testcontainers
8. **Stability Patterns** — Timeout, Circuit Breaker, Bulkhead, Retry, virtual thread 안전성

## 설치

```bash
/plugin marketplace add ppzxc/java-skills
/plugin install java-skills
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/java-coder
```

Java 코드 작성, 리뷰, 리팩토링 시 자동으로 활성화되기도 합니다.
