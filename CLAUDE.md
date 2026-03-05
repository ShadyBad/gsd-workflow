# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is the **GSD Workflow Engine** — a Claude Code plugin (`gsd-ship` v1.0.0) that provides an autonomous development lifecycle. It is NOT a traditional application. The primary consumers are Claude Code instances running the workflow commands.

The engine implements a Superpowers-style workflow that turns any request into provably DONE work, using bounded capabilities (repo access, browser, crawling, agent committees) with strict security, budgets, audit trails, and regression evals.

## Non-Negotiables

- Deny-by-default capabilities (see `capabilities.yaml`)
- Secrets are handle-only (never readable/printed)
- Evidence-based verification required for DONE (see `done_gate.yaml`)
- Budgets and spawn limits enforced
- Everything produces structured artifacts per run

## Plugin Architecture

```
.claude-plugin/       # Plugin metadata (plugin.json, marketplace.json)
capabilities.yaml     # Deny-by-default capability manifest
done_gate.yaml        # Completion contract — what DONE means
commands/             # Slash command definitions (markdown prompt files)
hooks/                # PreToolUse / PostToolUse hook scripts
runs/                 # Runtime artifacts — one directory per workflow run
evals/                # Golden cases, rubrics, baselines
docs/                 # Architecture, security, legal docs
```

### Slash Commands

| Command | Purpose | Prerequisite |
|---|---|---|
| `/ship <task>` | **Main entry point.** Clarifies intent, classifies complexity (MICRO/STANDARD/FULL), runs all applicable phases (0-10), drives to DONE with agent committees, security, legal, evals, and learning. | None |
| `/triage <task>` | Manual intake — scope, track selection (MICRO/STANDARD/FULL), acceptance criteria. | None |
| `/plan <run_id>` | Architecture, interfaces, error taxonomy, implementation checklist. | `/triage` |
| `/execute <run_id>` | Implementation with vertical-slice-first strategy. | `/plan` |
| `/verify <run_id>` | Evidence pack, Done Gate evaluation, DONE/NOT_DONE verdict. | `/execute` |
| `/scope-change <run_id> <desc>` | Documents mid-work scope drift. Blocks until recorded. | Active run |
| `/review` | Weekly housekeeping: lessons, self-improvement, failure patterns, security exceptions, cost audit. | Completed runs |

### Execution Tracks

| Track | When | Phases Run |
|---|---|---|
| **MICRO** | Typos, config, docs, obvious 1-3 file fixes | Clarify → Orient → Execute → Verify → Learn |
| **STANDARD** | Features, refactors, endpoints, components (no security/legal triggers) | Clarify → Orient → Intake → Plan → Execute → Verify → Learn |
| **FULL** | Auth, payments, PII, secrets, integrations, schema changes, architecture | All phases: 0 → 1 → 1.5 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → Learn |

STANDARD is only allowed if NOT touching: auth, payments, PII, secrets, permissions, new integrations.

### Full Track Phases (0-10)

| Phase | Name | Key Output |
|---|---|---|
| 0 | Setup + Guardrails | Budget verification, audit system, directory structure |
| 1 | Intake | Scope, acceptance criteria (user-confirmed), track decision |
| 1.5 | Agent Committee | Architecture, security, quality, legal reviews from role-locked agents |
| 2 | Architecture + Interfaces | Architecture doc, interface specs, error taxonomy, impl checklist |
| 3 | Security + Privacy | Threat model, data classification, privacy review, redaction verification |
| 4 | Evals + Regression | Golden cases, rubric, baseline, eval plan |
| 5 | Implementation | Vertical slice first, feature-flagged, artifact generation |
| 6 | Observability + Cost | Timeline, metrics, budget review |
| 7 | Hardening | Failure modes, recovery guide, rollback procedure |
| 8 | Legal/Compliance | License scan, privacy check, compliance checklist |
| 9 | Change Control | Scope drift detection, impact assessment |
| 10 | Verify + Done Gate | Evidence pack, eval run, secrets scan, done_gate.yaml evaluation |

