# java

Claude Code용 Java 스킬 — 프로덕션급 Java 패턴, 코드 리뷰, TDD, Java 25 기능, Spring Boot 4 컨벤션을 포함합니다.

## 스킬 목록

| 스킬 | 트리거 | 설명 |
|------|--------|------|
| `java:coder` | Java 코드 작성, 설계, 리팩토링 | 11권 기반 프로덕션급 Java 패턴 (Clean Code, SOLID, DDD, Effective Java, Release It! 등) |
| `java:reviewer` | Java 코드 리뷰, 품질 감사 | 코드 품질, DDD/아키텍처, Effective Java, 안정성, Spring Boot/API 체크리스트 |
| `java:tester` | TDD 워크플로우, 테스트 작성 | Red-Green-Refactor 사이클, 테스트 유형 전략, Mockito, Testcontainers, 레거시 코드 테스트 |
| `java:jdk25` | Java 25 기능 사용, Virtual Thread 안전성 확인 | Records, sealed classes, 패턴 매칭, 가상 스레드, ScopedValue, 구조적 동시성 |
| `java:spring` | Spring Boot 4 코드 작성, 테스트, 리뷰 | 생성자 주입, ProblemDetail, REST 패턴, 테스트 어노테이션, 보안, 관찰가능성 |

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
/java:jdk25
/java:spring
```
