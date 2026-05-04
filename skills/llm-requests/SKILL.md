---
name: llm-requests
description: >
  LLM integration guideline for Python backend projects.
  Covers how to make requests to LLMs via OpenRouter using the OpenAI SDK,
  structured outputs with Pydantic models, async patterns, model-specific
  provider routing, timeouts, and error handling.
  Use when writing code that calls LLM APIs, parses LLM responses,
  or integrates structured AI outputs into backend services.
argument-hint: "[question or context]"
user-invocable: true
---

# LLM Requests

You are a backend LLM integration assistant. Follow the patterns in `references/llm-patterns.md` strictly when generating or reviewing code that interacts with LLM APIs.

## When to activate

Claude SHOULD invoke this skill automatically when the user:

- Writes code that calls an LLM API (OpenAI, OpenRouter, or compatible)
- Creates Pydantic models for structured LLM output
- Asks how to parse or validate LLM responses
- Integrates LLM calls into a backend service or pipeline
- Asks about model selection, provider routing, or fallback strategies
- Works with async LLM calls, timeouts, or retry logic

## Core rules (always enforce)

1. **Client setup**: always use `AsyncOpenAI` with OpenRouter base URL. API key from environment variable, never hardcoded.
2. **Structured output**: define response schemas as Pydantic models with `Field(description=...)`. Use enums for categorical fields.
3. **Parsing method**: use `client.responses.parse()` with `text_format=` for standard models. Use `client.beta.chat.completions.parse()` with `response_format=` for models that require provider routing (e.g., GPT OSS 120B).
4. **Provider routing**: GPT OSS 120B and similar models MUST use explicit provider ordering via `extra_body={"provider": {"order": [...], "allow_fallbacks": True}}`. See reference for approved providers per model.
5. **Timeouts**: always wrap LLM calls in `asyncio.wait_for()` with an explicit timeout. Default 30-60 seconds depending on task complexity.
6. **Temperature**: use `temperature=0` for deterministic/structured outputs. Only increase for creative generation.
7. **System prompts**: always provide a system prompt that defines the role and expected output structure.
8. **Error handling**: catch `asyncio.TimeoutError`, `openai.APIError`, and Pydantic `ValidationError` separately. Never silently swallow LLM errors.
9. **Secrets**: API keys via `os.getenv()` only. Never in code, configs, or CLI arguments.

## How to apply

When generating LLM integration code:
- Read `references/llm-patterns.md` for the full code examples and model-specific rules.
- Match the exact pattern for the model being used.
- Always include timeout, error handling, and structured output parsing.

When reviewing LLM integration code:
- Check that API keys are not hardcoded.
- Verify correct parsing method is used for the model.
- Flag missing timeouts on LLM calls.
- Flag missing provider routing for models that require it.
- Flag raw string parsing instead of structured Pydantic output.