### Agent Committee (Phase 1.5)

The FULL track spawns role-locked sub-agents to review the approach before implementation:

- **Architecture Reviewer** — design soundness, alternatives, interfaces
- **Security Reviewer** — attack vectors, trust boundaries, data classification
- **Quality Reviewer** — test strategy, edge cases, criteria completeness
- **Legal Reviewer** (conditional) — licenses, privacy, compliance

Agents are budget-limited per `capabilities.yaml` (max 3 per phase, 15 per run).

### Hooks System

Every tool invocation is intercepted by hooks defined in `hooks/hooks.json`:

- **`pre-tool-guard.sh`** (PreToolUse) — Blocks secrets in tool input, enforces command denylist, blocks reads of sensitive files. Logs all allowed/blocked actions.
- **`post-tool-audit.sh`** (PostToolUse) — Writes audit trail to `runs/<run_id>/audit/events.jsonl`, redacts secrets from logged output, tracks test-gate status for commits.

### Config Files

- **`capabilities.yaml`** — Deny-by-default capability manifest. Defines what the system is allowed to do: repo access, browser, crawl, command exec, agent spawning, CI reads. Includes budgets, allowlists, and redaction rules.
- **`done_gate.yaml`** — The completion contract. Defines what DONE means for each track. All gates must pass for DONE status.

### Run Directory Convention

Each workflow run creates `runs/<run_id>/` with this structure:
```
runs/<YYYYMMDD-HHMMSS-three-word-slug>/
  intake/        # scope.md, acceptance_criteria.md, criteria_confirmed.md, track_decision.json
  committee/     # architecture_review.md, security_review.md, quality_review.md, review_summary.md
  plan/          # architecture.md, interfaces.md, error_taxonomy.md, impl_checklist.md
  security/      # threat_model.md, data_classification.md, privacy_review.md, scan_results.json
  evals/         # eval_plan.md, eval_report.json
  execute/       # slice.md, progress.md, criteria_status.md, scope_flags.md
  observability/ # (FULL only)
  hardening/     # failure_modes.md, recovery_guide.md
  legal/         # license_review.md, compliance_checklist.md
  final/         # done_gate.json, evidence_manifest.md, lint/test/typecheck logs
  audit/         # events.jsonl (append-only audit trail)
```

Cross-run persistent files:
```
runs/lessons.jsonl       # Append-only lessons from every run
runs/improvements.md     # Self-improvement proposals (PROPOSED → APPLIED)
```

## Development Setup

Python 3.13.5+, managed with `uv`:

```bash
uv sync                    # Install dependencies
uv run python main.py      # Run (placeholder entry point)
```

## Key Rules Enforced by the Workflow

- **Secrets:** Never print, log, or embed. Handle by reference only. Redact with `[REDACTED:<type>]`.
- **Capabilities:** Every privileged action must pass: enabled + allowlist + budget + phase-allowed. Deny-by-default.
- **Scope creep:** Must be documented via `/scope-change` before implementation continues.
- **Track escalation:** When ambiguous, always choose the more rigorous track (MICRO→STANDARD, STANDARD→FULL).
- **Vertical slice first:** Implement one end-to-end path before expanding.
- **Audit trail:** Every phase transition writes to `events.jsonl`.
- **Toolchain detection:** Never assume npm/node. Detect from project files (`package.json`, `pyproject.toml`, `Cargo.toml`, `Makefile`, `go.mod`). `CLAUDE.md` overrides auto-detection.
- **Agent committee:** FULL track requires committee review before implementation. Agents are role-locked and budget-limited.
- **Learning loop:** Every run captures lessons to `runs/lessons.jsonl`. Every 5 runs, the workflow analyzes patterns and proposes improvements. Safe improvements auto-apply; command/hook changes require user approval.
- **Evidence-based DONE:** DONE requires proof for every gate. NOT_DONE outputs ranked next actions.
