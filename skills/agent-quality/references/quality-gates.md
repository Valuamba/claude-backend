# Agent Quality Gates

## LLM Call Checklist

### Critical (must have)

- [ ] API key from `os.getenv()`, never hardcoded
- [ ] `AsyncOpenAI` with correct `base_url`
- [ ] `asyncio.wait_for()` timeout wrapping the LLM call
- [ ] Structured output via Pydantic model (`text_format=` or `response_format=`)
- [ ] Error handling: `asyncio.TimeoutError`, `openai.APIError`, `pydantic.ValidationError`
- [ ] System prompt defines role and expected output structure
- [ ] `temperature=0` for deterministic/structured outputs

### Warning (should have)

- [ ] Retry logic for transient API errors (429, 500, 503)
- [ ] Logging of LLM call duration and model used
- [ ] Token usage tracking or estimation
- [ ] Input validation before sending to LLM (not empty, within token limits)

### Info (nice to have)

- [ ] Cost estimation per call
- [ ] Rate limit awareness
- [ ] Fallback model configuration

---

## Post-Processing Classification

### Mapping-only (do NOT report)

Simple structural transformation that does not change business meaning:

- Field renaming (`first_name` → `firstName`)
- Type conversion (`str` → `date`, `int` → `Decimal`)
- DTO → domain model mapping
- Flattening nested structure
- Extracting a sub-field from a larger response

These are normal and expected. Do not include them in the post-processing report.

### Non-trivial (MUST report)

Post-processing that changes the business content of the LLM output:

| Type | Example | Impact |
|---|---|---|
| **Filtering** | Removing items where `confidence < 0.7` | LLM found 10 items, user sees 6 |
| **Dropping fields** | Removing `reasoning` field before returning to client | Explainability lost |
| **Enrichment** | Adding `customer.revenue` from DB to LLM classification result | Final result has data LLM never saw |
| **Aggregation** | Merging results from multiple LLM calls | Single result hides individual call quality |
| **Thresholds/cutoffs** | Marking `risk: high` only if LLM score > 80 | Business decision applied after LLM |
| **Deduplication** | Removing duplicate entities from LLM extraction | LLM may have had reasons for duplicates |
| **Override** | Forcing `category: spam` if sender is in blocklist | LLM decision completely ignored |
| **Sorting/ranking** | Re-ranking LLM results by a custom score | Order differs from LLM intent |
| **Default injection** | Adding default values for fields LLM left empty | Masks LLM incompleteness |
| **Validation + correction** | Auto-correcting LLM output (e.g., fixing country codes) | Hides LLM mistakes from eval |

Each of these MUST appear in the review report with:

```
Type:        <one of the types above>
Location:    <file>:<line>
Description: <what the code does>
Impact:      <what changes between raw and final output>
Removable:   yes / no / conditional
```

### How to determine "removable"

- **yes**: post-processing is for presentation or UX only, eval should skip it
- **no**: post-processing is business-critical, removing it would produce invalid results
- **conditional**: post-processing is needed in production but should be bypassable for eval

---

## Eval Readiness

### Required pattern

The pipeline should support a clean bypass for eval/debug mode:

```python
async def process(
    input: ProcessInput,
    *,
    skip_postprocessing: bool = False,
) -> ProcessResult | RawLLMOutput:
    raw = await call_llm(input)

    if skip_postprocessing:
        return raw

    return postprocess(raw)
```

### What this enables

- **Eval mode**: compare raw LLM output against ground truth without post-processing noise
- **Debug mode**: inspect what the model actually returned
- **A/B testing**: compare raw vs post-processed results on the same ground truth
- **Regression detection**: determine if a regression is in the LLM or in the post-processing

### Clean boundary rule

There must be a clear point in the code where:

```
everything before = LLM output (raw)
everything after  = post-processing (application logic)
```

If raw LLM output and post-processing are tangled in the same function, this is a **warning** — refactor to separate them.

---

## Pipeline Structure

A well-structured agent pipeline should follow this flow:

```
input
  ↓
input validation (Pydantic, business rules)
  ↓
preprocessing (build prompt, gather context)
  ↓
LLM call (structured output, timeout, error handling)
  ↓
raw LLM output (Pydantic model)         ← eval can capture here
  ↓
post-processing (filtering, enrichment)  ← reported in review
  ↓
final result (domain model)
```

### Common anti-patterns

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| LLM call + post-processing in one function | Can't bypass post-processing for eval | Split into `call_llm()` and `postprocess()` |
| Post-processing inside Pydantic `model_validator` | Silently transforms data on parse | Move to explicit step after parsing |
| Multiple LLM calls with interleaved processing | Hard to eval individual calls | Separate each LLM call into its own step |
| Catching all exceptions with bare `except` | Hides LLM errors | Catch specific: `TimeoutError`, `APIError`, `ValidationError` |
| Logging raw LLM output only at DEBUG level | Raw output lost in production | Always save raw output when eval mode is possible |
| Post-processing that mutates the original object | Raw output destroyed | Work on a copy or return a new object |

---

## Report Template

```
# Agent Quality Review: <module/service name>

## Pipeline
input → ... → LLM call → ... → final result

## LLM Call Quality
✓ AsyncOpenAI with env-based API key
✓ Timeout: 60s via asyncio.wait_for
✓ Structured output: Pydantic model
✗ Missing retry logic for transient errors
...

## Post-Processing Found

### 1. Confidence filtering
- Type: filtering
- Location: services/classifier.py:87
- Description: removes items with confidence < 0.7
- Impact: reduces result set, eval won't see low-confidence items
- Removable: yes (for eval)

### 2. Customer enrichment
- Type: enrichment
- Location: services/classifier.py:102
- Description: adds customer.revenue from DB lookup
- Impact: final result contains data not present in LLM output
- Removable: no (required for business logic downstream)

## Eval Readiness
✗ No skip_postprocessing flag
✗ Raw LLM output not accessible separately
✓ Pydantic model defines clean LLM output boundary

## Issues
- [critical] No timeout on LLM call (services/agent.py:45)
- [warning] Post-processing tangled with LLM call in same function
- [warning] No retry logic for API errors
- [info] Consider adding token usage logging
```
