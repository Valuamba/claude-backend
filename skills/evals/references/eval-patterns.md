# Eval Patterns

## Overview

Evals in this approach are practical regression benchmarks around real business scenarios.

This is not a set of abstract prompt tests. Each eval verifies a concrete production task: document or email classification, structured data extraction, entity matching, duplicate detection, risk assessment, intent classification, order parsing, or any other business pipeline.

The core flow:

```
real input -> real service run -> actual structured output -> normalized comparison -> strict metrics + diagnostic metrics + saved artifacts
```

The main idea: evaluate not the beauty of the LLM response, but the ability of the entire application pipeline to consistently produce the correct business result.

---

## When to Use This Eval Format

This format is especially useful when the system:

- Uses an LLM inside a backend process
- Extracts structured data from emails, documents, PDFs, HTML, or CRM records
- Makes business decisions based on incomplete or noisy data
- Uses matching, deduplication, classification, or risk detection
- Frequently changes prompts, models, parsers, heuristics, or orchestration logic
- Must protect against regressions on real cases

A unit test checks a small function. This eval checks the behavior of an entire workflow.

---

## Minimal Structure

For a new eval, start with this structure:

```
evals/
  customer_classification/
    ground_truth/
      core_regression.json
      edge_cases.json
      model_comparison.json
    models.py
    normalization.py
    mapping.py
    bench.py
    README.md
    .runs/
      2026/
        02/
          17/
            113926_req_openai_gpt-5-mini/
              report.txt
              output.json
              run_info.json
```

Names can be adapted to the domain. The important parts are:

- `ground_truth/` with curated cases
- Models for input, expected, and result
- A runner that calls the real service
- A mapping layer if actual output differs from expected shape
- A normalization layer to remove non-semantic noise
- A comparison layer that counts matches
- A report with strict and field-level metrics
- `.runs/YYYY/MM/DD/<HHMMSS_run_name>/` with run artifacts

---

## Use Real System Identifiers

If the system already has a database or other persistent storage, eval input should be built around real persisted IDs.

Examples of such IDs:

- `workflow_id`
- `email_message_id`
- `document_id`
- `lead_id`
- `order_id`
- `customer_id`
- `request_id`
- `file_id`

The idea is that the eval should not reinvent input data if the needed object already exists in the database. In this case, ground truth stores the object ID, the runner loads it through the real service/repository, then runs the production-like pipeline.

If there is no persisted ID, use a production-like payload directly in `input`: email text, path to a test document, order JSON, or a pre-prepared fixture. But if the object already exists in the database, reference it by ID. This makes the eval closer to real system behavior and better at catching regression issues in data loading, service orchestration, and mapping.

---

## Ground Truth

Ground truth must describe business truth, not the raw model response.

Ground truth is stored in the `ground_truth/` folder. Each JSON file usually contains a batch of eval cases grouped by meaning: core regression, edge cases, model comparison, a specific customer segment, a specific document type, or a particular production issue.

Example `ground_truth/core_regression.json`:

```json
[
  {
    "case_id": "core-001",
    "input": {
      "workflow_id": 10001,
      "email_message_id": "email_eval_001"
    },
    "expected": {
      "customer_id": 501,
      "country_code": "US",
      "domain": "alpha-industrial.test|alpha-industrial-alt.test",
      "company_name": "Alpha Industrial LLC",
      "contact": {
        "full_name": "Jane Doe",
        "email": "jane.doe@alpha-industrial.test",
        "phone_numbers": ["+1 555 0100"]
      }
    },
    "description": "Core regression case: existing customer with known domain and contact"
  },
  {
    "case_id": "core-002",
    "input": {
      "workflow_id": 10002,
      "email_message_id": "email_eval_002"
    },
    "expected": {
      "customer_id": null,
      "country_code": "GB",
      "domain": "beta-supplies.test",
      "company_name": "Beta Supplies Ltd",
      "contact": {
        "full_name": "John Smith",
        "email": "orders@beta-supplies.test",
        "phone_numbers": []
      }
    },
    "description": "Core regression case: new customer with no existing customer_id"
  }
]
```

Rules:

