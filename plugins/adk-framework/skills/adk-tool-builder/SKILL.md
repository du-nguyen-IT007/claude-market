---
name: adk-tool-builder
description: >
  Create custom tools, function tools, toolsets, and integrate third-party tools
  for Google ADK agents. Use when the user asks to "create a tool", "build a
  function tool", "add tools to agent", "integrate API with agent", "create
  toolset", "use OpenAPI spec", "wrap LangChain tool", "wrap CrewAI tool",
  "use google_search tool", "use code execution", or needs guidance on tool
  design patterns for ADK agents.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Tool Builder

Create and integrate tools for Google ADK agents. Tools extend agent capabilities beyond text generation — enabling API calls, database queries, code execution, and external interactions.

## Tool Types Overview

| Type | Use Case | Example |
|------|----------|---------|
| Function Tool | Custom logic, API calls | `def get_weather(city: str) -> dict` |
| Built-in Tool | Google Search, Code Exec | `tools=[google_search]` |
| AgentTool | Another agent as a tool | `AgentTool(agent=specialist)` |
| Long Running Tool | Async operations | Background jobs, polling |
| OpenAPI Tool | REST API integration | Auto-generated from spec |
| Third-party Tool | LangChain, CrewAI wrappers | `LangchainTool(tool=lc_tool)` |
| Toolset | Grouped/dynamic tools | `BigQueryToolset`, custom sets |

## 1. Function Tools (Most Common)

Transform any Python function into a tool. ADK auto-wraps functions in the `tools` list.

### Rules for Effective Tool Functions

**Function Name**: Use descriptive verb-noun names (`get_weather`, `search_documents`, `schedule_meeting`). Avoid generic names.

**Parameters**: Use type hints, JSON-serializable types. NO default values.

**Return Type**: Always return `dict` with a `status` key. Non-dict returns get auto-wrapped as `{'result': value}`.

**Docstring**: Critical — this is what the LLM reads to decide when/how to use the tool. Describe purpose, parameters, return values.

```python
def get_weather(city: str) -> dict:
    """Retrieves the current weather report for a specified city.

    Use this tool when the user asks about weather conditions in a specific location.

    Args:
        city: The name of the city to get weather for.

    Returns:
        dict with keys:
          - status: 'success' or 'error'
          - report: Weather description (on success)
          - error_message: Error details (on error)
    """
    if city.lower() == "paris":
        return {"status": "success", "report": "Sunny, 25°C"}
    return {"status": "error", "error_message": f"No data for {city}"}

# Add to agent
agent = Agent(name="weather_bot", model="gemini-2.5-flash",
              tools=[get_weather],  # Auto-wrapped as FunctionTool
              instruction="Use get_weather when asked about weather.")
```

### Explicit FunctionTool
```python
from google.adk.tools import FunctionTool

weather_tool = FunctionTool(func=get_weather)
agent = Agent(name="bot", tools=[weather_tool], ...)
```

## 2. ToolContext — Context-Aware Tools

Add `tool_context: ToolContext` parameter to access state, artifacts, memory, and control flow. Do NOT document this parameter in the docstring.

```python
from google.adk.tools import ToolContext

def save_preference(preference: str, value: str, tool_context: ToolContext) -> dict:
    """Updates a user preference setting.

    Args:
        preference: The preference name to update.
        value: The new value for the preference.
    """
    # Read/write session state
    tool_context.state[f"user:{preference}"] = value

    # Control agent flow
    # tool_context.actions.transfer_to_agent = "other_agent"
    # tool_context.actions.escalate = True  # Exit LoopAgent
    # tool_context.actions.skip_summarization = True

    return {"status": "success", "updated": preference}
```

### ToolContext Capabilities

| Capability | Usage |
|-----------|-------|
| Read state | `tool_context.state.get("key")` |
| Write state | `tool_context.state["key"] = value` |
| Transfer to agent | `tool_context.actions.transfer_to_agent = "agent_name"` |
| Exit loop | `tool_context.actions.escalate = True` |
| Skip summarization | `tool_context.actions.skip_summarization = True` |
| Load artifact | `tool_context.load_artifact("file.txt")` |
| Save artifact | `tool_context.save_artifact("file.txt", part)` |
| List artifacts | `tool_context.list_artifacts()` |
| Search memory | `tool_context.search_memory("query")` |
| Auth request | `tool_context.request_credential(auth_config)` |
| Get auth response | `tool_context.get_auth_response(auth_config)` |

### State Prefixes
- No prefix: Session-specific
- `app:`: Shared across all users
- `user:`: User-specific across sessions
- `temp:`: Temporary, not persisted across invocations

## 3. Built-in Tools

### Google Search (Gemini 2+ only)
```python
from google.adk.tools import google_search

agent = Agent(name="searcher", model="gemini-2.5-flash",
              tools=[google_search],
              instruction="Search the web to answer questions.")
```

