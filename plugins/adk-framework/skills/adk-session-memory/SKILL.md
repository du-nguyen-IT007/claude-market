---
name: adk-session-memory
description: >
  Manage sessions, state, artifacts, context, and long-term memory for Google ADK
  agents. Use when the user asks to "manage session state", "pass data between agents",
  "store artifacts", "implement memory", "use session service", "persist state",
  "share data between tools", "manage conversation history", "use InvocationContext",
  "use ToolContext", "use CallbackContext", or needs guidance on state management,
  data flow, and memory in ADK.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Session & Memory

Manage sessions, state, artifacts, context objects, and long-term memory in Google ADK agents.

## Context Objects Hierarchy

| Context | Where Used | Can Write State | Has Artifacts | Has Auth | Has Memory |
|---------|-----------|----------------|--------------|---------|-----------|
| `InvocationContext` | Agent's `_run_async_impl` | Yes (via session) | Via service | Via service | Via service |
| `ReadonlyContext` | Instruction providers | No (read-only) | No | No | No |
| `CallbackContext` | Agent/Model callbacks | Yes | Yes | No | No |
| `ToolContext` | Tool functions, tool callbacks | Yes | Yes | Yes | Yes |

## 1. Session Management

### InMemorySessionService (Development)
```python
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()

# Create session
session = await session_service.create_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456"
)

# Get session
session = await session_service.get_session(
    app_name="my_app", user_id="user_123", session_id="session_456"
)

# List sessions
sessions = await session_service.list_sessions(
    app_name="my_app", user_id="user_123"
)
```

### DatabaseSessionService (Production)
```python
from google.adk.sessions import DatabaseSessionService

session_service = DatabaseSessionService(db_url="sqlite:///./sessions.db")
# Or PostgreSQL: "postgresql://user:pass@host/db"
```

### VertexAiSessionService (Cloud)
```python
from google.adk.sessions import VertexAiSessionService

session_service = VertexAiSessionService(
    project="my-project", location="us-central1"
)
```

### Session with Initial State
```python
session = await session_service.create_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_789",
    state={
        "user_name": "Adward",
        "temperature_unit": "celsius",
        "app:global_config": "production"
    }
)
```

## 2. State Management

### State Prefixes
| Prefix | Scope | Example |
|--------|-------|---------|
| (none) | Current session only | `state["cart_items"]` |
| `app:` | All users, all sessions | `state["app:api_version"]` |
| `user:` | Current user, all sessions | `state["user:preferences"]` |
| `temp:` | Current invocation only | `state["temp:step_result"]` |

### Reading/Writing State in Tools
```python
from google.adk.tools import ToolContext

def my_tool(query: str, tool_context: ToolContext) -> dict:
    """Process a query using session state."""
    # Read state
    user_name = tool_context.state.get("user_name", "Guest")
    prefs = tool_context.state.get("user:preferences", {})

    # Write state — automatically tracked in event's state_delta
    tool_context.state["last_query"] = query
    tool_context.state["temp:processing_flag"] = True

    return {"status": "success", "greeting": f"Hello {user_name}"}
```

### Reading/Writing State in Callbacks
```python
from google.adk.agents.callback_context import CallbackContext

def my_callback(callback_context: CallbackContext, **kwargs):
    # Read
    count = callback_context.state.get("call_count", 0)
    # Write
    callback_context.state["call_count"] = count + 1
```

### Passing Data Between Agents via output_key
```python
from google.adk.agents import SequentialAgent, LlmAgent

fetcher = LlmAgent(
    name="fetcher",
    model="gemini-2.5-flash",
    instruction="Fetch data about the topic.",
    output_key="raw_data"  # Saves response to state["raw_data"]
)

analyzer = LlmAgent(
    name="analyzer",
    model="gemini-2.5-flash",
    instruction="Analyze this data: {raw_data}",  # Reads from state
    output_key="analysis"
)

reporter = LlmAgent(
    name="reporter",
    model="gemini-2.5-flash",
    instruction="Create a report from: {analysis}"
)

pipeline = SequentialAgent(name="pipeline", sub_agents=[fetcher, analyzer, reporter])
```

### Instruction Template Variables
```python
agent = LlmAgent(
    instruction="""
    User: {user_name}
    Preference: {user:preferences?}
    Last result: {temp:last_result?}
    """,
    # {var?} = optional, won't error if missing
)
```

## 3. Artifacts

Store and retrieve files/blobs associated with a session.

### Artifact Services

**InMemoryArtifactService** (Development):
```python
from google.adk.artifacts import InMemoryArtifactService
artifact_service = InMemoryArtifactService()
```

**GcsArtifactService** (Production):
```python
from google.adk.artifacts import GcsArtifactService
artifact_service = GcsArtifactService(bucket_name="my-bucket")
```

