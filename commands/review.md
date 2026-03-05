---
description: "GSD Weekly Review (Phase R) — Prune dead artifacts, aggregate lessons, trigger self-improvement, add golden cases from failures, close security exceptions, cut cost creep. Run weekly or after every 5 completed runs."
allowed-tools: Bash, Read, Write, Grep, Glob
---

# /review — Weekly Phase R

You are running the Review phase of the GSD Workflow Engine.

## Step 1: Inventory Runs

List all runs in `runs/` directory. Count:
- Total runs
- Completed (have `final/done_gate.json` with `"status": "DONE"`)
- Failed / abandoned (no done_gate or `"status": "NOT_DONE"`)

## Step 2: Failure Analysis

For every NOT_DONE run, read `runs/<id>/final/done_gate.json`.

Aggregate: Which gates fail most often?

Write `runs/review-<timestamp>/failure_patterns.md`:
```markdown
# Failure Patterns

| Gate | Failure Count | Runs |
|---|---|---|
| <gate_name> | N | <run_ids> |

## Root Causes
<analysis of why these gates are failing repeatedly>

## Systemic Fixes
<what to change in CLAUDE.md, hooks, capabilities.yaml, or workflow to prevent recurrence>
```

## Step 3: Golden Cases

For every failure pattern, create a golden test case.

Append to `evals/golden_cases.jsonl`:
```jsonl
{"id": "gc-<timestamp>-001", "input": "<what triggered the failure>", "expected_behavior": "<what should have happened>", "gate": "<which gate failed>", "source_run": "<run_id>"}
```

## Step 4: Security Exception Review

Scan for open security findings:
```bash
find runs/ -path "*/security/*" -name "*.md" -o -name "*.json" | head -50
```

For each open finding:
- Still relevant? → Keep, schedule fix
- Mitigated? → Close with evidence
- False positive? → Document and close

Write `runs/review-<timestamp>/security_exceptions.md`.

## Step 5: Committee Effectiveness

If any FULL track runs used agent committees:
- Did committee findings prevent implementation issues?
- Were there false positives (concerns raised that turned out to be non-issues)?
- Were there false negatives (issues missed that should have been caught)?

Write `runs/review-<timestamp>/committee_effectiveness.md`.

## Step 6: Cost Creep Audit

Count agent spawns across recent runs:
```bash
grep '"agents_spawned"' runs/*/metrics.json 2>/dev/null
```

Flag any run that exceeded budget limits from `capabilities.yaml`. Assess necessity.

Identify: Which phases produce the most artifact bloat?

Write `runs/review-<timestamp>/cost_analysis.md`.

## Step 7: Lessons & Self-Improvement

### 7a — Aggregate Lessons
Read `runs/lessons.jsonl`. Group by `type`. Identify recurring patterns (3+ occurrences).

For each recurring pattern:
- Determine if a systemic fix exists
- Check if already addressed in `runs/improvements.md`
- If not, add a new improvement proposal

### 7b — Review Pending Improvements
Read `runs/improvements.md`. For any `Status: PROPOSED`:
- Is the evidence strong enough to apply? (3+ occurrences, clear root cause)
- Present to user for approval
- If approved, apply and mark `Status: APPLIED`

### 7c — CLAUDE.md Health Check

Review current `CLAUDE.md`. Ask:
1. Are the key project commands still accurate?
2. Are there new patterns or conventions to add from lessons?
3. Is the redaction list complete?
4. Are any sections outdated?
5. Do `capabilities.yaml` budgets need adjustment?
6. Does `done_gate.yaml` need new gates or threshold changes?

If updates needed: edit directly and note changes.

## Step 8: Cleanup

Archive or delete abandoned runs older than 30 days:
```bash
find runs/ -maxdepth 1 -type d -mtime +30 | head -20
```

Do NOT delete runs with `"status": "DONE"` unless explicitly requested.

## Step 9: Summary Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WEEKLY REVIEW COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Runs Analyzed:    <N>
  DONE:           <N>
  NOT_DONE:       <N>
  Abandoned:      <N>

Golden Cases Added: <N>
Security Exceptions: <N> open / <N> closed
Committee Effectiveness: <summary>
Improvements Applied: <N> / Proposed: <N>
CLAUDE.md Updates: <N> changes

Top Failure Gate: <gate_name> (<N> failures)
Top Fix: <what to do>

Report: runs/review-<timestamp>/
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
