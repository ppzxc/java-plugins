# java-25

A Java 25 LTS feature guide skill for Claude Code.

## Contents

1. **Feature Quick Reference** — var, records, sealed classes, pattern matching, virtual threads, ScopedValue, text blocks, SequencedCollection, structured concurrency, stable values
2. **Virtual Thread Safety** — `synchronized` → `ReentrantLock`, `ThreadLocal` → `ScopedValue` (non-negotiable rules)
3. **Records** — DTOs, Value Objects, immutable data carriers
4. **Sealed Classes** — Algebraic types with exhaustive pattern matching switch
5. **ScopedValue** — Replacing ThreadLocal in Virtual Thread environments

## Installation

```bash
/plugin marketplace add ppzxc/java-plugins
/plugin install java-25
```

## Usage

Invoke as a slash command in Claude Code:

```
/java-25
```

Also activates automatically when writing or reviewing Java code that uses Java 25 features — records, sealed classes, virtual threads, ScopedValue, or checking Virtual Thread safety.