### Using Artifacts in Tools
```python
from google.adk.tools import ToolContext
from google.genai import types

def save_report(report_text: str, tool_context: ToolContext) -> dict:
    """Save a report as an artifact."""
    part = types.Part(text=report_text)
    version = tool_context.save_artifact("report.txt", part)
    return {"status": "success", "version": version}

def load_report(tool_context: ToolContext) -> dict:
    """Load the latest report artifact."""
    part = tool_context.load_artifact("report.txt")
    if not part:
        return {"status": "error", "message": "Report not found"}
    return {"status": "success", "content": part.text}

def list_files(tool_context: ToolContext) -> dict:
    """List all available artifacts."""
    files = tool_context.list_artifacts()
    return {"status": "success", "files": files}
```

### Using Artifacts in Callbacks
```python
def save_artifact_callback(callback_context: CallbackContext, **kwargs):
    config_part = types.Part(text='{"setting": "value"}')
    version = callback_context.save_artifact("config.json", config_part)

    loaded = callback_context.load_artifact("config.json")
```

### Binary Artifacts (Images, PDFs)
```python
def save_image(image_bytes: bytes, tool_context: ToolContext) -> dict:
    """Save an image artifact."""
    part = types.Part.from_bytes(image_bytes, "image/png")
    version = tool_context.save_artifact("output.png", part)
    return {"status": "success", "version": version}
```

## 4. Runner Setup

### Basic Runner
```python
from google.adk.runners import Runner

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service,
    artifact_service=artifact_service,  # Optional
    memory_service=memory_service,      # Optional
)
```

### InMemoryRunner (Quick Development)
```python
from google.adk.runners import InMemoryRunner

runner = InMemoryRunner(agent=root_agent, app_name="my_app")
# Bundles InMemorySessionService internally
```

### Running an Agent
```python
from google.genai import types

async def chat(query: str, user_id: str, session_id: str):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    async for event in runner.run_async(
        user_id=user_id, session_id=session_id, new_message=content
    ):
        if event.is_final_response() and event.content and event.content.parts:
            return event.content.parts[0].text
```

## 5. Long-Term Memory (MemoryService)

### Vertex AI RAG Memory
```python
from google.adk.memory import VertexAiRagMemoryService

memory_service = VertexAiRagMemoryService(
    rag_corpus="projects/123/locations/us-central1/ragCorpora/456"
)
```

### Searching Memory in Tools
```python
def research_topic(topic: str, tool_context: ToolContext) -> dict:
    """Research a topic using long-term memory."""
    results = tool_context.search_memory(f"Information about {topic}")
    if results.memories:
        snippets = [m.events[0].content.parts[0].text
                    for m in results.memories
                    if m.events and m.events[0].content]
        return {"status": "success", "findings": snippets}
    return {"status": "no_results", "message": "No relevant memories found."}
```

## 6. InvocationContext (Custom Agents)

Direct access in `_run_async_impl`:
```python
class MyAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        # Full access
        session = ctx.session
        state = ctx.session.state
        agent_name = ctx.agent.name
        inv_id = ctx.invocation_id
        user_msg = ctx.user_content

        # Services
        artifact_svc = ctx.artifact_service
        memory_svc = ctx.memory_service

        # End early
        if state.get("critical_error"):
            ctx.end_invocation = True
            yield Event(author=self.name, content="Stopping due to error.")
            return
```

## Data Flow Patterns

### Tool → Tool (via state)
```python
def tool_1(tool_context: ToolContext) -> dict:
    tool_context.state["temp:user_id"] = "u123"
    return {"status": "fetched"}

def tool_2(tool_context: ToolContext) -> dict:
    uid = tool_context.state.get("temp:user_id")
    return {"status": "processed", "user": uid}
```

### Agent → Agent (via output_key + state template)
```python
agent_a = LlmAgent(output_key="result_a", ...)
agent_b = LlmAgent(instruction="Work with: {result_a}", ...)
pipeline = SequentialAgent(sub_agents=[agent_a, agent_b])
```

### Callback → Agent (via state)
```python
def before_cb(callback_context):
    callback_context.state["context_info"] = "loaded from callback"
    return None

agent = LlmAgent(
    instruction="Context: {context_info}",
    before_agent_callback=before_cb
)
```

## Best Practices

1. **Use output_key** for passing data between sequential agents
2. **Use state prefixes** (`app:`, `user:`, `temp:`) for clarity and proper scoping
3. **Store file references, not content** in artifacts when possible
4. **Use InMemorySessionService** for development, DatabaseSessionService for production
5. **Don't over-share state** — use distinct keys to avoid collisions in parallel agents
6. **Clean up temp state** when no longer needed

## Additional Resources

- **`references/adk-context-full.md`** — Complete ADK context, sessions, artifacts, and memory documentation
