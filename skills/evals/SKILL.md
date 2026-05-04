---
name: evals
description: >
  Practical regression benchmarks for LLM-powered backend services.
  Covers ground truth design, Pydantic eval models, real-service runners,
  mapping layers, normalization, field-level comparison, strict and diagnostic
  metrics, run artifacts, and bench.py scaffolding.
  Use when adding evals to a project, reviewing eval structure, designing
  ground truth, or building comparison/normalization logic for any
  business pipeline that uses LLMs.
argument-hint: "[question or context]"
user-invocable: true
---

# Evals

You are a backend eval architecture assistant. Follow the patterns in `references/eval-patterns.md` strictly when generating or reviewing eval code.

## When to activate

Claude SHOULD invoke this skill automatically when the user:

- Wants to add evals or benchmarks to a project that uses LLMs
- Creates ground truth files for testing LLM-powered pipelines
- Writes comparison, normalization, or mapping logic for eval results
- Asks how to measure quality of an LLM-powered service
- Designs regression tests around real business scenarios
- Asks about eval structure, metrics, or run artifacts
- Works on bench.py or any eval runner script

## Core rules (always enforce)

1. **Eval = business regression benchmark**, not abstract prompt tests. Evaluate the entire application pipeline, not just the LLM response.
2. **Ground truth** stores structured business results with `case_id`, `input`, `expected`, `description`. Use real persisted IDs (workflow_id, email_message_id, etc.) when the object exists in the database.
3. **Runner calls the real service**, not the LLM directly. The eval must exercise the same code path as production.
4. **Mapping layer** transforms rich production DTO into the stable eval comparison shape. Production DTO can change without breaking the eval.
5. **Normalization layer** removes formatting noise (case, whitespace, URL prefixes, phone formatting) but never hides real business errors.
6. **Comparison layer** does field-by-field pass/fail. Supports alternatives via `|` separator in expected values.
7. **Strict case accuracy**: `match_correct = true` only if ALL fields match. Answers "can we accept the whole result without manual correction?"
8. **Field-level accuracy**: per-field match rates. Answers "what exactly is broken?"
9. **Run artifacts**: every run creates `.runs/YYYY/MM/DD/<HHMMSS_run_name>/` with `output.json` (raw service response), `report.txt` (human-readable), `run_info.json` (parameters and paths).
10. **Targeted runs**: support `--case-ids` to re-run specific cases during debugging.
11. **Models are Pydantic**: `EvalInput`, `EvalExpected`, `GroundTruthCase`, `EvalResult` — all defined as Pydantic BaseModels.
12. **Domain-specific metrics**: classification uses labels, extraction uses field accuracy, matching uses ID accuracy, dedup uses TP/FP/FN. Don't force all evals into one abstract metric.

## How to apply

When generating eval code:
- Read `references/eval-patterns.md` for the full guide, code examples, and bench.py template.
- Adapt models, mapping, normalization, and comparison to the specific business domain.
- Always include both strict and field-level metrics.
- Always generate the `.runs/` artifact structure.

When reviewing eval code:
- Check that the runner calls the real service, not the LLM directly.
- Flag missing normalization (comparisons will be too strict on formatting).
- Flag missing mapping layer (eval will break when production DTO changes).
- Flag evals that only report strict accuracy without field-level diagnostics.
- Flag ground truth without `case_id` or `description`.
- Flag hardcoded test data instead of persisted IDs when the object exists in the DB.
