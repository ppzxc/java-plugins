# java-plugins

Production-grade Java 패턴 레퍼런스 플러그인입니다.

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

## 스킬 구성

| 스킬 | 설명 |
|------|------|
| **java-coder** | 코드 작성 — SOLID, Clean Code, Effective Java, DDD, 안정성 패턴 |
| **java-25** | Java 25 기능 — records, sealed classes, pattern matching, virtual threads, ScopedValue |
| **spring** | Spring Boot 4 — 컨벤션, REST 패턴, 테스트 어노테이션, ProblemDetail |
| **java-tester** | TDD 워크플로우 — Red-Green-Refactor, 커버리지 체크리스트, 테스트 더블 |
| **java-reviewer** | 코드 리뷰 — 코드 품질, DDD, Effective Java, 안정성 체크리스트 |

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-plugins
```

## 사용 방법

각 스킬은 컨텍스트에 따라 자동으로 활성화됩니다. 직접 호출도 가능합니다:

```
/java-coder     # 코드 작성 및 설계
/java-25        # Java 25 기능 레퍼런스
/spring         # Spring Boot 4 컨벤션
/java-tester    # TDD 및 테스트
/java-reviewer  # 코드 리뷰
```
