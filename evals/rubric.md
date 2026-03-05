# Eval Rubric

## Scoring

- **PASS**: Output matches expected behavior exactly
- **PARTIAL**: Output is functionally correct but has minor issues (formatting, ordering)
- **FAIL**: Output is incorrect, missing, or violates constraints

## Thresholds

- **Regression**: 0 FAIL allowed on existing golden cases
- **New cases**: All must PASS before DONE
- **Partial tolerance**: Up to 10% PARTIAL allowed (rounded down)

## Golden Case Format

Each entry in `golden_cases.jsonl`:
```json
{
  "id": "gc-<timestamp>-<seq>",
  "input": "<what was given to the system>",
  "expected_behavior": "<what the correct output/behavior is>",
  "gate": "<which done gate this validates>",
  "source_run": "<run_id that generated this case>",
  "tags": ["<category>"]
}
```

## Baseline Storage

Baselines are stored in `evals/baselines/` as JSON snapshots. Each baseline is named `baseline-<YYYYMMDD>.json` and contains the eval results at that point in time.

A new baseline is created when:
- The eval suite is first established
- Golden cases are added or modified
- A `/review` cycle approves a threshold change
