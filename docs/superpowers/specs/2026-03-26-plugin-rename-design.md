# Plugin Rename: java-coder:java-coder → java:coder

## Context

현재 java-plugins marketplace의 스킬들이 `java-coder:java-coder`, `java-reviewer:java-reviewer` 등 플러그인명과 스킬명이 중복되는 형태로 로딩된다. 이를 `java:coder`, `java:reviewer` 등 깔끔한 네임스페이스 형태로 변경한다.

## 변경 전/후

| 현재 | 변경 후 |
|------|---------|
| `java-coder:java-coder` | `java:coder` |
| `java-reviewer:java-reviewer` | `java:reviewer` |
| `java-tester:java-tester` | `java:tester` |
| `java-25:java-25` | `java:jdk25` |
| `spring:spring` | `java:spring` |

## 설계

### 구조 변경: 5 plugins → 1 plugin, 5 skills

```
plugins/java/
  .claude-plugin/plugin.json       # name: "java"
  skills/
    coder/SKILL.md                 # → java:coder
    coder/references/
    reviewer/SKILL.md              # → java:reviewer
    reviewer/references/
    tester/SKILL.md                # → java:tester
    tester/references/
    jdk25/SKILL.md                 # → java:jdk25
    jdk25/references/
    spring/SKILL.md                # → java:spring
    spring/references/
```

### 변경 파일 목록

1. **`.claude-plugin/marketplace.json`** — 5개 plugin 엔트리를 1개로 통합
2. **`plugins/java/.claude-plugin/plugin.json`** — 새 단일 plugin.json (name: "java")
3. **`plugins/java/skills/coder/SKILL.md`** — 기존 java-coder SKILL.md에서 name을 `coder`로 변경
4. **`plugins/java/skills/reviewer/SKILL.md`** — name을 `reviewer`로 변경
5. **`plugins/java/skills/tester/SKILL.md`** — name을 `tester`로 변경
6. **`plugins/java/skills/jdk25/SKILL.md`** — name을 `jdk25`로 변경, description도 업데이트
7. **`plugins/java/skills/spring/SKILL.md`** — name을 `spring`로 변경
8. **각 스킬의 references/ 디렉토리** — 기존 위치에서 새 위치로 이동
9. **기존 5개 plugin 디렉토리 삭제** — `plugins/java-coder/`, `plugins/java-reviewer/`, `plugins/java-tester/`, `plugins/java-25/`, `plugins/spring/`

### SKILL.md description 업데이트

각 SKILL.md의 description에서 상호 참조 시 `(java-tester)` → `(java:tester)` 등으로 변경.

### 리스크 및 대응

- **marketplace 표시 문제**: 이전 1-plugin-5-skills 구조에서 스킬이 표시되지 않았던 원인은 `user_invocable: true` 누락일 가능성이 높음. 이번에는 모든 SKILL.md에 `user_invocable: true`를 유지하므로 문제 없을 것으로 예상.
- **롤백**: 문제 발생 시 git revert로 즉시 복구 가능.

## Verification

1. `marketplace.json`이 유효한 JSON인지 확인
2. 모든 SKILL.md에 `user_invocable: true`가 포함되어 있는지 확인
3. 디렉토리 구조가 `plugins/java/skills/{name}/SKILL.md` 패턴을 따르는지 확인
4. 기존 5개 plugin 디렉토리가 삭제되었는지 확인
5. Claude Code에서 `/skills` 명령으로 `java:coder`, `java:reviewer` 등이 정상 표시되는지 확인
