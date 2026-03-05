# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is the **GSD Workflow Engine** — a Claude Code plugin (`gsd-ship` v1.0.0) that provides an autonomous development lifecycle. It is NOT a traditional application. The primary consumers are Claude Code instances running the workflow commands.

## Plugin Architecture

```
.claude-plugin/       # Plugin metadata (plugin.json, marketplace.json)
commands/             # Slash command definitions (markdown prompt files)
hooks/                # PreToolUse / PostToolUse hook scripts
runs/                 # Runtime artifacts — one directory per workflow run
```

### Slash Commands

The plugin exposes these commands, which form a phased workflow:

| Command | Purpose | Prerequisite |
|---|---|---|
| `/ship <task>` | **Main entry point.** Clarifies intent, auto-classifies complexity (MICRO/STANDARD/FULL), drives the entire lifecycle to DONE, captures lessons. | None |
| `/triage <task>` | Scope capture, track selection (MICRO/STANDARD/FULL), acceptance criteria. | None |
| `/plan <run_id>` | Architecture, interfaces, error taxonomy, implementation checklist. | `/triage` |
| `/execute <run_id>` | Implementation with vertical-slice-first strategy. | `/plan` |
| `/verify <run_id>` | Evidence pack, Done Gate evaluation, DONE/NOT_DONE verdict. | `/execute` |
| `/scope-change <run_id> <desc>` | Documents mid-work scope drift. Blocks until recorded. | Active run |
| `/review` | Weekly housekeeping: lessons aggregation, self-improvement, failure patterns, security exceptions, cost audit. | Completed runs |

### Hooks System

Every tool invocation is intercepted by hooks defined in `hooks/hooks.json`:

- **`pre-tool-guard.sh`** (PreToolUse) — Blocks secrets in tool input, enforces a command denylist (rm -rf /, curl|bash, sudo, chmod 777, .env writes), blocks reads of sensitive files (.pem, .key, id_rsa). Logs all allowed/blocked actions.
- **`post-tool-audit.sh`** (PostToolUse) — Writes audit trail to `runs/<run_id>/audit/events.jsonl`, redacts secrets from logged output, tracks test-gate status for commits.

### Run Directory Convention

Each workflow run creates `runs/<run_id>/` with this structure:
```
runs/<YYYYMMDD-HHMMSS-three-word-slug>/
  intake/        # scope.md, acceptance_criteria.md, track_decision.json
  plan/          # architecture.md, interfaces.md, error_taxonomy.md, impl_checklist.md
  execute/       # slice.md, progress.md, criteria_status.md
  security/      # threat_model.md, scan_results.json (FULL track only)
  final/         # done_gate.json, verification.md, lint/test logs
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
- **Scope creep:** Must be documented via `/scope-change` before implementation continues.
- **Track escalation:** When ambiguous, always choose the more rigorous track (MICRO→STANDARD, STANDARD→FULL).
- **Vertical slice first:** Implement one end-to-end path before expanding.
- **Audit trail:** Every phase transition writes to `events.jsonl`.
- **Toolchain detection:** Never assume npm/node. Detect from project files (`package.json`, `pyproject.toml`, `Cargo.toml`, `Makefile`, `go.mod`). `CLAUDE.md` overrides auto-detection.
- **Learning loop:** Every run captures lessons to `runs/lessons.jsonl`. Every 5 runs, the workflow analyzes patterns and proposes improvements.
