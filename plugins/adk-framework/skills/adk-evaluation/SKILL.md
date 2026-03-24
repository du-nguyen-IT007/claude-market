---
name: adk-evaluation
description: >
  Test and evaluate Google ADK agents using eval sets, metrics, and the
  CLI/UI tools. Use when the user asks to "test agent", "evaluate agent",
  "create eval set", "write agent tests", "adk eval", "benchmark agent",
  "measure agent quality", "create test cases", "run evaluation",
  "check agent accuracy", or needs guidance on testing and evaluating
  ADK agent performance.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Evaluation

Test and evaluate Google ADK agents using eval sets, metrics, and built-in CLI/UI tools.

## Evaluation Approaches

| Approach | Best For | Complexity |
|----------|---------|------------|
| EvalSet files (`.evalset.json`) | Integration tests, multi-turn | Medium |
| `adk eval` CLI | Automated CI/CD testing | Low |
| Dev UI | Interactive testing & capture | Low |
| Custom Python tests | Unit tests, specific checks | Medium |

## 1. EvalSet Format

An evalset file contains multiple evaluation sessions, each with turns that define user queries and expected responses.

### Basic EvalSet Structure
```json
{
  "eval_set_id": "my_agent_eval_set",
  "evals": [
    {
      "eval_id": "test_basic_greeting",
      "conversation": [
        {
          "query": "Hello!",
          "expected_tool_use": [],
          "expected_intermediate_agent_responses": [],
          "reference": "Hello! How can I help you today?"
        }
      ],
      "initial_session": {
        "state": {},
        "app_name": "my_app",
        "user_id": "eval_user"
      }
    },
    {
      "eval_id": "test_weather_query",
      "conversation": [
        {
          "query": "What's the weather in Paris?",
          "expected_tool_use": [
            {
              "tool_name": "get_weather",
              "tool_input": {"city": "Paris"}
            }
          ],
          "expected_intermediate_agent_responses": [],
          "reference": "The weather in Paris is sunny with a temperature of 25 degrees Celsius."
        }
      ],
      "initial_session": {
        "state": {"user_name": "TestUser"},
        "app_name": "my_app",
        "user_id": "eval_user"
      }
    }
  ]
}
```

### Multi-Turn Evaluation
```json
{
  "eval_id": "test_multi_turn_conversation",
  "conversation": [
    {
      "query": "What's the weather in Paris?",
      "expected_tool_use": [
        {"tool_name": "get_weather", "tool_input": {"city": "Paris"}}
      ],
      "reference": "It's sunny and 25°C in Paris."
    },
    {
      "query": "What about London?",
      "expected_tool_use": [
        {"tool_name": "get_weather", "tool_input": {"city": "London"}}
      ],
      "reference": "London is cloudy with 18°C and a chance of rain."
    },
    {
      "query": "Which city is warmer?",
      "expected_tool_use": [],
      "reference": "Paris is warmer at 25°C compared to London's 18°C."
    }
  ]
}
```

### Multi-Agent Evaluation
```json
{
  "eval_id": "test_agent_delegation",
  "conversation": [
    {
      "query": "I need help with my bill",
      "expected_tool_use": [
        {"tool_name": "transfer_to_agent", "tool_input": {"agent_name": "billing"}}
      ],
      "expected_intermediate_agent_responses": [
        "Let me transfer you to our billing specialist."
      ],
      "reference": "I'll connect you with our billing department."
    }
  ]
}
```

## 2. Running Evaluations via CLI

### Basic Usage
```bash
adk eval \
    ./my_agent \
    ./my_agent/eval_sets/basic_eval.evalset.json
```

### Run Specific Evals
```bash
adk eval \
    ./my_agent \
    ./my_agent/eval_sets/basic_eval.evalset.json:test_greeting,test_weather
```

### Multiple Eval Files
```bash
adk eval \
    ./my_agent \
    ./my_agent/eval_sets/basic_eval.evalset.json \
    ./my_agent/eval_sets/advanced_eval.evalset.json
```

## 3. Evaluation Metrics

### Tool Use Accuracy
Checks if the agent called the correct tools with expected arguments:
- **Tool name match**: Did the agent use the right tool?
- **Tool input match**: Were the arguments correct?

### Response Quality
Compares agent's response against the reference answer:
- **Semantic similarity**: Is the meaning similar?
- **Key information presence**: Are critical facts included?

