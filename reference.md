# Bug-Free Incremental Development

A methodology for building and maintaining large software systems through human-AI collaboration, ensuring every commit delivers a verified, working product from the user's perspective.

---

## 1. Core Principle

### 1.1 The Goal

Every code commit in a human-AI collaborative development process should deliver a bug-free product from the user's perspective, verified by tests that replicate the user's real experience.

### 1.2 Why This Is Hard

AI within a single context window writes correct, high-quality code for focused tasks. Bugs in AI-collaborative development rarely come from coding mistakes within a single module. They arise from two specific causes:

- **Cross-module side effects**: A change in one part of the system silently affects behavior elsewhere, beyond the AI's current attention scope. For example, adding a `REDIS_URL` environment variable to enable pub/sub messaging may also silently activate a queue-based dispatch mode in an unrelated module.
- **Cross-session context loss**: When a conversation's context window overflows or a new session begins, previous decision context is compressed or lost. The new session sees the code's current state but doesn't know why it's that way or what trade-offs were made.

Both causes reduce to one thing: the system's complexity exceeds the cognitive capacity of a single AI conversation. Bugs are not capability failures — they are visibility failures.

### 1.3 The Two-Layer Solution

**Layer 1 — The Gate**: A mandatory end-to-end test runs after every code change. Nothing is committed until all tests pass. This catches bugs at the moment they're introduced, when the relevant context is still fresh and the cause is close at hand.

**Layer 2 — The Auxiliary Toolkit**: A set of accumulated assets (archived tests, decision records, debug summaries) that reduce the cost of diagnosing and fixing bugs when the gate catches them. Without this layer, the gate becomes an unpassable barrier — the AI detects a bug but cannot fix it within a single session, leading to context loss, repeated failed attempts, and eventual collaboration breakdown.

### 1.4 Why Both Layers Are Necessary

The gate alone is insufficient. When a test fails in a complex system, the end-to-end test says "something is broken" but doesn't say what or why. Diagnosing this can consume an entire context window worth of tokens. If the AI cannot fix the bug in one session, it opens a new session, loses accumulated debugging context, and often repeats the same failed investigation paths — a failure loop where AI tells the developer it cannot fix the issue, or worse, goes in circles for days.

The toolkit alone is insufficient. Without mandatory testing, bugs accumulate silently across multiple changes, making diagnosis exponentially harder when they're eventually discovered.

