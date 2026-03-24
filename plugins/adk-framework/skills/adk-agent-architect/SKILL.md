---
name: adk-agent-architect
description: >
  Design, scaffold, and implement AI agents using Google ADK (Agent Development Kit).
  Use when the user asks to "create an agent", "build a multi-agent system",
  "scaffold an ADK project", "design agent architecture", "create workflow agents",
  "set up sequential/parallel/loop agent", "vibe code an AI agent", or needs
  guidance on agent types, hierarchy, and multi-agent patterns with Google ADK.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Agent Architect

Design and scaffold AI agents using Google's Agent Development Kit (ADK). This skill covers all agent types, multi-agent patterns, and project scaffolding for vibe coding.

## Prerequisites

```bash
pip install google-adk
```

Set environment variable:
```bash
export GOOGLE_GENAI_USE_VERTEXAI=True  # or False for AI Studio
export GOOGLE_API_KEY="your-key"        # if using AI Studio
```

## Core Agent Types

### 1. LlmAgent (alias: Agent)
The "thinking" agent powered by an LLM. Use for reasoning, tool use, and dynamic decisions.

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="assistant",
    model="gemini-2.5-flash",
    description="A helpful assistant that answers questions.",
    instruction="""You are a helpful assistant.
    When asked about weather, use the get_weather tool.
    Respond concisely and accurately.""",
    tools=[get_weather]
)
```

Key parameters:
- `name` (required): Unique identifier, avoid "user"
- `model` (required): e.g., `"gemini-2.5-flash"`, `"gemini-2.5-pro"`
- `description`: Used by OTHER agents to decide routing
- `instruction`: Core behavior guide — supports `{state_var}` template syntax
- `tools`: List of functions, BaseTool instances, or AgentTool wrappers
- `output_key`: Auto-save final response to `session.state[output_key]`
- `output_schema`: Pydantic BaseModel for structured JSON output (disables tool use)
- `input_schema`: Pydantic BaseModel for expected input format
- `include_contents`: `'default'` or `'none'` (stateless mode)
- `generate_content_config`: Control temperature, max_output_tokens, etc.

### 2. Workflow Agents (Deterministic, no LLM for flow control)

**SequentialAgent** — fixed-order pipeline:
```python
from google.adk.agents import SequentialAgent, LlmAgent

step1 = LlmAgent(name="fetch", model="gemini-2.5-flash",
                  instruction="Fetch data...", output_key="raw_data")
step2 = LlmAgent(name="process", model="gemini-2.5-flash",
                  instruction="Process {raw_data}...", output_key="result")

pipeline = SequentialAgent(name="pipeline", sub_agents=[step1, step2])
```

**ParallelAgent** — concurrent execution:
```python
from google.adk.agents import ParallelAgent

fetch_a = LlmAgent(name="fetch_a", output_key="data_a", ...)
fetch_b = LlmAgent(name="fetch_b", output_key="data_b", ...)

gatherer = ParallelAgent(name="gather", sub_agents=[fetch_a, fetch_b])
```

**LoopAgent** — iterative refinement:
```python
from google.adk.agents import LoopAgent

def exit_loop(tool_context: ToolContext):
    """Call when refinement is complete."""
    tool_context.actions.escalate = True
    return {}

writer = LlmAgent(name="writer", output_key="draft", ...)
critic = LlmAgent(name="critic", output_key="feedback", ...)
refiner = LlmAgent(name="refiner", tools=[exit_loop], output_key="draft", ...)

loop = LoopAgent(name="refine_loop", sub_agents=[critic, refiner], max_iterations=5)

# Full pipeline: write once, then loop refine
root_agent = SequentialAgent(name="pipeline", sub_agents=[writer, loop])
```

### 3. Custom Agents
Extend `BaseAgent` for full control:
```python
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event, EventActions
from typing import AsyncGenerator

class MyCustomAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        status = ctx.session.state.get("status", "pending")
        if status == "done":
            yield Event(author=self.name, actions=EventActions(escalate=True))
        else:
            # Run sub-agents or custom logic
            for sub_agent in self.sub_agents:
                async for event in sub_agent.run_async(ctx):
                    yield event
```

## Multi-Agent Patterns

### Coordinator/Dispatcher (LLM-driven routing)
```python
billing = LlmAgent(name="billing", description="Handles billing inquiries.", ...)
support = LlmAgent(name="support", description="Handles technical issues.", ...)