- `case_id` is needed for diagnostics and targeted runs.
- `input` must be what the system actually operates on: email ID, workflow ID, document ID, order ID, lead ID, or a production-like payload.
- `expected` must be the structured business result.
- `description` explains why the case exists and what it protects.
- One file in `ground_truth/` can contain dozens of cases.
- If there are multiple correct variants, encode them explicitly, e.g., `alpha-industrial.test|alpha-industrial-alt.test`.

---

## Models

Minimal model example:

```python
from pydantic import BaseModel, Field


class EvalInput(BaseModel):
    email_message_id: str


class ContactExpected(BaseModel):
    full_name: str | None = None
    email: str | None = None
    phone_numbers: list[str] = Field(default_factory=list)


class EvalExpected(BaseModel):
    customer_id: int | None = None
    country_code: str | None = None
    domain: str | None = None
    company_name: str | None = None
    contact: ContactExpected | None = None


class GroundTruthCase(BaseModel):
    case_id: str
    input: EvalInput
    expected: EvalExpected
    description: str | None = None


class EvalResult(BaseModel):
    case_id: str
    expected: EvalExpected
    actual: dict | None = None
    match_correct: bool
    field_matches: dict[str, bool] = Field(default_factory=dict)
    error: str | None = None
    processing_time_sec: float = 0.0
```

---

## Runner

The runner must call the real application service. If the eval checks email classification, it must run the same service used in production. If the eval checks document analysis, it must run the real document pipeline.

Bad:

- Testing only the prompt directly
- Manually feeding text into the LLM and comparing the raw answer
- Mocking important orchestration logic

Good:

- Load the real object by ID
- Call the production-like service
- Get structured output
- Map actual output to the expected shape
- Compare fields
- Save the result

---

## Mapping Layer

Actual output is often richer or more complex than expected output.

For example, the production service may return a large DTO:

```json
{
  "message": {},
  "metadata": {},
  "final_result": {
    "customer": {
      "id": 34240,
      "domain": "example.de",
      "contact": {
        "email": "max@example.de"
      }
    }
  }
}
```

But the eval only checks stable business fields:

```json
{
  "customer_id": 34240,
  "domain": "example.de",
  "contact": {
    "email": "max@example.de"
  }
}
```

For this, a mapping layer is needed:

```python
def map_actual_to_eval_shape(actual: dict) -> dict:
    customer = actual.get("final_result", {}).get("customer", {})
    contact = customer.get("contact") or {}

    return {
        "customer_id": customer.get("id"),
        "country_code": customer.get("country_code"),
        "domain": customer.get("domain"),
        "company_name": customer.get("company_name"),
        "contact": {
            "full_name": contact.get("full_name"),
            "email": contact.get("email"),
            "phone_numbers": contact.get("phone_numbers") or [],
        },
    }
```

The mapping layer allows the production DTO to change without breaking the eval's meaning.

---

## Normalization Layer

The eval should not fail due to insignificant formatting differences.

Examples of noise:

- `Example GmbH` vs `example gmbh`
- `https://www.example.de/` vs `example.de`
- `+49 911 123-456` vs `+49911123456`
- Empty string vs `null`
- Different order of phone numbers in a list

Example normalization:

```python
import re
from typing import Any


def normalize_text(value: str | None) -> str | None:
    if value is None or value == "":
        return None
    value = " ".join(value.strip().split()).lower()
    return value or None


def normalize_domain(value: str | None) -> str | None:
    value = normalize_text(value)
    if value is None:
        return None
    value = re.sub(r"^https?://", "", value)
    value = re.sub(r"^www\.", "", value)
    return value.rstrip("/") or None


def normalize_phone(value: str | None) -> str | None:
    if value is None or value == "":
        return None
    value = re.sub(r"[^\d+]", "", value)
    if "+" in value[1:]:
        value = "+" + value.replace("+", "")
    return value or None


def normalize_field(value: Any, field_name: str) -> Any:
    if value is None:
        return None

    name = field_name.lower()

    if isinstance(value, list):
        if "phone" in name:
            return sorted(
                v for v in (normalize_phone(str(item)) for item in value) if v
            )
        return sorted(
            str(item).strip().lower() for item in value if item is not None
        )

    if not isinstance(value, str):
        return value

    if "domain" in name:
        return normalize_domain(value)
    if "phone" in name:
        return normalize_phone(value)
    if "email" in name:
        return value.strip().lower() or None
    if "country_code" in name:
        return value.strip().upper() or None
    if "company" in name:
        value = normalize_text(value)
        if value:
            value = re.sub(r"\s+(gmbh|llc|ltd|inc|corp|company)$", "", value)
        return value

    return normalize_text(value)
```

