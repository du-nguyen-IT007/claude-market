---
name: adk-callback-guardrails
description: >
  Implement callbacks, guardrails, and lifecycle hooks for Google ADK agents.
  Use when the user asks to "add guardrails", "implement callbacks",
  "add safety checks", "before_model_callback", "after_tool_callback",
  "block certain inputs", "validate tool arguments", "add logging hooks",
  "implement input/output filtering", "add caching to agent", or needs
  to intercept and control agent execution flow in ADK.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Callbacks & Guardrails

Implement lifecycle hooks to observe, customize, and control ADK agent behavior at every execution point — from input validation to output filtering.

## Callback Types Overview

| Callback | When | Purpose | Return to Skip |
|----------|------|---------|---------------|
| `before_agent_callback` | Before agent runs | Validate state, skip agent | `types.Content` |
| `after_agent_callback` | After agent finishes | Modify/replace output | `types.Content` |
| `before_model_callback` | Before LLM call | Input guardrails, caching | `LlmResponse` |
| `after_model_callback` | After LLM response | Output filtering, sanitization | `LlmResponse` |
| `before_tool_callback` | Before tool execution | Validate args, policy check | `dict` |
| `after_tool_callback` | After tool returns | Modify/log results | `dict` |

## Core Mechanism: Return None vs Return Object

- **`return None`** → proceed with normal execution
- **`return <specific_object>`** → skip/replace the step

## 1. Input Guardrail (before_model_callback)

Block or modify requests before they reach the LLM:

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmResponse, LlmRequest
from google.genai import types
from typing import Optional

BLOCKED_KEYWORDS = ["hack", "exploit", "bypass"]

def input_guardrail(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """Block requests containing forbidden keywords."""
    last_message = ""
    if llm_request.contents and llm_request.contents[-1].role == 'user':
        if llm_request.contents[-1].parts:
            last_message = llm_request.contents[-1].parts[0].text or ""

    for keyword in BLOCKED_KEYWORDS:
        if keyword.lower() in last_message.lower():
            return LlmResponse(
                content=types.Content(
                    role="model",
                    parts=[types.Part(text="I cannot process this request.")]
                )
            )
    return None  # Allow normal LLM call

agent = LlmAgent(
    name="safe_agent",
    model="gemini-2.5-flash",
    instruction="Be helpful.",
    before_model_callback=input_guardrail
)
```

## 2. Tool Argument Guardrail (before_tool_callback)

Validate or block tool calls based on arguments:

```python
from google.adk.tools import ToolContext
from typing import Optional

def validate_tool_args(
    callback_context: CallbackContext,
    tool_name: str,
    args: dict,
    tool_context: ToolContext
) -> Optional[dict]:
    """Block certain tool arguments."""
    if tool_name == "get_weather" and args.get("city", "").lower() == "restricted_city":
        return {"status": "error", "message": "Access to this city is restricted."}

    # Log all tool calls
    print(f"[Tool Call] {tool_name} with args: {args}")
    return None  # Allow tool execution

agent = LlmAgent(
    name="guarded_agent",
    model="gemini-2.5-flash",
    before_tool_callback=validate_tool_args,
    tools=[get_weather]
)
```

## 3. Output Sanitization (after_model_callback)

Filter or modify LLM responses:

```python
def sanitize_output(
    callback_context: CallbackContext, llm_response: LlmResponse
) -> Optional[LlmResponse]:
    """Remove sensitive information from responses."""
    if llm_response.content and llm_response.content.parts:
        for part in llm_response.content.parts:
            if part.text and "CONFIDENTIAL" in part.text:
                part.text = part.text.replace("CONFIDENTIAL", "[REDACTED]")
    return None  # Return None to use (modified) response as-is

agent = LlmAgent(
    name="sanitized_agent",
    model="gemini-2.5-flash",
    after_model_callback=sanitize_output
)
```

## 4. Conditional Agent Skip (before_agent_callback)

Skip an agent's execution based on state:

```python
def check_before_agent(callback_context: CallbackContext) -> Optional[types.Content]:
    """Skip agent if a flag is set in state."""
    if callback_context.state.get("skip_processing", False):
        return types.Content(
            role="model",
            parts=[types.Part(text="Processing skipped per configuration.")]
        )
    return None  # Run agent normally

agent = LlmAgent(
    name="conditional_agent",
    model="gemini-2.5-flash",
    before_agent_callback=check_before_agent
)
```

## 5. Dynamic State Management in Callbacks

Read/write session state to make agents context-aware:

```python
def track_model_calls(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """Count and limit model calls per session."""
    call_count = callback_context.state.get("model_call_count", 0)
    MAX_CALLS = 10

    if call_count >= MAX_CALLS:
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="Rate limit reached. Please try again later.")]
            )
        )

    callback_context.state["model_call_count"] = call_count + 1
    return None
