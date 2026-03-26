# java

Claude Code용 Java 스킬 — 프로덕션급 Java 패턴, 코드 리뷰, TDD, JDK 25 기능, Spring Boot 4 컨벤션을 포함합니다.

## 스킬 목록

| 스킬 | 트리거 | 설명 |
|------|--------|------|
| `java:coder` | Java 코드 작성, 설계, 리팩토링 | 결정 트리 중심 코딩 규칙 — 10권 + JDK 25 + DDD 핵심 |
| `java:reviewer` | Java 코드 리뷰, 품질 감사 | AI 최적화 리뷰 체크리스트 — 동시성 안전성, Null 안전성, 아키텍처, 안정성 |
| `java:tester` | TDD 워크플로우, 테스트 작성 | Red-Green-Refactor 사이클, Mock 전략, AssertJ 패턴, 레거시 코드 테스트 |
| `java:spring` | Spring Boot 4 코드 작성, 테스트, 리뷰 | 생성자 주입, ProblemDetail (RFC 9457), REST 패턴, Spring 테스트 어노테이션 |

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java
```

## 사용 방법

컨텍스트에 따라 자동으로 활성화되거나, 직접 호출할 수 있습니다:

```
/java:coder
/java:reviewer
/java:tester
/java:spring
```
