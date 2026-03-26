# java-plugins

Claude Code용 프로덕션급 Java 레퍼런스 플러그인. 10권의 핵심 도서 + JDK 25 + Spring Boot 4 규격에서 추출한 AI 최적화 규칙을 제공합니다.

## 참조 도서

**Tier 1** (가장 높은 신뢰도의 규칙):
- *Effective Java* (Joshua Bloch)
- *Refactoring* (Martin Fowler)
- *Design Patterns* / GoF (Gamma 외)
- *Java Concurrency in Practice* (Brian Goetz)
- *Modern Java in Action* (Urma 외)

**Tier 2**:
- *Clean Code* (Robert C. Martin)
- *Test-Driven Development* (Kent Beck)
- *Unit Testing Principles, Practices, and Patterns* (Vladimir Khorikov)
- *Clean Architecture* (Robert C. Martin)
- *Release It!* (Michael Nygard)

**규격**:
- JDK 25 LTS
- Spring Boot 4 / Spring Framework 7 / Jakarta EE 11

## 스킬 구성

| 스킬 | 설명 |
|------|------|
| `java:coder` | Java 코드 작성 — 결정 트리 형식, JDK 25 통합, DDD 핵심 |
| `java:reviewer` | 코드 리뷰 — 동시성 안전성, Null 안전성, 아키텍처, 안정성 체크리스트 |
| `java:tester` | TDD 워크플로우 — Red-Green-Refactor, Mock 전략, 레거시 코드 테스트 |
| `java:spring` | Spring Boot 4 — REST 패턴, ProblemDetail, 테스트 어노테이션, Observability |

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java
```

## 사용 방법

각 스킬은 컨텍스트에 따라 자동으로 활성화됩니다. 직접 호출도 가능합니다:

```
/java:coder     # Java 코드 작성 및 설계
/java:reviewer  # 코드 리뷰 및 품질 감사
/java:tester    # TDD 및 테스트
/java:spring    # Spring Boot 4 컨벤션
```
