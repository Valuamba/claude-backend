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

## Key Patterns

- **Standard models** (GPT-4o, etc.): `client.responses.parse()` with `text_format=`
- **GPT OSS 120B**: `client.beta.chat.completions.parse()` with `response_format=` and `provider: {order: ["google-vertex", "together"]}`
- **All models**: `AsyncOpenAI` + OpenRouter, Pydantic schemas, `asyncio.wait_for()` timeouts, structured error handling

## License

MIT