Together: the gate ensures bugs are caught immediately (when they're small and close to their cause), and the toolkit ensures caught bugs can be fixed efficiently (within a single session). The gate sets the standard; the toolkit makes the standard achievable.

---

## 2. The Three Rules

These rules are the operational core of the methodology. They form a logical progression: Rule 1 defines the standard (every change must be tested), Rule 2 defines how to handle failure (debug efficiently and accumulate assets), Rule 3 reduces the likelihood of failure in the first place (record decisions to prevent cross-session blind spots). They do not change regardless of project size, type, or stage.

### Rule 1: Every change must pass an end-to-end test before commit.

This is the gate. Bugs must be caught at the moment they are introduced. In incremental development, the cause of a new bug is almost always the most recent change. Delayed detection separates the symptom from its cause, making diagnosis exponentially harder.

Use a stop hook or pre-commit check. AI must not consider a change complete until all tests pass.

### Rule 2: When a test fails, follow the debug workflow and archive all artifacts.

This is the response to failure. When the gate catches a bug, AI must: narrow down the problem (using archived tests if available, or designing new targeted tests), diagnose the root cause, fix it, present the analysis to the developer for approval, and archive everything produced during the process — diagnostic tests, the debug summary, any intermediate findings.

Why archiving matters: during debugging, AI creates targeted tests and accumulates insights at real token cost. If these are discarded, the next similar bug starts from zero. Archiving converts each debugging session into reusable assets. Over time, the high-risk areas of the system naturally accumulate the richest diagnostic coverage, because those are the areas where bugs occur and debugging happens.

Why developer review matters: AI may apply surface-level fixes that make tests pass without addressing the underlying issue. The developer reviews the debug analysis and either approves (proceed to archive) or rejects and provides guidance for deeper investigation. Root cause depth is a human judgment call — AI handles the mechanical narrowing, human provides the critical eye.

### Rule 3: Record intent and anticipated impact for behavior-altering changes.

This is preventive. Git diff shows what changed but not why. When debugging crosses session boundaries, understanding the intent behind a past change is what connects a symptom to its cause. Without intent records, the AI must reconstruct reasoning from code alone, which is expensive and error-prone.

This rule is auxiliary, not primary. Decision logs cannot capture unforeseen side effects — the most insidious bugs come from impacts nobody anticipated. The decision log accelerates diagnosis when the relevant information was captured, but it is not a replacement for the testing system. It reduces the probability and cost of bugs, not eliminates them.

What constitutes a "behavior-altering change" is project-specific. At initialization, AI should define the criteria for the current project based on its architecture, configuration mechanisms, and module boundaries. The guiding principle is: if another AI in a future session, unaware of this change, might misunderstand the system's behavior because of it — log it.

---

## 3. Activation

This methodology is activated in one of two ways:

**Manual trigger**: The developer explicitly asks AI to apply this methodology to the current project.

**Automatic trigger**: The developer has installed this methodology as a Claude Skill or added it to the project's CLAUDE.md. Every new development session automatically operates under these rules without explicit request.

On activation, AI performs one check: does the project already have asset infrastructure (conventions document and index)?

**No (first time)** → Run the initialization protocol (Section 4). This is the only complex step — analyze the project, create the asset directory, conventions, initial e2e test, and index.

**Yes (continuing project)** → Read the conventions document and index. If continuing previous work, read relevant recent decision log entries and debug summaries. Then proceed with development — the three rules apply automatically.

---

## 4. Initialization

When this methodology is first applied to a project, AI sets up the infrastructure that all subsequent sessions will use. The initialization produces project-specific conventions — once established, they become the fixed standard for that project.

### 4.1 Analyze the Project

Before creating anything, understand the project:

- Identify major modules and their boundaries
- **Understand the end user's real experience**: Who is the end user? What do they actually do? What input do they provide? What output do they expect? What would make them say "this works" or "this is broken"? This is the foundation for all test design.
- Identify the core user-facing path (the "happy path" that must always work)
- Identify configuration mechanisms (env vars, config files, feature flags)
- Note existing naming patterns and coding conventions
- For brownfield projects: note areas with no test coverage and known fragile points

### 4.2 Create the Asset Infrastructure

Create an asset directory at the project root. The specific structure — directory names, subdivision, organization — should be designed by AI based on the project's needs and conventions. What must exist:

- A place for the e2e test and diagnostic tests (separated, since they serve different purposes)
- A place for test run baselines (successful results) and debug summaries (failure investigations), stored separately so AI can quickly scan failures and compare against known-good states
- A place for decision log entries
- A conventions document that defines naming rules, decision log criteria, templates, and test commands for this project
- A master index that catalogs all assets for quick lookup

The conventions document is critical. It must define: how files are named (so any future session can find assets predictably), what qualifies as a behavior-altering change for this project (so decision log entries are consistent), and what templates to use for debug summaries and decision entries. These conventions should fit the project's existing patterns — a Python project might organize differently from a Node project.

Why this must be fixed after initialization: if AI names and organizes assets differently each session, the archive becomes unsearchable. The conventions document is the contract that keeps all future sessions consistent.

### 4.3 Create the Initial End-to-End Test

Before writing any test code, revisit the user experience answers from §4.1. Design the test to replicate what a real user would do:

- Uses representative input that reflects what real users actually provide
- Exercises the full path the user would experience, with real processing (not mocked)
- Verifies the output is what a user would expect — not just "did it complete" but "would the user say this works?"
- Prioritize realism over speed — a slow test that catches real bugs is worth more than a fast test that only catches crashes

For greenfield projects, this test starts minimal and grows with the project. For brownfield projects, this test should cover the most critical user experience path and may take a session or two to design properly. The test itself is the first and most valuable asset.

### 4.4 Initialize the Index

Create the master index with the initial e2e test entry. The index format should be defined in the conventions document. It needs to support quick lookup by module, by date, and by status. AI designs the format based on the project's anticipated scale.

---

## 5. Normal Development

Once activated and initialized, AI develops normally. The methodology does not prescribe how to write code or implement features. The three rules operate in the background:

- After each code change, run the e2e test (Rule 1). If it passes, save a lightweight baseline record (which tests passed, key output snapshots), log the decision if the change qualifies (Rule 3), and commit. If it fails, enter the debug workflow (Section 6, Rule 2).
- When adding new functionality, extend the e2e test to cover it. If the e2e test grows too slow, split it — keep a fast gate test for the critical path, run the full suite less frequently.

That's it. The methodology adds testing, logging, and archiving to the normal development process. Everything else is standard development.

---

## 6. Debug Workflow

This workflow activates when any test fails after a code change.

### 6.1 Initial Assessment

1. Read the test failure output — note the specific error and which assertion failed
2. Check the decision log for recent changes that might relate to the failure
3. If the change was just made in the current session, the cause is likely the most recent change — check there first

In incremental development, most bugs are caught immediately after the change that caused them. The direct cause is usually the root cause because the change was just made. Deep root cause analysis is primarily needed for brownfield systems or bugs that span multiple changes.

### 6.2 Narrowing Down

If the cause isn't obvious:

1. Check the index for archived diagnostic tests related to the failing area — run them first (zero token cost to create, instant diagnostic value)
2. If no relevant archived tests exist, design targeted tests for suspected modules (AI is good at this — it's a focused, single-module task)
3. Run targeted tests to identify which module or boundary is broken
4. Archive any newly designed tests immediately — don't wait until after the fix

### 6.3 Diagnosis

Once the problem area is identified:

1. Examine the relevant code
2. Cross-reference with decision log entries for recent changes to this module or its dependencies
3. Compare current failure against the last passing baseline — what outputs differ? This delta often points directly to the broken component.
4. Form a hypothesis, verify with a targeted test

For brownfield systems without decision log history, diagnosis relies on iterative testing. Design a test, run it, narrow down, design a more targeted test, archive everything. If one session cannot solve it, the archived tests and partial findings carry forward to the next. Each session strictly reduces the search space.

### 6.4 Fix and Verify

1. Apply the fix
2. Run the e2e test to confirm
3. Run archived diagnostic tests for the affected area to check for regressions
4. If the fix itself is a behavior-altering change, record a decision log entry

### 6.5 Human Review

Present the analysis to the developer:

- What symptom was observed
- What was the direct cause
- Why this cause was able to exist (at least one level of "why" deeper)
- What was fixed
- Whether a structural prevention is recommended

The developer approves or rejects. This is the quality gate for root cause depth. The REDIS_URL incident illustrates why: the surface fix (switching dispatch mode) resolved the symptom, but only human persistence led to the root cause (silent behavior change from env var with no validation).

### 6.6 Archival

After developer approval:

1. Write the debug summary per the project's template
2. Archive all newly created diagnostic tests with metadata
3. Update the index
4. If the investigation revealed a gap in the e2e test, extend it

---

## 7. Asset Management

### 7.1 Why Management Matters

The toolkit's value depends entirely on asset quality. Poorly organized or stale assets waste tokens and can mislead diagnosis. AI has inherent randomness in naming and organizing — the conventions established at initialization are the constraint that prevents quality degradation.

### 7.2 Test Archive

Tests are a byproduct of debugging, not a separate development effort. The archive grows organically, concentrated where bugs actually occur — high-risk areas naturally accumulate the most diagnostic tests.

When designing tests during debugging, consider what level of assertion provides the best trade-off between stability and diagnostic value for the situation: structural assertions (output shape) are stable but less diagnostic; quantitative assertions (counts, sizes in expected ranges) are moderately stable with good diagnostic value; content assertions (output relates to input domain) are the most diagnostic but most fragile. Record this in test metadata so future sessions know what to expect.

### 7.3 Staleness and Lifecycle

When a module's interface changes, related tests may become stale. The approach is lazy maintenance: mark affected tests as needing review in the index, but don't update them immediately. Update when a test is actually needed during a future debugging session. This avoids wasting tokens on tests that may never be used again.

When a module is removed, mark its tests as deprecated. Periodically clean up deprecated entries to keep the index manageable.

Decision log entries and debug summaries are append-only — never modified after creation. If a decision is reversed, write a new entry referencing the old one.

### 7.4 Index Maintenance

The index must be updated whenever assets are added, modified, or change status. It should remain small enough to read in full at session start. If it grows too large, split into sub-indexes. The conventions document should specify when and how to split, based on the project's expected scale.

### 7.5 Incremental Refinement

The conventions document itself can evolve. If the AI or developer identifies that the decision log criteria are too broad or too narrow for this project, or that the naming scheme doesn't work well in practice, update the conventions document. But changes should be deliberate and documented — not silently drifted.

---

## 8. Human-AI Interaction

### 8.1 When AI Involves the Developer

**Debug review**: After every bug fix, present the analysis for approval. The developer decides if the fix is genuine and the analysis deep enough.

**Extended debugging**: If AI cannot resolve a bug in the current session, present findings, archive everything, and inform the developer. The next session continues from archived state.

**Architecture decisions**: When debugging reveals a systemic design issue, surface it to the developer rather than making architectural changes autonomously.

### 8.2 When AI Acts Autonomously

Running and designing tests, writing decision log entries, archiving assets, extending the e2e test, maintaining the index — all routine operations that follow the established conventions.

### 8.3 Developer Override

The developer can always reject analysis, provide hints, modify assets directly, override the gate for exceptional cases, or request asset quality review and cleanup.

---

## 9. Scenarios

### 9.1 Greenfield (From Day 1)

Initialize on day one. Each session develops incrementally: write code, extend test, run test, log decisions, commit. Bugs are caught immediately, and the cause is almost always the most recent change. Assets accumulate naturally. Near single-session bug-free development from early on.

### 9.2 Brownfield (Mid-Way Adoption)

Initialize by analyzing the existing system and writing an initial e2e test. First bug encounters are slower — no archived tests, no decision log history. AI designs diagnostic tests from scratch, archives everything. Subsequent sessions read previous findings, design more targeted tests, narrow further. Each session accumulates assets and reduces the search space for future bugs.

High-risk areas naturally build the richest diagnostic coverage because that's where bugs and debugging concentrate. After several cycles, the system approaches greenfield-level diagnostic efficiency.

For unknown systems requiring complex debugging: the iterative test-narrow-archive cycle works across sessions. Each session's archived results become the starting point for the next. The problem strictly shrinks because assets persist even when conversational context doesn't.

---

## 10. Assumptions and Limitations

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

## 11. Quick Reference

### The Three Rules
1. Every change must pass the e2e test before commit
2. When a test fails: debug → fix → get developer approval → archive all artifacts
3. Record intent and impact for behavior-altering changes

### Activation
1. Check: does asset infrastructure exist?
2. No → initialize (Section 4)
3. Yes → read conventions + index → develop normally with rules applied

### Debug Cycle
1. Read failure, check decision log
2. Run archived tests or design new ones to narrow down
3. Diagnose, fix, verify with e2e test
4. Present analysis to developer
5. Archive: summary + tests + update index

### Post-Fix Checklist
- [ ] E2e test passes
- [ ] Debug summary written
- [ ] New diagnostic tests archived
- [ ] Index updated
- [ ] Developer review obtained

### Post-Feature Checklist
- [ ] E2e test extended
- [ ] All tests pass
- [ ] Decision log entry written (if applicable)
- [ ] Index updated