### Code Execution (Gemini 2+ only)
```python
from google.adk.code_executors import BuiltInCodeExecutor

agent = LlmAgent(name="coder", model="gemini-2.5-flash",
                  code_executor=[BuiltInCodeExecutor],
                  instruction="Write and execute Python code to solve problems.")
```

### Vertex AI Search
```python
from google.adk.tools import VertexAiSearchTool

search_tool = VertexAiSearchTool(data_store_id="projects/123/locations/us/collections/default/dataStores/my-ds")
agent = Agent(name="doc_qa", tools=[search_tool], ...)
```

### BigQuery Toolset
```python
from google.adk.tools.bigquery import BigQueryToolset, BigQueryCredentialsConfig, BigQueryToolConfig, WriteMode
import google.auth

creds, _ = google.auth.default()
bq_toolset = BigQueryToolset(
    credentials_config=BigQueryCredentialsConfig(credentials=creds),
    bigquery_tool_config=BigQueryToolConfig(write_mode=WriteMode.BLOCKED)
)
agent = Agent(name="data_agent", tools=[bq_toolset], ...)
```

**Built-in tool limitation**: Only ONE built-in tool per single agent. Use AgentTool to combine:
```python
from google.adk.tools import agent_tool

search_agent = Agent(model="gemini-2.5-flash", name="searcher", tools=[google_search])
code_agent = Agent(model="gemini-2.5-flash", name="coder", code_executor=[BuiltInCodeExecutor])

root_agent = Agent(name="root", model="gemini-2.5-flash",
    tools=[agent_tool.AgentTool(agent=search_agent), agent_tool.AgentTool(agent=code_agent)])
```

## 4. Agent as Tool (AgentTool)

Wrap any agent to be called as a tool by another agent:
```python
from google.adk.tools import agent_tool

specialist = LlmAgent(name="analyst", description="Deep analysis", model="gemini-2.5-flash", ...)
analyst_tool = agent_tool.AgentTool(agent=specialist)

main_agent = LlmAgent(name="main", model="gemini-2.5-flash",
    instruction="Use analyst tool for complex questions.",
    tools=[analyst_tool])
```

## 5. OpenAPI Integration

Auto-generate tools from an OpenAPI spec:
```python
from google.adk.tools.openapi_tool.openapi_spec_parser import OpenApiSpecParser

# From URL or file
toolset = OpenApiSpecParser.parse("https://api.example.com/openapi.json")
# Or from string
toolset = OpenApiSpecParser.parse_from_string(spec_string)

# Each API operation becomes a tool (name = operationId in snake_case)
agent = Agent(name="api_agent", tools=toolset, ...)
```

## 6. Third-Party Tool Integration

### LangChain Tools
```python
from google.adk.tools.langchain_tool import LangchainTool
from langchain_community.tools import TavilySearchResults

lc_tool = TavilySearchResults(max_results=5)
adk_tool = LangchainTool(tool=lc_tool)
agent = Agent(name="searcher", tools=[adk_tool], ...)
```

### CrewAI Tools
```python
from google.adk.tools.crewai_tool import CrewaiTool
from crewai_tools import SerperDevTool

crewai_tool = SerperDevTool()
adk_tool = CrewaiTool(tool=crewai_tool)
agent = Agent(name="researcher", tools=[adk_tool], ...)
```

## 7. Custom Toolsets

Group related tools dynamically:
```python
from google.adk.tools.base_toolset import BaseToolset
from google.adk.tools import FunctionTool, BaseTool
from typing import Optional

class MathToolset(BaseToolset):
    async def get_tools(self, readonly_context=None) -> list[BaseTool]:
        return [FunctionTool(func=add), FunctionTool(func=subtract)]

    async def close(self):
        pass  # Cleanup if needed

agent = Agent(name="math_bot", tools=[MathToolset()], ...)
```

## 8. Sequential Tool Chaining

Guide the agent to use tools in sequence via instructions:
```python
agent = Agent(
    model="gemini-2.5-flash",
    name="weather_mood",
    instruction="""
    1. Use get_weather to get weather for the city
    2. If successful, use analyze_sentiment on the weather report
    3. Combine both results in your response
    """,
    tools=[get_weather, analyze_sentiment]
)
```

## Tool Design Best Practices

1. **One tool, one task** — decompose complex operations into focused tools
2. **Descriptive names** — `get_user_orders` not `process`
3. **Always return dict with status** — `{"status": "success", "data": ...}`
4. **Rich docstrings** — explain when to use, parameters, return values
5. **Type hints on all params** — `def func(city: str, count: int) -> dict`
6. **No default parameter values** — LLM doesn't support them
7. **Handle errors gracefully** — return error dict, don't raise
8. **Keep params simple** — prefer basic types over complex objects

## Additional Resources

- **`references/adk-tools-full.md`** — Complete ADK tools documentation with all code examples
