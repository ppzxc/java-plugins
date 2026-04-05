# java:coder Skill Strict DDD & Pure Java 25 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Modify the java:coder skill and its references to enforce Strict DDD, remove Lombok dependency, implement Result Pattern for errors, and update concurrency rules for JDK 25.

**Architecture:** Update markdown files containing instructions and rules for the `java:coder` AI agent.

**Tech Stack:** Markdown

---

### Task 1: Update `SKILL.md`

**Files:**
- Modify: `plugins/java/skills/coder/SKILL.md`

- [ ] **Step 1: Update Decision Tree - Object Creation**

Modify `plugins/java/skills/coder/SKILL.md` to remove the Builder pattern recommendation and enforce manual static inner builders/factory methods for objects with 4+ parameters. Remove Lombok allowance.

- [ ] **Step 2: Update Decision Tree - Concurrency**

Modify `plugins/java/skills/coder/SKILL.md` to allow `synchronized` due to JDK 24+ pinning resolution.

- [ ] **Step 3: Update Decision Tree - Null/Error handling**

Modify `plugins/java/skills/coder/SKILL.md` to recommend `Sealed Class (Result Pattern)` for expected failures instead of throwing exceptions.

- [ ] **Step 4: Update Quick Rules**

Update the "DO" and "DON'T" lists in `SKILL.md` to reflect:
- DO: Result pattern (Sealed interface) for domain errors
- DO: `synchronized` is allowed in JDK 25
- DON'T: Use Lombok (`@Builder`, `@Data`) in domain layer

- [ ] **Step 5: Commit**

```bash
git add plugins/java/skills/coder/SKILL.md
git commit -m "feat(java:coder): update SKILL.md for Strict DDD and JDK 25 concurrency"
```

### Task 2: Update `coding-rules.md`

**Files:**
- Modify: `plugins/java/skills/coder/references/coding-rules.md`

- [ ] **Step 1: Update Object Creation Section**

Rewrite the Builder Pattern rule. Emphasize that Lombok `@Builder` is banned in the domain layer. Show an example of a manual static inner Builder for a `record` or `class`.

- [ ] **Step 2: Update Error Handling Section**

Introduce the "Result Pattern" using Sealed Classes for domain errors. Explicitly state that Exceptions should only be used for infrastructure/unexpected errors.

- [ ] **Step 3: Update Concurrency Section**

Update the `synchronized` rule. State that JEP 491 resolved pinning, so `synchronized` is now preferred in domain logic over `ReentrantLock` for simplicity, unless advanced lock features (like timeouts) are needed.

- [ ] **Step 4: Commit**

```bash
git add plugins/java/skills/coder/references/coding-rules.md
git commit -m "docs(java:coder): update coding-rules.md with strict builder, result pattern, and updated concurrency"
```

### Task 3: Update `ddd-essentials.md`

**Files:**
- Modify: `plugins/java/skills/coder/references/ddd-essentials.md`

- [ ] **Step 1: Add Primitive Obsession Rule**

Add a new rule under Value Object to mitigate Primitive Obsession. Enforce using `record` for wrappers like `Email(String value)` and validating in compact constructors.

- [ ] **Step 2: Add Anemic Domain Model Ban**

Add a rule enforcing that all state changes must happen through domain-specific methods (e.g., `cancel()`) and ban pure getter/setter data holders.

- [ ] **Step 3: Ban Lombok**

Explicitly state that Lombok (`@Builder`, `@Data`, `@Getter`, etc.) is strictly forbidden in Entity, Value Object, and Aggregate classes to ensure encapsulation.

- [ ] **Step 4: Commit**

```bash
git add plugins/java/skills/coder/references/ddd-essentials.md
git commit -m "docs(java:coder): add primitive obsession and anemic model rules to ddd-essentials"
```

### Task 4: Update `jdk25-rules.md`

**Files:**
- Modify: `plugins/java/skills/coder/references/jdk25-rules.md`

- [ ] **Step 1: Update Virtual Threads Section**

Change the rule regarding `synchronized`. Explain that JEP 491 (JDK 24+) resolved carrier thread pinning for `synchronized` blocks. Remove the ban and instead recommend it as the default, replacing the `ReentrantLock` example.

- [ ] **Step 2: Update Pattern Matching Section**

If not present, add a quick example of how Pattern Matching is used with the Result Pattern (Sealed Classes) for error handling.

- [ ] **Step 3: Commit**

```bash
git add plugins/java/skills/coder/references/jdk25-rules.md
git commit -m "docs(java:coder): update jdk25-rules for JEP 491 synchronized support"
```