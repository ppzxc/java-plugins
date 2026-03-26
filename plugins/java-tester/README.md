# java-tester

A Java TDD workflow and test writing skill for Claude Code.

## Contents

1. **TDD Workflow** — Red-Green-Refactor cycle
2. **Test Type Strategy** — No-Docker/Slice (~70%), Integration (~25%), E2E (~5%)
3. **Coverage Checklist** — Controller, Service, Repository test requirements
4. **Test Naming** — `methodName_scenario_expectedBehavior` convention
5. **Test Doubles** — `@Mock` vs `@MockBean` usage rules
6. **Legacy Code Testing** — Seams, characterization tests, extract and isolate

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-tester
```

## Usage

Invoke as a slash command in Claude Code:

```
/java-tester
```

Also activates automatically when applying TDD, writing `*Test.java` or `*IT.java` files, or making test strategy decisions.
