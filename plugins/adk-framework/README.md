# ADK Framework Plugin

Google Agent Development Kit (ADK) plugin for vibe coding AI agents. Built from the official [google/adk-python](https://github.com/google/adk-python) documentation (`llms-full.txt`).

## Overview

This plugin provides 6 specialized skills covering the complete ADK development lifecycle — from agent design to production deployment.

## Skills

| Skill | Trigger Phrases | Description |
|-------|----------------|-------------|
| **adk-agent-architect** | "create agent", "multi-agent system", "scaffold ADK project", "workflow agents" | Design and scaffold all agent types: LlmAgent, SequentialAgent, ParallelAgent, LoopAgent, Custom agents, and multi-agent patterns |
| **adk-tool-builder** | "create tool", "function tool", "integrate API", "OpenAPI", "toolset" | Build custom tools, toolsets, integrate built-in tools (Google Search, Code Execution), OpenAPI specs, and third-party tools (LangChain, CrewAI) |
| **adk-callback-guardrails** | "add guardrails", "callbacks", "safety checks", "input filtering" | Implement lifecycle hooks: before/after agent, model, and tool callbacks for guardrails, caching, logging, and flow control |
| **adk-session-memory** | "session state", "pass data", "artifacts", "memory", "context" | Manage sessions, state (with prefixes), artifacts, context objects (InvocationContext, ToolContext, CallbackContext), and long-term memory |
| **adk-deploy** | "deploy agent", "Cloud Run", "Vertex AI", "GKE", "production" | Deploy to Vertex AI Agent Engine, Cloud Run (adk CLI or gcloud), and GKE with complete configurations |
| **adk-evaluation** | "test agent", "evaluate", "eval set", "benchmark", "adk eval" | Create eval sets, run evaluations via CLI/UI, write custom Python tests, and measure agent quality |

## Setup

```bash
pip install google-adk
export GOOGLE_API_KEY="your-key"  # For AI Studio
# OR
export GOOGLE_GENAI_USE_VERTEXAI=True  # For Vertex AI
```

## Quick Start

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="my_assistant",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant.",
    tools=[]
)
```

```bash
adk web ./my_agent  # Start dev UI
```

## Reference Data

Each skill includes a `references/` directory containing extracted sections from the official ADK documentation (~33K lines), providing comprehensive code examples and API details for on-demand loading.

## Source

Built from [google/adk-python llms-full.txt](https://github.com/google/adk-python/blob/main/llms-full.txt) — the official context file recommended by Google for vibe coding with ADK.
