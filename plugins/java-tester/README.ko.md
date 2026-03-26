# java-tester

Java TDD 워크플로우 및 테스트 작성 스킬입니다.

## 포함 내용

1. **TDD 워크플로우** — Red-Green-Refactor 사이클
2. **테스트 유형 전략** — No-Docker/Slice (~70%), Integration (~25%), E2E (~5%)
3. **커버리지 체크리스트** — Controller, Service, Repository 테스트 요구사항
4. **테스트 네이밍** — `methodName_scenario_expectedBehavior` 컨벤션
5. **테스트 더블** — `@Mock` vs `@MockBean` 사용 규칙
6. **레거시 코드 테스트** — 심(Seam), 특성화 테스트, 추출 및 격리

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-tester
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/java-tester
```

TDD 적용, `*Test.java` 또는 `*IT.java` 파일 작성, 테스트 전략 결정 시 자동으로 활성화됩니다.