Normalization must be honest. It removes formatting noise but must not hide real business errors.

---

## Comparison Layer

The comparison layer is responsible for pass/fail on each field.

```python
from typing import Any


def compare_value(expected: Any, actual: Any, field_name: str) -> bool:
    expected = normalize_field(expected, field_name)
    actual = normalize_field(actual, field_name)

    if expected is None and actual is None:
        return True
    if expected is None or actual is None:
        return False

    if isinstance(expected, str) and "|" in expected and isinstance(actual, str):
        return any(
            part.strip() == actual
            for part in expected.split("|")
            if part.strip()
        )

    return expected == actual


def compare_case(
    expected: EvalExpected, actual: dict
) -> tuple[bool, dict[str, bool]]:
    field_matches = {}

    for field in ["customer_id", "country_code", "domain", "company_name"]:
        field_matches[field] = compare_value(
            getattr(expected, field),
            actual.get(field),
            field,
        )

    if expected.contact:
        actual_contact = actual.get("contact") or {}
        field_matches["contact.full_name"] = compare_value(
            expected.contact.full_name,
            actual_contact.get("full_name"),
            "contact.full_name",
        )
        field_matches["contact.email"] = compare_value(
            expected.contact.email,
            actual_contact.get("email"),
            "contact.email",
        )
        field_matches["contact.phone_numbers"] = compare_value(
            expected.contact.phone_numbers,
            actual_contact.get("phone_numbers"),
            "contact.phone_numbers",
        )

    return all(field_matches.values()), field_matches
```

---

## Metrics

Two levels of metrics are required at minimum.

### Strict Case Accuracy

`match_correct = true` only if all checked fields match.

This metric answers:

> Can we accept the entire business result without manual correction?

It is intentionally strict.

### Field-Level Accuracy

`field_matches` shows which fields matched and which did not.

This metric answers:

> What exactly broke?

Example report:

```text
Total: 58
Strict correct: 41/58 (70.7%)
Errors: 1

Field-level accuracy:
- customer_id: 58/58 (100.0%)
- country_code: 56/58 (96.6%)
- domain: 55/58 (94.8%)
- company_name: 53/58 (91.4%)
- contact.email: 52/58 (89.7%)
- contact.phone_numbers: 40/58 (69.0%)
```

Strict accuracy shows end-to-end readiness. Field-level accuracy shows the direction for improvement.

---

## Run Artifacts

Each eval run must create a separate folder in `.runs/`:

```
.runs/
  2026/
    02/
      17/
        113926_req_openai_gpt-5-mini/
          report.txt
          output.json
          run_info.json
```

Where:

- `output.json` — raw model or service response before benchmark post-processing. If the service returns a large DTO, save the raw payload.
- `report.txt` — human-readable report: summary, strict accuracy, field-level accuracy, failed cases, errors.
- `run_info.json` — run parameters: timestamp, model, run name, selected case IDs, ground truth files, artifact paths, counts.

This layout is convenient for comparing runs by date and does not mix results from different models or runs.

---

## Full Minimal bench.py

This file can be copied into a new project. Only `run_real_service()` needs to be replaced with the actual application service call.

