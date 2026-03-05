---
description: "GSD Verify — Evidence pack generation and Done Gate evaluation. Produces DONE or NOT_DONE with ranked next actions. Final phase."
argument-hint: "<run_id>"
allowed-tools: Bash, Read, Write, Grep, Glob
---

# /verify — Evidence Pack + Done Gate

You are running the Verify phase of the GSD Workflow Engine.

Run ID: **$ARGUMENTS**

## Step 1: Load State

Read:
- `runs/$ARGUMENTS/intake/track_decision.json` — determine MICRO / STANDARD / FULL
- `runs/$ARGUMENTS/intake/acceptance_criteria.md`
- `runs/$ARGUMENTS/execute/criteria_status.md`
- `capabilities.yaml` — capability limits
- `done_gate.yaml` — gate definitions for the active track

Write audit event:
```json
{"event": "verify_started", "run_id": "$ARGUMENTS", "timestamp": "<ISO8601>"}
```

Create evidence directory: `runs/$ARGUMENTS/final/evidence/`

## Step 2: Run Verification Suite

Detect the project toolchain and run the appropriate commands. Always check `CLAUDE.md` first — it overrides auto-detection.

| Detect | Lint | Type Check | Test | Audit |
|---|---|---|---|---|
| `package.json` | `npm run lint` | `npm run type-check` | `npm test` | `npm audit --json` |
| `pyproject.toml` | `uv run ruff check .` or `make lint` | `uv run mypy .` (if configured) | `uv run pytest` or `make test` | `uv run pip-audit` |
| `Cargo.toml` | `cargo clippy` | N/A (compiled) | `cargo test` | `cargo audit` |
| `Makefile` | `make lint` | `make type-check` (if exists) | `make test` | N/A |
| `go.mod` | `go vet ./...` | N/A (compiled) | `go test ./...` | `govulncheck ./...` |

Capture ALL output to log files. Append exit code to each:
```bash
echo "EXIT=$?" >> runs/$ARGUMENTS/final/<step>.log
```

If a command doesn't exist, document why in a `<step>_waiver.md` file.

## Step 3: Secrets Scan

Scan all artifacts and source for leaked secrets. Write `runs/$ARGUMENTS/final/redaction_scan.json`:
```json
{
  "findings": [],
  "clean": true,
  "scanned_paths": ["runs/$ARGUMENTS/", "src/"],
  "timestamp": "<ISO8601>"
}
```

If findings > 0: **STOP. Redact before proceeding.**

## Step 4: FULL Track — Evals

If FULL track and `evals/golden_cases.jsonl` exists:
- Run all golden cases
- Compare against baselines
- Write `runs/$ARGUMENTS/evals/eval_report.json`

## Step 5: FULL Track — Integration Tests

If FULL track, run integration tests using the project's toolchain (check `CLAUDE.md` first). Capture output to `runs/$ARGUMENTS/final/integration_tests.log`.

If integration tests don't exist: create `runs/$ARGUMENTS/final/integration_test_waiver.md`.

## Step 6: Build Evidence Pack

Write `runs/$ARGUMENTS/final/evidence_manifest.md`:

```markdown
# Evidence Pack — $ARGUMENTS

## Track: MICRO | STANDARD | FULL
## Timestamp: <ISO8601>

| Gate | Status | Proof |
|---|---|---|
| scope_locked | PASS/FAIL | runs/.../intake/scope.md |
| acceptance_criteria | PASS/FAIL | runs/.../execute/criteria_status.md |
| lint | PASS/FAIL | runs/.../final/lint.log |
| type_check | PASS/FAIL/N/A | runs/.../final/typecheck.log |
| tests | PASS/FAIL | runs/.../final/tests.log |
| no_secrets | PASS/FAIL | runs/.../final/redaction_scan.json |
| audit_trail | PASS/FAIL | runs/.../audit/events.jsonl |
| scope_changes | PASS/N/A | runs/.../execute/scope_flags.md |
```

For FULL track, also include:
```markdown
| committee_review | PASS/FAIL | runs/.../committee/review_summary.md |
| threat_model | PASS/FAIL | runs/.../security/threat_model.md |
| security_scan | PASS/FAIL | runs/.../security/scan_results.json |
| privacy_review | PASS/N/A | runs/.../security/privacy_review.md |
| evals | PASS/FAIL/N/A | runs/.../evals/eval_report.json |
| integration_tests | PASS/FAIL/WAIVER | runs/.../final/integration_tests.log |
| observability | PASS/FAIL | runs/.../timeline.json |
| hardening | PASS/FAIL | runs/.../hardening/failure_modes.md |
| legal | PASS/FAIL/N/A | runs/.../legal/license_review.md |
```

## Step 7: Done Gate Evaluation

Read `done_gate.yaml`. Evaluate every gate for the active track against evidence.

Write `runs/$ARGUMENTS/final/done_gate.json`:
```json
{
  "run_id": "$ARGUMENTS",
  "track": "MICRO | STANDARD | FULL",
  "timestamp": "<ISO8601>",
  "status": "DONE | NOT_DONE",
  "gates": {
    "<gate_id>": {"status": "PASS | FAIL | N/A", "proof": "<path>"}
  },
  "failed_gates": [],
  "next_actions": []
}
```

## Step 8: Final Output

### If DONE:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DONE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Run ID:  $ARGUMENTS
Track:   MICRO | STANDARD | FULL
Gates:   <N>/<N> passed

Evidence: runs/$ARGUMENTS/final/
Audit:    runs/$ARGUMENTS/audit/events.jsonl

Ready to commit / ship.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### If NOT DONE:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NOT DONE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Run ID:  $ARGUMENTS

FAILED GATES (ranked by blocking priority):
1. <gate> — <what to do>
2. <gate> — <what to do>

Fix blockers and re-run: /verify $ARGUMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Write final audit event:
```json
{"event": "verify_complete", "run_id": "$ARGUMENTS", "status": "DONE|NOT_DONE", "failed_gates": [], "timestamp": "<ISO8601>"}
```
