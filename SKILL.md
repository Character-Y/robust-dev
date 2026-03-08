---
name: robust-dev
description: Bug-free incremental development for human-AI collaborative systems. Enforces mandatory e2e testing, structured debugging with asset archival, and decision logging. Use this when developing, debugging, or maintaining any software project.
user-invocable: true
disable-model-invocation: true
---
 
# Bug-Free Incremental Development

For design rationale, scenario guidance (greenfield vs brownfield), and known limitations, consult `reference.md` in this skill directory when facing non-routine situations.

## The Three Rules
 
**Rule 1: Every change must pass an end-to-end test before commit.**
An e2e test must exercise the system through the same interface as the end user. If the user operates via a browser, the e2e test must drive a browser. If the user calls a CLI, the test must invoke the CLI. If the user consumes an API, the test must call the API. A test that bypasses the user's interface (e.g., calling backend APIs directly when the user operates via browser) is an integration test, not e2e — valuable, but insufficient as the gate. Nothing is committed until the e2e test passes.
 
**Rule 2: When a test fails, debug with the structured workflow and archive all artifacts.**
Narrow down the problem using archived or newly designed tests, fix it, present the analysis to the developer for approval, then archive everything — diagnostic tests, debug summary, updated index. Every debugging session becomes a reusable asset.
 
**Rule 3: Record intent and impact for behavior-altering changes.**
After changes that alter system behavior (config changes, new dependencies, interface changes, async behavior, operational mode changes), write a decision log entry capturing what changed, why, and what might be affected. This is auxiliary — it accelerates diagnosis but cannot replace testing.
 
## Activation
 
On activation, check: does this project have asset infrastructure (a conventions document and index)?
 
**No (first time)** → Run initialization:
1. Analyze the project — identify modules, boundaries, core path, config mechanisms, existing conventions
2. **Identify the end user's interface**: Who is the end user? How do they interact with the system? (Browser? CLI? API? Mobile app?) This determines what "e2e" means for this project. Record this in the conventions document.
3. **Audit existing tests**: Review all existing tests (including those labeled "e2e"). Classify each as unit, integration, or true e2e based on whether it uses the end user's interface. If tests are misclassified, note the gap — don't assume names are accurate.
4. Create an asset directory at the project root. Design the structure based on project needs. What must exist:
   - A place for e2e and diagnostic tests (separated)
   - A place for test run baselines (successful results) and debug summaries (failure investigations), stored separately
   - A place for decision log entries
   - A conventions document defining: naming rules, decision log criteria, templates, test commands, **and what constitutes e2e for this project**
   - A master index cataloging all assets for quick lookup
5. Design the initial e2e test — exercises the core path through the end user's interface, uses a small fixture, verifies output is reasonable (not just non-null), runs fast
6. Initialize the index with the e2e test entry
 
The conventions document is the contract for consistency. It must be fixed after initialization — if AI names and organizes assets differently each session, the archive becomes unsearchable.
 
**Yes (continuing project)** → Read the conventions document and index. Read relevant recent decision logs and debug summaries if continuing previous work. Then develop normally with the three rules applied.
 
## Normal Development
 
Develop normally. The methodology does not prescribe how to write code. The three rules operate in the background:
 
- After each change, run the e2e test. If it passes, save a lightweight baseline record, log the decision if applicable, and commit. If it fails, enter the debug workflow.
- When adding new functionality, extend the e2e test to cover it. If the e2e test grows too slow, split it — keep a fast gate test for the critical path.
 
## Debug Workflow
 
When any test fails after a code change:
 
**1. Assess** — Read the failure output. Check the decision log for recent related changes. If the change was just made, it's likely the cause — check there first. In incremental development, the direct cause is usually the root cause.
 
**2. Narrow down** — Check the index for archived diagnostic tests related to the failing area and run them. If none exist, design targeted tests for suspected modules. Run them to identify which module or boundary is broken. Archive any newly designed tests immediately.
 
**3. Diagnose** — Examine the relevant code. Cross-reference with decision log entries. Compare the current failure against the last passing baseline — what outputs differ? Form a hypothesis, verify with a targeted test. For brownfield systems without history, use iterative testing across sessions — each session's archived results become the next session's starting point.
 
**4. Fix and verify** — Apply the fix. Run e2e test to confirm. Run archived diagnostic tests to check regressions. If the fix is behavior-altering, record a decision log entry.
 
**5. Human review** — Present to the developer: symptom, direct cause, why this cause existed (at least one level of "why" deeper), what was fixed, whether structural prevention is recommended. Developer approves or rejects for deeper investigation.
 
**6. Archive** — After approval: write debug summary per the project's template, archive all new diagnostic tests with metadata, update the index, extend the e2e test if a gap was revealed.
 
## Asset Management
 
**Test archive**: Tests are a byproduct of debugging, not a separate effort. The archive grows organically, concentrated where bugs actually occur. When designing tests, note the assertion level (structural / quantitative / content-based) in metadata.
 
**Staleness**: When a module's interface changes, mark dependent tests as needs-review in the index. Don't update immediately — update lazily when a test is actually needed. When a module is removed, mark its tests as deprecated.
 
**Decision logs and summaries**: Append-only, never modified after creation. If a decision is reversed, write a new entry referencing the old one.
 
**Index**: Update whenever assets are added or change status. Keep it small enough to read in full at session start. Split into sub-indexes if it grows too large.
 
**Conventions can evolve**: If criteria or naming schemes don't work in practice, update the conventions document deliberately and document the change.
 
## Human-AI Interaction
 
**AI involves the developer for**: debug summary review (developer approves or rejects fix depth), extended debugging (when a bug can't be resolved in one session — archive everything, inform developer), architecture decisions revealed during debugging.
 
**AI acts autonomously for**: running and designing tests, writing decision log entries, archiving assets, extending e2e tests, maintaining the index.
 
**Developer can always**: reject analysis and request deeper investigation, provide hints, modify assets directly, override the gate for exceptional cases.
 