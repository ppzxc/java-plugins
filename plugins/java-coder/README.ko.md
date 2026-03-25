# java-coder

Production-grade Java 패턴 플러그인입니다. 5개의 집중화된 스킬을 포함합니다.

11권의 책 기반: Clean Code, SOLID, Pragmatic Programmer, Code Complete, Refactoring, Design Patterns, TDD, Legacy Code, Effective Java, DDD, Release It!

## 스킬 구성

| 스킬 | 트리거 |
|------|--------|
| **java-coder** | Java 코드 작성, 설계, 리팩토링 |
| **java-25** | Java 25 기능 — records, sealed classes, virtual threads, ScopedValue |
| **spring** | Spring Boot 4 — 컨벤션, REST 패턴, 테스트 어노테이션, ProblemDetail |
| **java-tester** | TDD 워크플로우, 테스트 작성 (*Test.java, *IT.java) |
| **java-reviewer** | 코드 리뷰, 품질 감사, 아키텍처 준수 |

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-plugins
```

## 사용 방법

각 스킬은 컨텍스트에 따라 자동으로 활성화됩니다:

```
/java-coder     # 코드 작성 및 설계
/java-25        # Java 25 기능 레퍼런스
/spring         # Spring Boot 4 컨벤션
/java-tester    # TDD 및 테스트
/java-reviewer  # 코드 리뷰
```
