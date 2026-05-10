# Prompt Audit Guide

## Two Layers of a Prompt

Every prompt that has been used in production accumulates two layers:

**Design layer** — rules derived from the task a priori. Universal, structural, written before the prompt meets real data. Examples:

- Role definition ("You are a customer classifier")
- Output format specification
- Field definitions and types
- General policies (how to handle missing data, conflicts)
- Terminology definitions
- Anti-hallucination baselines

**Patch layer** — rules added after the model produced bad output. Narrow, reactive, often restating the design in a stricter form. Written to plug a specific observed failure.

The goal of a prompt audit is to separate the second layer from the first.

---

## Patch Signals Reference

| Signal | Code | Description | Strength |
|---|---|---|---|
| Concrete counter-example | A | Rule contains a specific value, name, field from real data | Strong |
| General → specific narrowing | B | Same topic stated generally first, then specifically later | Strong |
| Idea repetition | C | Same norm in 2-3 different wordings | Medium |
| Modality escalation | D | Soft → hard wording shift (should → must, prefer → never) | Medium |
| Emphasis bursts | E | ALL CAPS, IMPORTANT, CRITICAL, exclamation marks | Medium |
| Synonym enumerations | F | Multiple terms for the same thing in one prohibition | Medium |
| "X is not Y" constructions | G | Direct negation of a wrong interpretation | Strong |
| Workaround prohibitions | H | Ban on a technique that bypasses an earlier rule | Strong |
| Pre-return check blocks | I | "Before returning, verify / re-read / walk through" | Medium |
| Counter-default behavior | J | "Do not shorten / summarize / infer / drop" | Medium |

**Confidence rule:** one signal alone is weak evidence. Two or more signals on the same rule is a confident patch flag.

---

## What Is NOT a Patch

These are design elements regardless of when they were added:

- Task description and role definition
- Input/output specification
- Output schema (Pydantic models, Field descriptions, types)
- Format allowlists and blocklists
- Term and concept definitions
- Universal policies (conflict resolution, missing-data handling)
- Anti-hallucination baselines ("only use information from the provided context")
- General tone and style requirements

---

## Patch Clusters

Related patches should be grouped into clusters. A cluster is one analytical unit:

- Several rules guarding the same field or topic
- Rules that form an iteration sequence (original → strengthening → workaround ban)
- Rules scattered across the prompt but addressing the same failure mode

Inside a cluster, reconstruct the iteration trace when possible:

```
Round 1: "Prefer using the official company name"
Round 2: "Always use the official company name, never abbreviations"
Round 3: "IMPORTANT: The company name must match exactly. Do NOT use
          'Inc', 'LLC', abbreviations, or informal names."
```

This shows three editing rounds around the same bug.

---

## Recommendations Framework

For each patch cluster, recommend one of:

### Keep as-is

The patch addresses a real, recurring failure that cannot be prevented by better design. The rule is narrow but necessary.

### Merge into design

The patch reveals a gap in the original design. Instead of keeping a reactive clamp, rewrite the design rule to be comprehensive enough that the patch becomes redundant.

Example:
- Design: "Extract the company name from the email"
- Patch: "NEVER use the sender's display name as the company name"
- Merged: "Extract the company name from the email body or signature. The sender's display name, email prefix, and domain name are not reliable sources for company name."

### Remove

The patch is obsolete because:
- The model no longer exhibits the failure it guards against
- A better design rule now covers the case
- The patch conflicts with other rules
- The patch is so narrow it only applies to one historical case

### Extract to eval

The patch documents a real failure case that should become a ground truth case in the eval suite instead of a prompt rule. Move the knowledge from the prompt into `ground_truth/edge_cases.json`.

---

## Prompt Debt

Prompt debt is the accumulation of patches that:

- Make the prompt longer without making it clearer
- Duplicate design rules in stricter form
- Contradict each other when they guard overlapping areas
- Create a false sense of coverage (the patch fixes one case but the underlying issue affects many)

Signs of high prompt debt:

- Prompt is 2-3x longer than the task requires
- More than 30% of prompt text is patches
- 3+ editing rounds visible on the same topic
- Multiple "IMPORTANT" / "CRITICAL" / "NOTE" markers
- Rules that start with "Do NOT" outnumber constructive rules

---

## Audit Report Template

```
# Prompt Audit

## Summary

- **Patch clusters found:** N
- **Estimated patch share of prompt text:** roughly X%
- **Editing rounds visible:** N
- **Most patched topics:** topic 1, topic 2, topic 3
- **Overall impression:** stable design / moderate debt / heavy accumulation

## Patch clusters

### P1 — <topic>

**Signals:** A, C, D
**Iteration trace:** Round 1 set a soft preference, round 2 made it mandatory,
round 3 banned a workaround the model found.

**Quotes:**
> exact quote from the prompt

> exact quote from the prompt

**Likely bug:** The model was using X instead of Y, causing Z.

**Recommendation:** merge into design / keep / remove / extract to eval

---

### P2 — <topic>

(same structure)

## Recommendations Summary

| Cluster | Action | Rationale |
|---|---|---|
| P1 | merge into design | general rule can cover this |
| P2 | extract to eval | one-off case, better as ground truth |
| P3 | keep | real recurring failure |
```
