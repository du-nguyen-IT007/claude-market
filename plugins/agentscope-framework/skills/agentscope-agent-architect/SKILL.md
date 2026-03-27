---
name: agentscope-agent-architect
description: >
  Design, scaffold, and implement AI agents using AgentScope (Python).
  Use when the user asks to "create an agent", "build a ReActAgent",
  "scaffold an AgentScope project", "design agent architecture",
  "build a multi-agent system", "vibe code an AI agent with AgentScope",
  or needs guidance on agent types, MsgHub, and multi-agent patterns.
metadata:
  version: "0.1.0"
  framework: "agentscope-python"
---

# AgentScope Agent Architect

Design and scaffold AI agents using AgentScope. Covers ReActAgent, UserAgent, multi-agent patterns, and project structure.

## Prerequisites

```bash
pip install agentscope
# Python 3.10+ required
```

Set environment variables:
```bash
export DASHSCOPE_API_KEY="your-key"   # Alibaba DashScope (Qwen)
export OPENAI_API_KEY="your-key"      # OpenAI
export ANTHROPIC_API_KEY="your-key"   # Anthropic Claude
```

## Core Agent: ReActAgent

The primary agent following Reason + Act pattern.

```python
import os, asyncio
from agentscope.agent import ReActAgent, UserAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit, execute_python_code, execute_shell_command

async def main():
    toolkit = Toolkit()
    toolkit.register_tool_function(execute_python_code)
    toolkit.register_tool_function(execute_shell_command)

    agent = ReActAgent(
        name="Friday",
        sys_prompt="You're a helpful assistant named Friday.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        ),
        memory=InMemoryMemory(),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
    )

    user = UserAgent(name="user")
    msg = None
    while True:
        msg = await agent(msg)
        msg = await user(msg)
        if msg.get_text_content() == "exit":
            break

asyncio.run(main())
```

## Supported Models

```python
from agentscope.model import (
    DashScopeChatModel,    # Alibaba Qwen (qwen-max, qwen-plus, ...)
    OpenAIChatModel,       # OpenAI (gpt-4o, gpt-4o-mini, ...)
    AnthropicChatModel,    # Anthropic (claude-sonnet-4-20250514, ...)
    GeminiChatModel,       # Google Gemini
    OllamaChatModel,       # Local models via Ollama
)
```

## Multi-Agent với MsgHub

MsgHub auto-broadcasts messages to all participants — no manual routing needed.

```python
from agentscope.agent import ReActAgent
from agentscope.pipeline import MsgHub
from agentscope.message import Msg
from agentscope.formatter import DashScopeMultiAgentFormatter

model = DashScopeChatModel(model_name="qwen-max", api_key=os.environ["DASHSCOPE_API_KEY"])
formatter = DashScopeMultiAgentFormatter()  # MUST use MultiAgent formatter

def make_agent(name: str, role: str) -> ReActAgent:
    return ReActAgent(
        name=name,
        sys_prompt=f"You are {name}. {role}",
        model=model,
        formatter=formatter,
        toolkit=Toolkit(),
        memory=InMemoryMemory(),
    )

alice = make_agent("Alice", "A data scientist who proposes solutions.")
bob   = make_agent("Bob",   "A skeptic who challenges assumptions.")
carol = make_agent("Carol", "A moderator who summarizes discussion.")

async def run():
    async with MsgHub(
        [alice, bob, carol],
        announcement=Msg("system", "Topic: Best ML framework for production?", "system"),
    ):
        await alice()
        await bob()
        await carol()

asyncio.run(run())
```

## Dynamic MsgHub Management

```python
async with MsgHub(participants=[alice, bob]) as hub:
    await alice()
    hub.add(carol)          # Add participant
    hub.delete(alice)       # Remove participant
    await hub.broadcast(Msg("system", "New context added.", "system"))
    await bob()
    await carol()
```

## Structured Output

```python
from pydantic import BaseModel

class AnalysisResult(BaseModel):
    summary: str
    key_findings: list[str]
    confidence_score: float

agent = ReActAgent(
    name="Analyst",
    sys_prompt="Analyze and return structured results.",
    model=model,
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
    output_format=AnalysisResult,  # Type-safe output
)
```

## Project Structure

```
my_agent/
├── main.py          # Entry point with asyncio.run(main())
├── agents/
│   ├── coordinator.py
│   └── specialists.py
├── tools/
│   └── custom_tools.py
└── requirements.txt # agentscope
```

## Decision Guide

| Need | Solution |
|------|----------|
| Single agent với tools | `ReActAgent` + `Toolkit` |
| Multi-agent conversation | `MsgHub` + multiple `ReActAgent` |
| Fixed pipeline A→B→C | `sequential_pipeline` trong `MsgHub` |
| Parallel processing | `fanout_pipeline` |
| Type-safe output | `output_format=PydanticModel` |
| Human interaction | `UserAgent` trong vòng lặp |

## Critical Rules cho Vibecoding

1. **Luôn `async/await`** — AgentScope hoàn toàn async
2. **`asyncio.run(main())`** ở cuối file
3. **Multi-agent dùng `DashScopeMultiAgentFormatter`**, không phải `DashScopeChatFormatter`
4. **`async with MsgHub(...)`** — dùng context manager để cleanup
5. **Docstring rõ ràng cho tools** — schema được tự generate từ docstring + type hints

## Additional Resources

- **`references/agentscope-agents-full.md`** — Full agent reference với tất cả code examples