```

## 6. Caching Pattern

Cache LLM responses to avoid redundant calls:

```python
import hashlib
import json

def cache_before_model(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """Check cache before calling LLM."""
    cache_key = hashlib.md5(
        json.dumps(str(llm_request.contents[-1])).encode()
    ).hexdigest()

    cached = callback_context.state.get(f"cache:{cache_key}")
    if cached:
        return LlmResponse(
            content=types.Content(role="model", parts=[types.Part(text=cached)])
        )
    return None

def cache_after_model(
    callback_context: CallbackContext, llm_response: LlmResponse
) -> Optional[LlmResponse]:
    """Store response in cache."""
    if llm_response.content and llm_response.content.parts:
        text = llm_response.content.parts[0].text
        if text:
            cache_key = hashlib.md5(text[:50].encode()).hexdigest()
            callback_context.state[f"cache:{cache_key}"] = text
    return None
```

## 7. Request Modification

Inject system context dynamically:

```python
def inject_context(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """Add user-specific context to system instruction."""
    user_lang = callback_context.state.get("user:language", "English")
    original = llm_request.config.system_instruction or types.Content(role="system", parts=[])

    if not isinstance(original, types.Content):
        original = types.Content(role="system", parts=[types.Part(text=str(original))])
    if not original.parts:
        original.parts.append(types.Part(text=""))

    original.parts[0].text = f"[User language: {user_lang}] " + (original.parts[0].text or "")
    llm_request.config.system_instruction = original
    return None
```

## 8. Tool Result Post-Processing

Standardize or enrich tool outputs:

```python
def enrich_tool_result(
    callback_context: CallbackContext,
    tool_name: str,
    args: dict,
    tool_context: ToolContext,
    tool_response: dict
) -> Optional[dict]:
    """Add metadata to all tool results."""
    tool_response["_metadata"] = {
        "tool": tool_name,
        "timestamp": "2024-01-01T00:00:00Z",
        "invocation_id": callback_context.invocation_id
    }
    return None  # Use modified response
```

## 9. Artifact Handling in Callbacks

Save/load files during agent lifecycle:

```python
def save_report_artifact(
    callback_context: CallbackContext,
    tool_name: str,
    args: dict,
    tool_context: ToolContext,
    tool_response: dict
) -> Optional[dict]:
    """Save generated reports as artifacts."""
    if tool_name == "generate_report" and tool_response.get("status") == "success":
        report_text = tool_response.get("report", "")
        part = types.Part(text=report_text)
        version = tool_context.save_artifact("latest_report.txt", part)
        tool_response["artifact_version"] = version
    return None
```

## Combining Multiple Callbacks

```python
agent = LlmAgent(
    name="full_guarded_agent",
    model="gemini-2.5-flash",
    instruction="Be helpful and safe.",
    before_agent_callback=check_before_agent,
    before_model_callback=input_guardrail,
    after_model_callback=sanitize_output,
    before_tool_callback=validate_tool_args,
    after_tool_callback=enrich_tool_result,
    tools=[get_weather, search_docs]
)
```

## Best Practices

1. **Single responsibility** — one callback, one purpose
2. **Return None by default** — only return objects to override behavior
3. **Handle errors with try/except** — don't crash the agent
4. **Use state prefixes** — `app:`, `user:`, `temp:` for clarity
5. **Keep callbacks fast** — avoid blocking operations
6. **Test callbacks independently** — mock context objects for unit tests
7. **Log descriptively** — include agent name, invocation ID, tool name

## Additional Resources

- **`references/adk-callbacks-full.md`** — Complete ADK callbacks documentation with all code examples
