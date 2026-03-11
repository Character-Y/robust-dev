---
name: robust-dev
description: Bug-free product delivery for human-AI collaborative systems. Enforces mandatory e2e testing from the user's perspective, structured debugging with asset archival, and decision logging. Use this when developing, debugging, or maintaining any software project.
user-invocable: true
disable-model-invocation: true
---
 
# Bug-Free Incremental Development

For design rationale, scenario guidance (greenfield vs brownfield), and known limitations, consult `reference.md` in this skill directory when facing non-routine situations.

## The Three Rules
 
**Rule 1: Every change must pass an end-to-end test before commit.**
An e2e test must replicate the end user's real experience. Ask: what does the user do? What input do they provide? What do they see? What would make them say "this is broken"? Design the test from those answers. A test that shortcuts any part of the user's real workflow — mock data when users use real data, API calls when users use a browser, toy input when users provide real documents — is an integration test, not e2e — valuable, but insufficient as the gate. Nothing is committed until the e2e test passes.
 
**Rule 2: When a test fails, debug with the structured workflow and archive all artifacts.**
Narrow down the problem using archived or newly designed tests, fix it, present the analysis to the developer for approval, then archive everything — diagnostic tests, debug summary, updated index. Every debugging session becomes a reusable asset.
 
**Rule 3: Record intent and impact for behavior-altering changes.**
After changes that alter system behavior (config changes, new dependencies, interface changes, async behavior, operational mode changes), write a decision log entry capturing what changed, why, and what might be affected. This is auxiliary — it accelerates diagnosis but cannot replace testing.
 
## Activation
 
On activation, check: does this project have asset infrastructure (a conventions document and index)?
 
**No (first time)** → Run initialization:
1. Analyze the project — identify modules, boundaries, core path, config mechanisms, existing conventions
2. **Define the end user's real experience**: Who is the end user? What do they actually do, step by step? Write out the concrete workflow — from what input they provide, through what they see and interact with, to what output they expect. What would make them say "this works" or "this is broken"? Record this workflow in the conventions document — this determines what "e2e" means for this project. The e2e test must then be aligned to this workflow step by step.
3. **Audit existing tests against user reality**: Review all existing tests. For each, ask: does this test replicate what the user actually experiences? Tests that shortcut the user's real workflow (e.g., calling APIs when users use a browser, using mock data when users rely on real processing) are integration tests regardless of their label. Note the gaps.
4. Create an asset directory at the project root. Design the structure based on project needs. What must exist:
   - A place for e2e and diagnostic tests (separated)
   - A place for test run baselines (successful results) and debug summaries (failure investigations), stored separately
   - A place for decision log entries
   - A conventions document defining: naming rules, decision log criteria, templates, test commands, **and what constitutes e2e for this project (grounded in the user experience answers from step 2)**
   - A master index cataloging all assets for quick lookup
5. Design the initial e2e test — replicates the user's core experience with representative input and real processing, verifies the output is what a user would expect (not just non-null)
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
 