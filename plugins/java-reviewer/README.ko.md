# java-reviewer

Java 코드 리뷰 체크리스트 스킬입니다.

## 포함 내용

1. **코드 품질** — Virtual Thread 안전성, null 처리, DTO 구조, record 활용
2. **DDD / 아키텍처** — 레이어 배치, 의존성 방향, Aggregate 규칙, 도메인 행위
3. **Effective Java** — Static factory/Builder, 불변성, Optional 사용, enum 상수
4. **안정성 / 프로덕션 준비** — 타임아웃, Circuit Breaker, Retry, 입력 검증
5. **Spring Boot / API** — 생성자 주입, ProblemDetail, API 버전 관리, 트랜잭션 규칙

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-reviewer
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/java-reviewer
```

Java 코드 리뷰, 품질 감사, 아키텍처 준수 확인 시 자동으로 활성화됩니다.