### Intermediate Response Accuracy
For multi-agent systems, checks if intermediate responses match expectations.

## 4. Dev UI Evaluation

### Capturing Sessions as Evals
1. Start dev UI: `adk web ./my_agent`
2. Have conversations with the agent
3. Use the UI to convert good sessions into eval entries
4. Export as `.evalset.json`

### Running Evals in UI
1. Select test cases from your evalset
2. Run evaluations
3. View results with pass/fail indicators
4. Inspect detailed metrics per turn

## 5. Custom Python Testing

### Unit Testing Agent Responses
```python
import asyncio
import pytest
from google.adk.runners import InMemoryRunner
from google.genai import types

async def run_agent_query(agent, query: str) -> str:
    """Helper to run a single query and get response."""
    runner = InMemoryRunner(agent=agent, app_name="test_app")
    content = types.Content(role='user', parts=[types.Part(text=query)])

    response_text = ""
    async for event in runner.run_async(
        user_id="test_user",
        session_id="test_session",
        new_message=content
    ):
        if event.is_final_response() and event.content and event.content.parts:
            response_text = event.content.parts[0].text
    return response_text

# Using pytest
@pytest.mark.asyncio
async def test_greeting():
    from my_agent.agent import root_agent
    response = await run_agent_query(root_agent, "Hello!")
    assert "hello" in response.lower() or "hi" in response.lower()

@pytest.mark.asyncio
async def test_weather_tool_used():
    from my_agent.agent import root_agent
    runner = InMemoryRunner(agent=root_agent, app_name="test")
    content = types.Content(role='user', parts=[types.Part(text="Weather in Paris?")])

    tool_called = False
    async for event in runner.run_async(
        user_id="test", session_id="test", new_message=content
    ):
        if event.content and event.content.parts:
            for part in event.content.parts:
                if part.function_call and part.function_call.name == "get_weather":
                    tool_called = True
    assert tool_called, "Expected get_weather tool to be called"
```

### Testing Callbacks
```python
from unittest.mock import MagicMock

def test_input_guardrail():
    """Test that guardrail blocks forbidden keywords."""
    from my_agent.callbacks import input_guardrail
    from google.adk.models import LlmRequest
    from google.genai import types

    # Mock context
    mock_context = MagicMock()
    mock_context.state = {}

    # Create request with blocked content
    request = LlmRequest(
        contents=[
            types.Content(role='user', parts=[types.Part(text="How to hack a system")])
        ]
    )

    result = input_guardrail(mock_context, request)
    assert result is not None  # Should return LlmResponse to block
```

### Testing State Flow
```python
@pytest.mark.asyncio
async def test_state_passing():
    """Test that data passes correctly between sequential agents."""
    runner = InMemoryRunner(agent=pipeline_agent, app_name="test")

    content = types.Content(role='user', parts=[types.Part(text="Process this")])
    async for event in runner.run_async(
        user_id="test", session_id="test", new_message=content
    ):
        pass  # Process all events

    session = await runner.session_service.get_session(
        app_name="test", user_id="test", session_id="test"
    )
    assert "result" in session.state
    assert session.state["result"] is not None
```

## 6. Project Structure for Testing

```
my_agent/
├── __init__.py
├── agent.py
├── tools/
│   └── weather.py
├── callbacks/
│   └── guardrails.py
├── eval_sets/
│   ├── basic_eval.evalset.json
│   ├── tool_use_eval.evalset.json
│   └── multi_turn_eval.evalset.json
├── tests/
│   ├── test_tools.py
│   ├── test_callbacks.py
│   └── test_agent_flow.py
└── requirements.txt
```

## Evaluation Best Practices

1. **Start with simple single-turn evals** before adding multi-turn complexity
2. **Test tool selection** — verify the agent picks the right tool
3. **Test error handling** — include queries that should trigger error paths
4. **Test edge cases** — empty inputs, very long inputs, unexpected formats
5. **Use `initial_session.state`** in eval sets to test state-dependent behavior
6. **Version your eval sets** alongside your agent code
7. **Run evals in CI/CD** — `adk eval` returns non-zero on failures
8. **Monitor drift** — re-run evals when changing models or instructions
9. **Test multi-agent routing** — verify delegation works correctly
10. **Keep reference answers flexible** — use semantic similarity, not exact match

## Additional Resources

- **`references/adk-eval-full.md`** — Complete ADK evaluation documentation with examples and metrics
