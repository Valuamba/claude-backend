---
name: prompt-audit
description: >
  Audit AI agent prompts for quality. Identifies reactive patches
  (rules added after bad outputs) vs original design rules in system prompts,
  user prompt templates, and output schemas. Surfaces patch clusters,
  editing rounds, and prompt debt. Use when reviewing, cleaning up,
  or refactoring prompts in LLM-powered services.
argument-hint: "<file path or paste prompt text>"
user-invocable: true
---

# Prompt Audit

Review AI agent prompts for quality and identify accumulated prompt debt.

## When to activate

Claude SHOULD invoke this skill automatically when the user:

- Asks to review or audit a prompt
- Wants to clean up or simplify a system prompt
- Asks why a prompt is so long or complex
- Refactors an AI agent and wants to understand prompt structure
- Asks about prompt quality, prompt debt, or prompt hygiene
- Mentions "patches" or "reactive rules" in prompts

## Workflow

1. **Locate the prompt.** If the user provides a file path, read it. If the user provides a module or service name, search for string constants named `SYSTEM_PROMPT`, `USER_PROMPT`, `PROMPT_TEMPLATE`, or similar. Also check for prompt-building functions.
2. **Identify all prompt components:**
   - System prompt (role, instructions, rules)
   - User prompt template (input formatting, context injection)
   - Output schema (Pydantic model with Field descriptions — this is design, not a patch)
3. **Delegate to the `prompt-auditor` agent.** Pass all prompt components found.
4. **Compile the final report** from the agent's findings.

## How to find prompts in code

Search for these patterns:

```
SYSTEM_PROMPT = "..."
SYSTEM_PROMPT = """..."""
system_prompt = "..."
messages=[{"role": "system", "content": ...}]
{"role": "system", "content": ...}
prompt = f"..."
prompt_template = "..."
```

Also check:
- Constants at module top level
- Class attributes in service/agent classes
- Separate `.txt` or `.md` files loaded at runtime
- Jinja2 or string templates

## What to include in the report

Read `references/prompt-audit-guide.md` for the full classification framework.

The report must contain:

1. **Summary** — patch clusters found, estimated patch share, editing rounds, most patched topics, overall impression
2. **Patch clusters** — grouped findings with signals, quotes, and likely bugs
3. **Recommendations** — which patches to keep, which to merge into design, which to remove

## Important distinctions

- **Output schema** (Pydantic models, Field descriptions, type annotations) is DESIGN, not a patch. Never flag it.
- **One signal is weak.** Only flag when two or more patch signals converge on the same rule.
- **When in doubt, classify as design.** A false patch is worse than a missed one.