coordinator = LlmAgent(
    name="coordinator",
    model="gemini-2.5-flash",
    instruction="Route billing questions to billing agent, tech issues to support.",
    sub_agents=[billing, support]
)
```
The LLM auto-generates `transfer_to_agent(agent_name='billing')` calls.

### Agent-as-Tool (explicit invocation)
```python
from google.adk.tools import agent_tool

specialist = LlmAgent(name="analyst", description="Deep data analysis", ...)
analyst_tool = agent_tool.AgentTool(agent=specialist)

main_agent = LlmAgent(
    name="main",
    model="gemini-2.5-flash",
    instruction="Use the analyst tool for complex data questions.",
    tools=[analyst_tool]
)
```

### Sequential Pipeline with State Passing
Use `output_key` on each agent to pass data via shared `session.state`:
```python
validator = LlmAgent(name="validate", output_key="validation_status", ...)
processor = LlmAgent(name="process", instruction="If {validation_status} is valid, process...", output_key="result", ...)
reporter = LlmAgent(name="report", instruction="Report: {result}", ...)

pipeline = SequentialAgent(name="data_pipeline", sub_agents=[validator, processor, reporter])
```

### Parallel Fan-Out / Gather
```python
parallel_fetch = ParallelAgent(name="fetch_all", sub_agents=[fetcher1, fetcher2, fetcher3])
synthesizer = LlmAgent(name="synthesize", instruction="Combine {data_1}, {data_2}, {data_3}...")

workflow = SequentialAgent(name="fan_out_gather", sub_agents=[parallel_fetch, synthesizer])
```

### Iterative Refinement (Generator-Critic)
```python
generator = LlmAgent(name="writer", output_key="draft", ...)
critic = LlmAgent(name="critic", output_key="feedback", ...)
refiner = LlmAgent(name="refiner", tools=[exit_loop], output_key="draft", ...)

loop = LoopAgent(name="refine", sub_agents=[critic, refiner], max_iterations=5)
pipeline = SequentialAgent(name="write_and_refine", sub_agents=[generator, loop])
```

## Project Structure

Standard ADK project layout:
```
my_agent/
├── __init__.py          # from . import agent
├── agent.py             # Must export root_agent
└── requirements.txt     # google-adk, other deps
```

For multi-agent:
```
my_agents/
├── __init__.py
├── agent.py             # root_agent = SequentialAgent(...)
├── agents/
│   ├── researcher.py
│   ├── writer.py
│   └── reviewer.py
└── tools/
    ├── search.py
    └── database.py
```

## Running Agents

### Dev UI
```bash
adk web my_agent/
# Opens at http://localhost:8000
```

### Programmatic
```python
from google.adk.runners import Runner, InMemoryRunner
from google.adk.sessions import InMemorySessionService
from google.genai import types

runner = InMemoryRunner(agent=root_agent, app_name="my_app")

async def chat(query: str):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    async for event in runner.run_async(
        user_id="user1", session_id="session1", new_message=content
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(event.content.parts[0].text)
```

## Using Different Models

### Gemini (default)
```python
agent = Agent(name="a", model="gemini-2.5-flash", ...)
```

### Anthropic via LiteLLM
```python
from google.adk.models.lite_llm import LiteLlm
agent = Agent(name="a", model=LiteLlm(model="anthropic/claude-sonnet-4-20250514"), ...)
```

### OpenAI via LiteLLM
```python
agent = Agent(name="a", model=LiteLlm(model="openai/gpt-4o"), ...)
```

### Local models via LiteLLM + Ollama
```python
agent = Agent(name="a", model=LiteLlm(model="ollama_chat/llama3"), ...)
```

## Instruction Template Syntax

Use `{var}` to inject state variables into instructions:
```python
agent = LlmAgent(
    instruction="Help the user. Their preference is {user_preference}. Topic: {current_topic?}",
    # {var?} = optional, won't error if missing
)
```

Use `{artifact.name}` to inject artifact text content.

## Decision Guide

| Need | Agent Type |
|------|-----------|
| Reasoning, tool use, NL understanding | `LlmAgent` / `Agent` |
| Fixed-order multi-step pipeline | `SequentialAgent` |
| Parallel independent tasks | `ParallelAgent` |
| Iterative refinement with exit condition | `LoopAgent` |
| Dynamic routing to specialists | `LlmAgent` with `sub_agents` |
| Complex custom orchestration | Custom `BaseAgent` subclass |
| One agent calling another as tool | `AgentTool` wrapper |

## Additional Resources

- **`references/adk-agents-full.md`** — Complete ADK agents documentation with all code examples