```python
import argparse
import asyncio
import json
import re
import time
from datetime import datetime
from pathlib import Path
from typing import Any

from pydantic import BaseModel, Field


class EvalInput(BaseModel):
    email_message_id: str


class ContactExpected(BaseModel):
    full_name: str | None = None
    email: str | None = None
    phone_numbers: list[str] = Field(default_factory=list)


class EvalExpected(BaseModel):
    customer_id: int | None = None
    country_code: str | None = None
    domain: str | None = None
    company_name: str | None = None
    contact: ContactExpected | None = None


class GroundTruthCase(BaseModel):
    case_id: str
    input: EvalInput
    expected: EvalExpected
    description: str | None = None


class EvalResult(BaseModel):
    case_id: str
    expected: EvalExpected
    actual: dict[str, Any] | None = None
    match_correct: bool
    field_matches: dict[str, bool] = Field(default_factory=dict)
    error: str | None = None
    processing_time_sec: float = 0.0


def normalize_text(value: str | None) -> str | None:
    if value is None or value == "":
        return None
    value = " ".join(value.strip().split()).lower()
    return value or None


def normalize_domain(value: str | None) -> str | None:
    value = normalize_text(value)
    if value is None:
        return None
    value = re.sub(r"^https?://", "", value)
    value = re.sub(r"^www\.", "", value)
    return value.rstrip("/") or None


def normalize_phone(value: str | None) -> str | None:
    if value is None or value == "":
        return None
    value = re.sub(r"[^\d+]", "", value)
    if "+" in value[1:]:
        value = "+" + value.replace("+", "")
    return value or None


def normalize_field(value: Any, field_name: str) -> Any:
    if value is None:
        return None

    name = field_name.lower()

    if isinstance(value, list):
        if "phone" in name:
            return sorted(
                v
                for v in (normalize_phone(str(item)) for item in value)
                if v
            )
        return sorted(
            str(item).strip().lower() for item in value if item is not None
        )

    if not isinstance(value, str):
        return value

    if "domain" in name:
        return normalize_domain(value)
    if "phone" in name:
        return normalize_phone(value)
    if "email" in name:
        return value.strip().lower() or None
    if "country_code" in name:
        return value.strip().upper() or None
    if "company" in name:
        value = normalize_text(value)
        if value:
            value = re.sub(r"\s+(gmbh|llc|ltd|inc|corp|company)$", "", value)
        return value

    return normalize_text(value)


def compare_value(expected: Any, actual: Any, field_name: str) -> bool:
    expected = normalize_field(expected, field_name)
    actual = normalize_field(actual, field_name)

    if expected is None and actual is None:
        return True
    if expected is None or actual is None:
        return False

    if (
        isinstance(expected, str)
        and "|" in expected
        and isinstance(actual, str)
    ):
        return any(
            part.strip() == actual
            for part in expected.split("|")
            if part.strip()
        )

    return expected == actual


def compare_case(
    expected: EvalExpected, actual: dict[str, Any]
) -> tuple[bool, dict[str, bool]]:
    field_matches: dict[str, bool] = {}

    for field in ["customer_id", "country_code", "domain", "company_name"]:
        field_matches[field] = compare_value(
            getattr(expected, field),
            actual.get(field),
            field,
        )

    if expected.contact:
        actual_contact = actual.get("contact") or {}
        field_matches["contact.full_name"] = compare_value(
            expected.contact.full_name,
            actual_contact.get("full_name"),
            "contact.full_name",
        )
        field_matches["contact.email"] = compare_value(
            expected.contact.email,
            actual_contact.get("email"),
            "contact.email",
        )
        field_matches["contact.phone_numbers"] = compare_value(
            expected.contact.phone_numbers,
            actual_contact.get("phone_numbers"),
            "contact.phone_numbers",
        )

    return all(field_matches.values()), field_matches


async def run_real_service(case_input: EvalInput) -> dict[str, Any]:
    """
    Replace this function with the real project service call.

    Example:
        service = container.get(CustomerClassificationService)
        result = await service.classify(case_input.email_message_id)
        return result.model_dump()
    """
    raise NotImplementedError(
        "Connect this benchmark to the real application service"
    )


def map_actual_to_eval_shape(actual: dict[str, Any]) -> dict[str, Any]:
    customer = actual.get("final_result", {}).get("customer", {})
    contact = customer.get("contact") or {}

    return {
        "customer_id": customer.get("id"),
        "country_code": customer.get("country_code"),
        "domain": customer.get("domain"),
        "company_name": customer.get("company_name"),
        "contact": {
            "full_name": contact.get("full_name"),
            "email": contact.get("email"),
            "phone_numbers": contact.get("phone_numbers") or [],
        },
    }


async def run_case_with_raw_output(
    case: GroundTruthCase,
) -> tuple[EvalResult, dict[str, Any] | None]:
    started = time.time()

    try:
        raw_actual = await run_real_service(case.input)
        actual = (
            map_actual_to_eval_shape(raw_actual)
            if "final_result" in raw_actual
            else raw_actual
        )
        match_correct, field_matches = compare_case(case.expected, actual)

        result = EvalResult(
            case_id=case.case_id,
            expected=case.expected,
            actual=actual,
            match_correct=match_correct,
            field_matches=field_matches,
            processing_time_sec=time.time() - started,
        )
        return result, raw_actual
    except Exception as exc:
        result = EvalResult(
            case_id=case.case_id,
            expected=case.expected,
            match_correct=False,
            error=str(exc),
            processing_time_sec=time.time() - started,
        )
        return result, None


def build_report_text(results: list[EvalResult]) -> str:
    total = len(results)
    correct = sum(1 for r in results if r.match_correct)
    errors = sum(1 for r in results if r.error)

    lines = [
        "# Eval Report",
        "",
        f"Total: {total}",
        (
            f"Strict correct: {correct}/{total} ({correct / total * 100:.1f}%)"
            if total
            else "Strict correct: 0/0"
        ),
        f"Errors: {errors}",
        "",
        "## Field-level accuracy",
        "",
    ]

    field_stats: dict[str, dict[str, int]] = {}
    for result in results:
        for field, matched in result.field_matches.items():
            field_stats.setdefault(field, {"correct": 0, "total": 0})
            field_stats[field]["total"] += 1
            field_stats[field]["correct"] += int(matched)

    for field, stats in sorted(field_stats.items()):
        accuracy = stats["correct"] / stats["total"] * 100
        lines.append(
            f"- {field}: {stats['correct']}/{stats['total']} ({accuracy:.1f}%)"
        )

    failed = [r for r in results if not r.match_correct]
    if failed:
        lines.extend(["", "## Failed cases", ""])
        for r in failed:
            lines.append(
                f"- {r.case_id}: error={r.error!r}, fields={r.field_matches}"
            )

    return "\n".join(lines)


def load_ground_truth_cases(
    ground_truth: Path | None, ground_truth_dir: Path
) -> tuple[list[GroundTruthCase], list[str]]:
    files = (
        [ground_truth]
        if ground_truth
        else sorted(ground_truth_dir.glob("*.json"))
    )
    cases: list[GroundTruthCase] = []
    used_files: list[str] = []

    for file_path in files:
        payload = json.loads(file_path.read_text())
        items = payload if isinstance(payload, list) else [payload]
        cases.extend(GroundTruthCase(**item) for item in items)
        used_files.append(str(file_path))

    return cases, used_files


def build_run_dir(runs_dir: Path, run_name: str) -> Path:
    now = datetime.now()
    run_id = now.strftime("%H%M%S")
    safe_name = (
        run_name.replace("/", "_").replace(":", "_").replace(" ", "_")
    )
    return (
        runs_dir
        / now.strftime("%Y")
        / now.strftime("%m")
        / now.strftime("%d")
        / f"{run_id}_{safe_name}"
    )


async def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--ground-truth",
        type=Path,
        default=None,
        help="Path to one ground truth batch JSON file",
    )
    parser.add_argument(
        "--ground-truth-dir", type=Path, default=Path("ground_truth")
    )
    parser.add_argument("--runs-dir", type=Path, default=Path(".runs"))
    parser.add_argument("--run-name", default="req_openai_gpt-5-mini")
    parser.add_argument("--model", default="openai/gpt-5-mini")
    parser.add_argument(
        "--case-ids", default=None, help="Comma-separated case IDs"
    )
    args = parser.parse_args()

    cases, ground_truth_files = load_ground_truth_cases(
        args.ground_truth, args.ground_truth_dir
    )

    if args.case_ids:
        selected = {item.strip() for item in args.case_ids.split(",")}
        cases = [c for c in cases if c.case_id in selected]

    run_dir = build_run_dir(args.runs_dir, args.run_name)
    run_dir.mkdir(parents=True, exist_ok=True)

    results: list[EvalResult] = []
    raw_outputs: list[dict[str, Any]] = []

    for case in cases:
        result, raw_actual = await run_case_with_raw_output(case)
        results.append(result)
        raw_outputs.append(
            {
                "case_id": case.case_id,
                "input": case.input.model_dump(),
                "raw_actual": raw_actual,
                "error": result.error,
                "processing_time_sec": result.processing_time_sec,
            }
        )

    report_text = build_report_text(results)
    print(report_text)

    output_payload = {
        "run": {
            "run_name": run_dir.name,
            "model": args.model,
            "cases_total": len(cases),
        },
        "outputs": raw_outputs,
    }
    (run_dir / "output.json").write_text(
        json.dumps(output_payload, indent=2, ensure_ascii=False, default=str),
        encoding="utf-8",
    )

    (run_dir / "report.txt").write_text(report_text, encoding="utf-8")

    run_info = {
        "run_name": run_dir.name,
        "timestamp": datetime.now().isoformat(),
        "model": args.model,
        "case_ids": [c.case_id for c in cases],
        "ground_truth_files": ground_truth_files,
        "cases_total": len(cases),
        "cases_passed": sum(1 for r in results if r.match_correct),
        "cases_failed": sum(1 for r in results if not r.match_correct),
        "output_json": str(run_dir / "output.json"),
        "report_txt": str(run_dir / "report.txt"),
        "run_info_json": str(run_dir / "run_info.json"),
    }
    (run_dir / "run_info.json").write_text(
        json.dumps(run_info, indent=2, ensure_ascii=False),
        encoding="utf-8",
    )

    print(f"\nRun directory: {run_dir}")
    print(f"Raw model/service output: {run_dir / 'output.json'}")
    print(f"Report: {run_dir / 'report.txt'}")
    print(f"Run info: {run_dir / 'run_info.json'}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Domain Metric Variants

Different tasks need different metrics:

| Domain | Key Metrics |
|---|---|
| Classification | expected label, confidence, explanation fields |
| Extraction | field-level accuracy, missing fields, extra fields |
| Matching | selected ID accuracy, candidate set accuracy |
| Deduplication | true positives, false positives, false negatives |
| Document analysis | clause recall, requirement coverage, risk detection accuracy |
| Order parsing | client match, positions match, quantity match, order metadata match |
| Security scanning | detection rate, false positive rate, false negative rate |

Do not try to force all evals into one abstract metric. A good eval measures the quality of a specific business decision.

---

## How to Read Failures

If a case failed, first identify the problem type:

- **wrong expected**: ground truth is outdated or was incorrectly labeled
- **wrong actual**: the pipeline genuinely made an error
- **mapping bug**: actual is correct but was incorrectly mapped to the eval shape
- **normalization bug**: comparison is too strict or too lenient
- **service error**: the pipeline could not process the input
- **data drift**: the source data has changed
- **model drift**: the model started responding differently

Do not look only at the overall score. Strict accuracy shows the symptom; field-level metrics show the cause.

---

## Practical Work Cycle

1. Find a real production or production-like case.
2. Determine what the correct business result should be.
3. Add the case to `ground_truth/`.
4. Run the eval.
5. Check strict accuracy.
6. Check field-level accuracy.
7. Fix the prompt, parser, mapping, service logic, or normalization.
8. Run the eval again.
9. Compare artifacts in `.runs/YYYY/MM/DD/<HHMMSS_run_name>/`.

This turns production failures into long-term regression protection.

---

## Limitations

### Strict accuracy can be too harsh

If one secondary field does not match, the entire case becomes failed. That is why strict accuracy must be read together with field-level accuracy.

### Ground truth requires maintenance

If data changed or the correct expected result was refined, ground truth must be updated deliberately.

### Real-service evals are heavier than unit tests

They are slower and more dependent on the environment because they check the real workflow. This is a deliberate tradeoff: this eval is needed for confidence in behavior, not for millisecond verification of a small function.

### Normalization must not hide errors

Normalization should remove only formatting noise. It must not transform different business values into identical ones.

---

## Checklist for a New Eval

1. The eval has its own directory.
2. There is a `ground_truth/` folder.
3. Each case has `case_id`, `input`, `expected`, `description`.
4. `expected` describes a structured business result.
5. The runner calls the real application service.
6. Actual output is mapped to a stable comparison shape.
7. There is a normalization layer.
8. There is field-by-field comparison.
9. There is strict case accuracy.
10. There is diagnostic field-level accuracy.
11. Targeted runs by case IDs are supported.
12. Each run creates `.runs/YYYY/MM/DD/<HHMMSS_run_name>/`.
13. The run folder contains `output.json`, `report.txt`, `run_info.json`.
14. There is a README with run instructions and failure interpretation.
