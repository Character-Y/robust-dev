# Bug-Free Incremental Development — Reference

This document provides the theoretical foundations, design rationale, and scenario guidance for the methodology defined in SKILL.md. Read this when you need deeper understanding — for example, when initializing on a brownfield project, debugging a complex cross-session bug, or deciding whether to skip a step.

---

## Why This Methodology Exists

AI within a single context window writes correct, high-quality code for focused tasks. Bugs in AI-collaborative development rarely come from coding mistakes within a single module. They arise from two specific causes:

- **Cross-module side effects**: A change in one part of the system silently affects behavior elsewhere, beyond the AI's current attention scope. For example, adding a `REDIS_URL` environment variable to enable pub/sub messaging may also silently activate a queue-based dispatch mode in an unrelated module.
- **Cross-session context loss**: When a conversation's context window overflows or a new session begins, previous decision context is compressed or lost. The new session sees the code's current state but doesn't know why it's that way or what trade-offs were made.

Both causes reduce to one thing: the system's complexity exceeds the cognitive capacity of a single AI conversation. Bugs are not capability failures — they are visibility failures.

---

## Why Two Layers

### The Gate Alone Is Insufficient

When a test fails in a complex system, the end-to-end test says "something is broken" but doesn't say what or why. Diagnosing this can consume an entire context window worth of tokens. If the AI cannot fix the bug in one session, it opens a new session, loses accumulated debugging context, and often repeats the same failed investigation paths — a failure loop where AI tells the developer it cannot fix the issue, or worse, goes in circles for days.

### The Toolkit Alone Is Insufficient

Without mandatory testing, bugs accumulate silently across multiple changes, making diagnosis exponentially harder when they're eventually discovered.

### Together

The gate ensures bugs are caught immediately (when they're small and close to their cause), and the toolkit ensures caught bugs can be fixed efficiently (within a single session). The gate sets the standard; the toolkit makes the standard achievable.

The toolkit's deepest purpose is keeping the agentic loop running. If the AI detects a bug but cannot fix it efficiently, the collaboration breaks down — not because of the bug itself, but because the AI's debugging cost exceeds a single session's capacity. The toolkit (archived tests, baselines, decision logs, debug summaries) reduces that cost to stay within budget.

---

## Rule Design Rationale

### Rule 1 — The Gate

Bugs must be caught at the moment they are introduced. In incremental development, the cause of a new bug is almost always the most recent change. Delayed detection separates the symptom from its cause, making diagnosis exponentially harder.

### Rule 2 — Debug and Archive

Two aspects deserve explanation:

**Why archiving matters**: During debugging, AI creates targeted tests and accumulates insights at real token cost. If these are discarded, the next similar bug starts from zero. Archiving converts each debugging session into reusable assets. Over time, the high-risk areas of the system naturally accumulate the richest diagnostic coverage, because those are the areas where bugs occur and debugging happens.

**Why developer review matters**: AI may apply surface-level fixes that make tests pass without addressing the underlying issue. The developer reviews the debug analysis and either approves (proceed to archive) or rejects and provides guidance for deeper investigation. Root cause depth is a human judgment call — AI handles the mechanical narrowing, human provides the critical eye.

### Rule 3 — Decision Logging

This rule is auxiliary, not primary. Decision logs cannot capture unforeseen side effects — the most insidious bugs come from impacts nobody anticipated. The decision log accelerates diagnosis when the relevant information was captured, but it is not a replacement for the testing system. It reduces the probability and cost of bugs, not eliminates them.

---

## Initialization Guidance

### Why Conventions Must Be Fixed

If AI names and organizes assets differently each session, the archive becomes unsearchable. The conventions document is the contract that keeps all future sessions consistent. It must define: how files are named (so any future session can find assets predictably), what qualifies as a behavior-altering change for this project (so decision log entries are consistent), and what templates to use for debug summaries and decision entries.

These conventions should fit the project's existing patterns — a Python project might organize differently from a Node project.

### E2E Test Design

The e2e test is the first and most valuable asset. For greenfield projects, it starts minimal and grows with the project. For brownfield projects, it should cover the most critical user-facing path and may take a session or two to design properly.

Key quality criterion: the test should verify output correctness, not just completion. It must be able to distinguish real output from mock/fallback output.

---

## Scenario Guidance

### Greenfield (From Day 1)

Initialize on day one. Each session develops incrementally: write code, extend test, run test, log decisions, commit. Bugs are caught immediately, and the cause is almost always the most recent change. Assets accumulate naturally. Near single-session bug-free development from early on.

### Brownfield (Mid-Way Adoption)

Initialize by analyzing the existing system and writing an initial e2e test. First bug encounters are slower — no archived tests, no decision log history. AI designs diagnostic tests from scratch, archives everything. Subsequent sessions read previous findings, design more targeted tests, narrow further. Each session accumulates assets and reduces the search space for future bugs.

High-risk areas naturally build the richest diagnostic coverage because that's where bugs and debugging concentrate. After several cycles, the system approaches greenfield-level diagnostic efficiency.

### Complex Cross-Session Debugging

For unknown systems requiring complex debugging: the iterative test-narrow-archive cycle works across sessions. Each session's archived results become the starting point for the next. The problem strictly shrinks because assets persist even when conversational context doesn't.

If AI cannot resolve a bug in the current session: present findings, archive everything, and inform the developer. The next session continues from archived state — it does NOT start from scratch.

---

## Assumptions and Limitations

### Assumptions

**AI writes correct code within focused scope.** Bugs come from system-level visibility gaps, not coding skill. If this assumption fails, the methodology helps but cannot fully compensate.

**AI can design effective targeted tests.** Test design for a specific module is a focused task — a corollary of the first assumption.

**Archiving overhead is justified by future savings.** True for complex systems under sustained development. For trivial projects, the overhead may not be worthwhile.

**Asset quality is maintained through conventions.** The initialization conventions keep assets organized and searchable. The developer should periodically verify this.

### Limitations

**Decision logs cannot capture unforeseen side effects.** They record anticipated impacts only. Completely unexpected interactions are caught by the gate but cannot be accelerated by the decision log.

**Root cause depth depends on human judgment.** The methodology structures the process and requires review, but the developer's persistence in questioning is the ultimate quality control.

**Brownfield bootstrapping is gradual.** Multiple debugging sessions are needed to build useful assets. This improves monotonically but cannot be shortcut.

**AI may inconsistently follow rules.** In complex development, AI may skip a log entry or produce a shallow summary. System-level injection (CLAUDE.md or Skill) reduces this risk; developer awareness catches lapses.

---

## Test Assertion Strategy

When designing tests during debugging, consider what level of assertion provides the best trade-off between stability and diagnostic value:

- **Structural assertions** (output shape) — stable but less diagnostic
- **Quantitative assertions** (counts, sizes in expected ranges) — moderately stable with good diagnostic value
- **Content assertions** (output relates to input domain) — most diagnostic but most fragile

Record the assertion level in test metadata so future sessions know what to expect from each test.
