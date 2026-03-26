# java-25

Java 25 LTS 기능 가이드 스킬입니다.

## 포함 내용

1. **기능 빠른 참조** — var, records, sealed classes, pattern matching, virtual threads, ScopedValue, text blocks, SequencedCollection, structured concurrency, stable values
2. **Virtual Thread 안전성** — `synchronized` → `ReentrantLock`, `ThreadLocal` → `ScopedValue` (필수 규칙)
3. **Records** — DTO, Value Object, 불변 데이터 캐리어
4. **Sealed Classes** — 완전한 패턴 매칭 switch와 함께 사용하는 대수 타입
5. **ScopedValue** — Virtual Thread 환경에서 ThreadLocal 대체

## 설치

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-25
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/java-25
```

records, sealed classes, virtual threads, ScopedValue 등 Java 25 기능을 사용하거나 Virtual Thread 안전성 확인 시 자동으로 활성화됩니다.
