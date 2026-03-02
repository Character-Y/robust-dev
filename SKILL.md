---
name: robust-dev
description: >
  Bug-free incremental development methodology for human-AI collaborative software projects.
  Ensures every code commit delivers verified, working code through mandatory end-to-end testing
  and an auxiliary debugging toolkit. Automatically applies to all development work in this project.
---

# Bug-Free Incremental Development

For deeper understanding of design rationale, scenario guidance (greenfield vs brownfield), and known limitations, read `reference.md` in this skill directory. Consult it when facing non-routine situations — complex cross-session debugging, brownfield initialization, or trade-off decisions about skipping steps.

You operate under two layers: a **gate** (mandatory e2e test before every commit) and an **auxiliary toolkit** (accumulated test assets, decision logs, debug summaries that let you diagnose and fix bugs within a single session). The gate catches bugs immediately; the toolkit ensures you can fix them efficiently without losing context across sessions.

## The Three Rules

**Rule 1: Every change must pass the e2e test before commit.**
Do not consider any change complete until all tests pass. Use a stop hook or pre-commit check.

**Rule 2: When a test fails, follow the debug workflow and archive all artifacts.**
Narrow down → diagnose → fix → present analysis to developer for approval → archive everything (diagnostic tests, debug summary, findings). Developer approval is required — they judge whether the root cause analysis is deep enough.

**Rule 3: Record intent and anticipated impact for behavior-altering changes.**
Decision logs help future sessions connect symptoms to causes. What qualifies as "behavior-altering" is project-specific — define criteria during initialization. Guiding principle: if a future AI session, unaware of this change, might misunderstand the system's behavior — log it.

## Activation

On every session start:

1. Check: does the project have asset infrastructure (conventions document + index)?
2. **No** → Run initialization (below)
3. **Yes** → Read conventions document and index. Read recent decision logs and debug summaries if continuing previous work. Apply the three rules to all development.

## Initialization (First Time Only)

### 1. Analyze the Project
- Identify major modules and boundaries
- Identify the core user-facing path ("happy path")
- Identify configuration mechanisms (env vars, config files, feature flags)
- Note existing naming and coding conventions
- For brownfield: note untested areas and known fragile points

### 2. Create Asset Infrastructure
Create an asset directory at the project root. Design the structure based on project needs. Required components:
- E2e test and diagnostic tests (separated — they serve different purposes)
- Test run baselines (successful) and debug summaries (failures), stored separately for comparison
- Decision log entries
- **Conventions document**: naming rules, decision log criteria, templates, test commands. This contract keeps all future sessions consistent — fix it after initialization.
- **Master index**: catalogs all assets for quick lookup by module, date, and status

### 3. Create Initial E2E Test
- Use a known, small test fixture (checked into repo)
- Exercise the main pipeline from input to output
- Verify output correctness at key checkpoints — not just completion, but whether output is reasonable
- Must run fast enough to run after every change without hesitation

### 4. Initialize the Index
Create the master index with the initial e2e test entry.

## Normal Development

Develop normally. The three rules operate in the background:
- After each code change: run e2e test → if pass, save baseline record + log decision if applicable + commit → if fail, enter debug workflow
- When adding functionality: extend the e2e test. Split if it grows too slow.

## Debug Workflow

### 1. Initial Assessment
- Read test failure output — note the specific error and which assertion failed
- Check decision log for recent related changes
- In incremental development, the most recent change is usually the cause — check there first

### 2. Narrow Down (if cause isn't obvious)
- Check index for archived diagnostic tests related to the failing area — run them first
- If none exist, design targeted tests for suspected modules
- Run to identify which module or boundary is broken
- Archive newly designed tests immediately — don't wait until after the fix

### 3. Diagnose
- Examine relevant code
- Cross-reference decision log for recent changes to this module or its dependencies
- Compare current failure against last passing baseline — the delta points to the broken component
- Form hypothesis, verify with targeted test
- If one session cannot solve it, archive all findings for the next session to continue from

### 4. Fix and Verify
- Apply fix
- Run e2e test to confirm
- Run archived diagnostic tests for the area to check regressions
- Log decision if the fix is behavior-altering

### 5. Human Review
Present to developer:
- Symptom observed
- Direct cause
- Why this cause was able to exist (one level deeper)
- What was fixed
- Whether structural prevention is recommended

Developer approves or rejects. If rejected, investigate deeper per developer guidance.

### 6. Archive (after approval)
- Write debug summary per project template
- Archive new diagnostic tests with metadata
- Update index
- Extend e2e test if a gap was revealed

## Asset Management

- **Test archive grows organically** from debugging, not as a separate effort. High-risk areas naturally accumulate the most coverage.
- **Assertion levels** — structural (stable, less diagnostic) → quantitative (moderate) → content (diagnostic, fragile). Record in test metadata.
- **Staleness** — lazy maintenance: mark stale tests in index, update only when needed during debugging. Mark deprecated when modules are removed.
- **Decision logs and debug summaries are append-only.** To reverse a decision, write a new entry referencing the old one.
- **Index** — update on every asset change. Keep small enough to read at session start. Split if too large.
- **Conventions can evolve** but changes must be deliberate and documented, not silently drifted.

## Human-AI Boundaries

**Involve developer for**: debug review (every bug fix), extended debugging (can't resolve in one session — archive and report), architecture decisions revealed by debugging.

**Act autonomously for**: running/designing tests, writing decision logs, archiving assets, extending e2e test, maintaining index.

**Developer can always**: reject analysis, provide hints, modify assets directly, override the gate, request asset cleanup.

## Checklists

### Post-Fix
- [ ] E2e test passes
- [ ] Debug summary written
- [ ] New diagnostic tests archived
- [ ] Index updated
- [ ] Developer review obtained

### Post-Feature
- [ ] E2e test extended
- [ ] All tests pass
- [ ] Decision log entry written (if applicable)
- [ ] Index updated
