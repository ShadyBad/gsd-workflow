---
description: "GSD Triage — Start here for manual phased workflow. Captures scope, selects MICRO/STANDARD/FULL track, locks acceptance criteria. Required before /plan or /execute. (Use /ship instead for fully autonomous flow.)"
argument-hint: "<feature/change description>"
allowed-tools: Bash, Read, Write, Grep, Glob
---

# /triage — Intake & Track Selection

You are running the Triage phase of the GSD Workflow Engine.

## Step 1: Generate Run ID

Create a run ID in this format: `YYYYMMDD-HHMMSS-<slug>` where slug is a 3-word kebab-case summary of the work.

Create the run directory:
```
runs/<run_id>/intake/
runs/<run_id>/audit/
```

Write the first audit event:
```json
{"event": "run_started", "run_id": "<run_id>", "command": "triage", "input": "$ARGUMENTS", "timestamp": "<ISO8601>"}
```
to `runs/<run_id>/audit/events.jsonl`

## Step 2: Scope Capture

Using the input: **$ARGUMENTS**

Before asking questions, search the codebase with Grep and Glob to understand existing code, patterns, and conventions relevant to the task.

Ask the following clarifying questions IF the input is ambiguous. If input is clear, use smart defaults and proceed.

**Clarifying questions (ask max 3, only if truly unclear):**
1. What is the expected outcome / user-facing behavior change?
2. What systems/files/APIs are definitely in scope?
3. Are there hard constraints (deadline, performance, compatibility)?

Write `runs/<run_id>/intake/scope.md`:
```markdown
# Scope: <title>

## Summary
<1-2 sentence plain English description>

## In Scope
- <explicit inclusions>

## Out of Scope
- <explicit exclusions>

## Constraints
- <hard constraints>

## Run ID: <run_id>
## Locked: <timestamp>
```

## Step 3: Track Selection

Evaluate against track signals. Check each:

### MICRO signals (fast path):
- Fix a typo / rename / cosmetic change
- Add/update a single config value
- Update documentation only
- Fix an obvious bug with a clear, localized fix (1-3 files)

### STANDARD signals (needs planning):
- New feature touching 3+ files
- Refactor of a module or component
- Adding a new API endpoint
- New UI component or page

### FULL signals (security/compliance/high-risk):

| Trigger | Present? |
|---|---|
| Auth / permissions touched | Yes / No |
| Payments / billing touched | Yes / No |
| User data handling / PII | Yes / No |
| Secrets / credentials | Yes / No |
| New external integrations | Yes / No |
| DB schema changes | Yes / No |
| Security-sensitive code paths | Yes / No |

**Decision:**
- 0 FULL triggers + small scope → **MICRO track**
- 0 FULL triggers + multi-file scope → **STANDARD track**
- 1+ FULL triggers → **FULL track**
- Ambiguous → always choose the more rigorous track

Write `runs/<run_id>/intake/track_decision.json`:
```json
{
  "run_id": "<run_id>",
  "track": "MICRO | STANDARD | FULL",
  "triggers_found": ["<list>"],
  "rationale": "<1 sentence>"
}
```

## Step 4: Acceptance Criteria

Write clear, testable acceptance criteria. Every criterion must be verifiable.

Format: Given / When / Then OR bullet checklist. No vague language ("works correctly", "looks good").

Write `runs/<run_id>/intake/acceptance_criteria.md`:
```markdown
# Acceptance Criteria: <title>

## Criteria

- [ ] <specific, testable criterion 1>
- [ ] <specific, testable criterion 2>
- [ ] <specific, testable criterion 3>

## Definition of Done
All criteria checked. Done Gate passes. No open blockers.
```

## Step 5: Next Actions

Write `runs/<run_id>/intake/next_actions.md` with:
- The track selected (MICRO, STANDARD, or FULL)
- Ordered phases to run
- Exact commands to run next
- Any pre-conditions or decisions needed before proceeding

## Step 6: Summary Output

Print to terminal:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TRIAGE COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Run ID:  <run_id>
Track:   MICRO | STANDARD | FULL
Scope:   <1-line summary>

Acceptance Criteria: <N> defined

Next:    /plan <run_id>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Write final audit event:
```json
{"event": "triage_complete", "run_id": "<run_id>", "track": "<track>", "criteria_count": N, "timestamp": "<ISO8601>"}
```
