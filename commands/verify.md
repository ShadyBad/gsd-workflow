---
description: "GSD Verify — Evidence pack generation and Done Gate evaluation. Produces DONE or NOT_DONE with ranked next actions. Final phase."
argument-hint: "<run_id>"
allowed-tools: Bash, Read, Write
---

# /verify — Evidence Pack + Done Gate

You are running the Verify phase of the GSD Workflow Engine.

Run ID: **$ARGUMENTS**

## Step 1: Load State

Read:
- `runs/$ARGUMENTS/intake/track_decision.json` — determine MICRO / STANDARD / FULL
- `runs/$ARGUMENTS/intake/acceptance_criteria.md`
- `runs/$ARGUMENTS/execute/criteria_status.md`

Write audit event:
```json
{"event": "verify_started", "run_id": "$ARGUMENTS", "timestamp": "<ISO8601>"}
```

Create evidence directory: `runs/$ARGUMENTS/final/evidence/`

## Step 2: Run Verification Suite

Detect the project toolchain and run the appropriate commands. Always check `CLAUDE.md` first — it overrides auto-detection.

| Detect | Lint | Type Check | Test |
|---|---|---|---|
| `package.json` | `npm run lint` | `npm run type-check` | `npm test` |
| `pyproject.toml` | `uv run ruff check .` or `make lint` | `uv run mypy .` (if configured) | `uv run pytest` or `make test` |
| `Cargo.toml` | `cargo clippy` | N/A (compiled) | `cargo test` |
| `Makefile` | `make lint` | `make type-check` (if exists) | `make test` |
| `go.mod` | `go vet ./...` | N/A (compiled) | `go test ./...` |

Capture ALL output to log files. Append exit code to each:
```bash
echo "EXIT=$?" >> runs/$ARGUMENTS/final/<step>.log
```

If a command doesn't exist, document why in a `<step>_waiver.md` file.

## Step 3: Secrets Scan

Scan all artifacts and source for leaked secrets:

```bash
# Scan for common secret patterns
grep -r "sk-[a-zA-Z0-9]\+" runs/$ARGUMENTS/ src/ --include="*.json" --include="*.log" --include="*.md" | grep -v "[REDACTED]" > /tmp/secret_scan.txt 2>&1
grep -r "password=" runs/$ARGUMENTS/ src/ >> /tmp/secret_scan.txt 2>&1
grep -r "token=" runs/$ARGUMENTS/ src/ >> /tmp/secret_scan.txt 2>&1
grep -r "api_key" runs/$ARGUMENTS/ src/ >> /tmp/secret_scan.txt 2>&1
```

Write results to `runs/$ARGUMENTS/final/redaction_scan.json`:
```json
{
  "findings": [],
  "clean": true,
  "scanned_paths": ["runs/$ARGUMENTS/", "src/"],
  "timestamp": "<ISO8601>"
}
```

If findings > 0: **STOP. Redact before proceeding.**

## Step 4: Build Evidence Pack

Collect proof for every Done Gate check.

Write `runs/$ARGUMENTS/final/verification.md`:
```markdown
# Verification Report

## Run ID: $ARGUMENTS
## Track: MICRO | STANDARD | FULL
## Timestamp: <ISO8601>

## Gate Results

| Gate | Status | Proof |
|---|---|---|
| Scope locked | ✓ PASS / ✗ FAIL | runs/.../intake/scope.md |
| Acceptance criteria | ✓ PASS / ✗ FAIL | runs/.../intake/acceptance_criteria.md |
| Lint | ✓ PASS / ✗ FAIL | runs/.../final/lint.log |
| Type check | ✓ PASS / ✗ FAIL | runs/.../final/typecheck.log |
| Unit tests | ✓ PASS / ✗ FAIL | runs/.../final/unit_tests.log |
| No secrets | ✓ PASS / ✗ FAIL | runs/.../final/redaction_scan.json |
| Audit trail | ✓ PASS / ✗ FAIL | runs/.../audit/events.jsonl |
| Scope changes tracked | ✓ PASS / N/A | docs/scope-changes/ |
<FULL track gates if applicable>

## Acceptance Criteria
<paste from criteria_status.md>
```

## Step 5: FULL Track — Integration Tests

If FULL track, run integration tests using the project's toolchain (check `CLAUDE.md` first, then detect from project files). Capture output to `runs/$ARGUMENTS/final/integration_tests.log`.

If integration tests don't exist yet: create `runs/$ARGUMENTS/final/integration_test_waiver.md` explaining why and when they'll be added.

## Step 6: Done Gate Evaluation

Read `done_gate.yaml`. Evaluate every gate against evidence collected.

Write `runs/$ARGUMENTS/final/done_gate.json`:
```json
{
  "run_id": "$ARGUMENTS",
  "track": "MICRO | STANDARD | FULL",
  "timestamp": "<ISO8601>",
  "status": "DONE | NOT_DONE",
  "gates": {
    "scope_locked": {"status": "PASS | FAIL", "proof": "<path>"},
    "acceptance_criteria": {"status": "PASS | FAIL", "proof": "<path>"},
    "lint_pass": {"status": "PASS | FAIL", "exit_code": 0},
    "type_check_pass": {"status": "PASS | FAIL", "exit_code": 0},
    "unit_tests_pass": {"status": "PASS | FAIL", "exit_code": 0},
    "no_secrets_in_artifacts": {"status": "PASS | FAIL", "findings": 0},
    "audit_trail_exists": {"status": "PASS | FAIL", "proof": "<path>"}
  },
  "failed_gates": [],
  "next_actions": []
}
```

## Step 7: Final Output

### If DONE:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ DONE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Run ID:  $ARGUMENTS
Track:   MICRO | STANDARD | FULL
Gates:   <N>/<N> passed

Evidence: runs/$ARGUMENTS/final/
Audit:    runs/$ARGUMENTS/audit/events.jsonl

Ready to commit / ship.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### If NOT_DONE:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✗ NOT DONE
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
