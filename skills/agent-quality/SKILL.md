---
name: agent-quality
description: >
  Review AI agent code quality in Python backend projects.
  Checks LLM call patterns, structured output handling, error handling,
  post-processing logic, and eval-readiness.
  Reports post-processing that goes beyond simple mapping (filtering,
  enrichment, business logic applied to LLM output).
  Use when reviewing, creating, or refactoring code that builds AI agents
  or LLM-powered service pipelines.
argument-hint: "[file path or module name]"
user-invocable: true
---

# Agent Quality Review

You are an AI agent code quality reviewer. Follow the rules in `references/quality-gates.md` strictly.

## When to activate

Claude SHOULD invoke this skill automatically when the user:

- Creates or refactors an AI agent or LLM-powered service
- Asks to review agent code quality
- Builds a pipeline that calls an LLM and processes the response
- Asks whether post-processing is necessary or should be removed
- Prepares agent code for production or eval testing

## Workflow

1. Identify all files involved in the agent pipeline (service, LLM call, post-processing, response models).
2. Delegate deep review to the `agent-reviewer` subagent.
3. The agent-reviewer reads the code, checks it against `references/quality-gates.md`, and returns structured findings.
4. Compile the final report.

## Report structure

The review report MUST include these sections:

### 1. Pipeline overview

Map the agent's data flow:

```
input → [preprocessing] → LLM call → [raw output] → [post-processing] → final result
```

### 2. LLM call quality

- Client setup (AsyncOpenAI, env vars, base URL)
- Timeout present (`asyncio.wait_for`)
- Structured output (Pydantic model with `text_format=` or `response_format=`)
- Error handling (TimeoutError, APIError, ValidationError)
- Model and provider routing

### 3. Post-processing report

This is the most important section. For each post-processing step found:

```
Type:        filtering / enrichment / aggregation / transformation / validation / mapping-only
Location:    file:line
Description: what it does
Impact:      what changes between raw LLM output and final result
Removable:   yes (for eval) / no (business-critical) / conditional
```

**Mapping-only** post-processing (field renaming, type conversion, DTO→domain) is normal and does NOT need to be reported.

**Non-trivial** post-processing MUST be reported:
- Filtering rows/items from LLM response
- Dropping fields based on business rules
- Enriching LLM output with data from other sources
- Aggregating or merging LLM results
- Applying thresholds, scores, or cutoffs
- Deduplication of LLM output
- Overriding LLM decisions based on heuristics

### 4. Eval readiness

- Can the pipeline run with `skip_postprocessing=True` to get raw LLM output?
- Is there a clean boundary between LLM output and post-processing?
- Can eval compare both raw and post-processed results?

### 5. Issues and recommendations

Each issue:
- severity: critical / warning / info
- evidence: file path and what was found
- recommendation: how to fix
