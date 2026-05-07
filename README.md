# backend-guide

Claude Code plugin for Python backend development. Provides architecture guidance for LLM integration, structured outputs, async patterns, and API design.

## Installation

```bash
# Add marketplace (one-time)
/plugin marketplace add Valuamba/claude-backend

# Install plugin
/plugin install backend-guide@valuamba-claude-backend
```

Or for local development/testing:

```bash
claude --plugin-dir /path/to/claude-backend
```

## Skills

### llm-requests

LLM integration guideline. Covers how to make requests to LLMs via OpenRouter, parse structured outputs with Pydantic, handle model-specific provider routing, timeouts, and errors.

Activates automatically when writing code that calls LLM APIs or when asked about LLM integration patterns.

```
/backend-guide:llm-requests
```

### evals

Practical regression benchmarks for LLM-powered backend services. Covers ground truth design, Pydantic eval models, real-service runners, mapping/normalization layers, field-level comparison, strict and diagnostic metrics, run artifacts, and a full bench.py template.

Activates automatically when adding evals to a project or designing comparison logic for any business pipeline that uses LLMs.

```
/backend-guide:evals
```

### agent-quality

AI agent code quality review. Checks LLM call patterns, post-processing logic, error handling, and eval readiness. Reports non-trivial post-processing (filtering, enrichment, overrides, thresholds) separately from simple mapping. Delegates deep review to the `agent-reviewer` subagent.

```
/backend-guide:agent-quality path/to/service.py
```

## Key Patterns

- **Standard models** (GPT-4o, etc.): `client.responses.parse()` with `text_format=`
- **GPT OSS 120B**: `client.beta.chat.completions.parse()` with `response_format=` and `provider: {order: ["google-vertex", "together"]}`
- **All models**: `AsyncOpenAI` + OpenRouter, Pydantic schemas, `asyncio.wait_for()` timeouts, structured error handling

## License

MIT
