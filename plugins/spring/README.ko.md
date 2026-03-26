# spring

Spring Boot 4 컨벤션 스킬입니다.

## 포함 내용

1. **핵심 컨벤션** — 생성자 주입, `@RestController`, `@Transactional` 배치, ProblemDetail, API 버전 관리, Actuator
2. **REST Controller 패턴** — 생성자 주입을 사용한 표준 컨트롤러 구조
3. **에러 처리** — `@RestControllerAdvice`와 RFC 9457 ProblemDetail
4. **테스트** — `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` + Testcontainers 패턴
5. **리뷰 체크리스트** — Spring Boot / API 품질 검사

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install spring
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/spring
```

Spring Boot 4 코드 작성, 테스트, 리뷰 시 — 컨트롤러, 서비스, 리포지토리, 보안 설정, Spring 테스트 어노테이션 사용 시 자동으로 활성화됩니다.
