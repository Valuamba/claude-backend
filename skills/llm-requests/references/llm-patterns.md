# LLM Integration Patterns

## Client Setup

Always use `AsyncOpenAI` with OpenRouter. API key must come from environment.

```python
import os
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url="https://openrouter.ai/api/v1",
)
```

Never hardcode API keys. Never pass them via CLI arguments.

---

## Pattern 1: Standard Structured Output

Use `client.responses.parse()` with `text_format=` for standard models (GPT-4o, Claude, etc.).

```python
import os
from enum import Enum
from pydantic import BaseModel, Field
from openai import AsyncOpenAI

# 1. Define response schema with Pydantic
class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

class Analysis(BaseModel):
    sentiment: Sentiment
    summary: str = Field(description="Brief summary of the key points")

# 2. Create client
client = AsyncOpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url="https://openrouter.ai/api/v1",
)

# 3. Make request with structured output
response = await client.responses.parse(
    model="openai/gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "You are a data analyst. Structure this review."},
        {"role": "user", "content": "The app is fast but the interface is confusing."},
    ],
    text_format=Analysis,
)

# 4. Access parsed result
result = response.output_parsed
print(result.sentiment)  # Sentiment.NEUTRAL
print(result.summary)    # "The app performs well but has UX issues."
```

### When to use

- GPT-4o and other standard OpenAI models
- Any model that supports the `responses.parse()` API
- Simple request/response without special provider requirements

---

## Pattern 2: Provider-Routed Structured Output

Use `client.beta.chat.completions.parse()` with `response_format=` and explicit `provider` ordering for models that require specific infrastructure.

**GPT OSS 120B** and similar models MUST use this pattern with approved providers.

```python
import os
import asyncio
from pydantic import BaseModel, Field
from openai import AsyncOpenAI

TIMEOUT_SECONDS = 60
MODEL = "openai/gpt-4o-oss-120b"  # or whatever model ID

SYSTEM_PROMPT = "You are a decision analyst. Analyze the provided data and return structured output."

class AnalyzedDecision(BaseModel):
    recommendation: str = Field(description="Clear recommendation based on the analysis")
    confidence: float = Field(description="Confidence score from 0.0 to 1.0")
    reasoning: str = Field(description="Step-by-step reasoning behind the recommendation")

client = AsyncOpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url="https://openrouter.ai/api/v1",
)

async def analyze(user_prompt: str) -> AnalyzedDecision:
    response = await asyncio.wait_for(
        client.beta.chat.completions.parse(
            model=MODEL,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_prompt},
            ],
            response_format=AnalyzedDecision,
            temperature=0,
            extra_body={
                "provider": {
                    "order": ["google-vertex", "together"],
                    "allow_fallbacks": True,
                }
            },
        ),
        timeout=TIMEOUT_SECONDS,
    )

    return response.choices[0].message.parsed
```

### When to use

- GPT OSS 120B — **only** with providers: `["google-vertex", "together"]`
- Any model that requires specific provider routing
- When you need explicit timeout control

### Approved providers by model

| Model | Providers | Fallbacks |
|---|---|---|
| GPT OSS 120B | `["google-vertex", "together"]` | `allow_fallbacks: True` |

If a model is not in this table, use Pattern 1 (standard) instead.

---

## Pydantic Schema Design

### Use enums for categorical fields

```python
from enum import Enum

class Priority(str, Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
```

### Always add Field descriptions

```python
from pydantic import BaseModel, Field

class Report(BaseModel):
    title: str = Field(description="One-line title summarizing the finding")
    severity: Priority = Field(description="Impact severity level")
    details: str = Field(description="Detailed explanation with evidence")
```

Descriptions help the LLM understand what to put in each field.

### Use nested models for complex output

```python
class Issue(BaseModel):
    title: str = Field(description="Issue title")
    severity: Priority
    evidence: str = Field(description="Concrete evidence from the input")

class AuditResult(BaseModel):
    summary: str = Field(description="Executive summary")
    issues: list[Issue] = Field(description="List of identified issues")
    score: float = Field(description="Overall score from 0.0 to 1.0")
```

---

## Error Handling

Always handle these three error types separately:

```python
import asyncio
from openai import APIError
from pydantic import ValidationError

async def safe_llm_call(prompt: str) -> Result | None:
    try:
        response = await asyncio.wait_for(
            client.responses.parse(
                model="openai/gpt-4o-2024-08-06",
                input=[
                    {"role": "system", "content": SYSTEM_PROMPT},
                    {"role": "user", "content": prompt},
                ],
                text_format=Result,
            ),
            timeout=TIMEOUT_SECONDS,
        )
        return response.output_parsed

    except asyncio.TimeoutError:
        # LLM took too long — retry with shorter prompt or fail gracefully
        logger.error("LLM request timed out after %ds", TIMEOUT_SECONDS)
        return None

    except APIError as exc:
        # OpenRouter/model error — log status and message
        logger.error("LLM API error: %s (status %s)", exc.message, exc.status_code)
        return None

    except ValidationError as exc:
        # LLM returned data that doesn't match the Pydantic schema
        logger.error("LLM output validation failed: %s", exc)
        return None
```

### Rules

- Never silently swallow errors from LLM calls.
- Always log the error type and details.
- Timeouts are mandatory — LLM calls can hang indefinitely without them.
- Return `None` or raise a domain-specific exception — never crash the service.

---

## Checklist

Before shipping LLM integration code:

- [ ] API key from `os.getenv()`, never hardcoded
- [ ] `AsyncOpenAI` with OpenRouter base URL
- [ ] Response schema defined as Pydantic model with Field descriptions
- [ ] Correct parsing method for the model (Pattern 1 vs Pattern 2)
- [ ] Provider routing configured for models that require it
- [ ] `asyncio.wait_for()` timeout wrapping every LLM call
- [ ] `temperature=0` for structured/deterministic outputs
- [ ] System prompt defines role and expected output
- [ ] Error handling for TimeoutError, APIError, ValidationError
- [ ] No raw string parsing — always use structured output
