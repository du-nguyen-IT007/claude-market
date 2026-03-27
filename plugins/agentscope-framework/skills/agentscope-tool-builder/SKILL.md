---
name: agentscope-tool-builder
description: >
  Create custom tools, toolkits, and MCP integrations for AgentScope agents.
  Use when the user asks to "add a tool", "create a custom tool", "integrate MCP",
  "register tool function", "add MCP server to agent", "use MCP tools in AgentScope",
  or needs guidance on Toolkit, tool schema generation, and MCP client setup.
metadata:
  version: "0.1.0"
  framework: "agentscope-python"
---

# AgentScope Tool Builder

Create tools and integrate MCP servers into AgentScope agents.

## Custom Tool Functions

AgentScope auto-generates JSON schema from Python docstring + type hints.

```python
from agentscope.tool import Toolkit

# Rule: Viết docstring rõ ràng — đây là source of truth cho schema
def search_web(query: str, max_results: int = 5) -> str:
    """Search the web for up-to-date information.

    Args:
        query: The search query string.
        max_results: Maximum number of results to return (default: 5).

    Returns:
        Search results as a formatted string.
    """
    # Implementation here
    return f"Results for: {query}"

def read_file(file_path: str) -> str:
    """Read content from a file on disk.

    Args:
        file_path: Absolute or relative path to the file.

    Returns:
        File content as string, or error message if file not found.
    """
    try:
        with open(file_path, "r") as f:
            return f.read()
    except FileNotFoundError:
        return f"Error: File not found at {file_path}"

# Register vào toolkit
toolkit = Toolkit()
toolkit.register_tool_function(search_web)
toolkit.register_tool_function(read_file)
```

## Built-in Tools

```python
from agentscope.tool import (
    Toolkit,
    execute_python_code,    # Chạy Python code trong sandbox
    execute_shell_command,  # Chạy shell commands
)

toolkit = Toolkit()
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(execute_shell_command)
```

## MCP Integration

### Stateless Client (dùng cho hầu hết cases)

```python
from agentscope.mcp import HttpStatelessClient
from agentscope.tool import Toolkit

# Khởi tạo client
client = HttpStatelessClient(
    name="my_mcp_server",
    url="http://localhost:8000/mcp",
)

# Option 1: Đăng ký toàn bộ MCP tools vào toolkit
toolkit = Toolkit()
await toolkit.register_mcp_client(client)

# Option 2: Lấy specific function và gọi trực tiếp
func = await client.get_callable_function(func_name="search_documents")
result = await func(query="agentscope examples")
```

### Stateful Client (dùng khi server cần session state)

```python
from agentscope.mcp import HttpStatefulClient

client = HttpStatefulClient(
    name="stateful_server",
    url="http://localhost:8001/mcp",
)

await client.connect()
try:
    func = await client.get_callable_function("process_document")
    result = await func(doc_id="doc_123")
finally:
    await client.close()
```

### Agent với MCP Tools

```python
import asyncio
from agentscope.agent import ReActAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit
from agentscope.mcp import HttpStatelessClient

async def main():
    # Setup MCP
    client = HttpStatelessClient("tools_server", "http://mcp-server:8000/mcp")
    toolkit = Toolkit()
    await toolkit.register_mcp_client(client)

    # Thêm custom tools
    toolkit.register_tool_function(search_web)

    # Agent với cả MCP + custom tools
    agent = ReActAgent(
        name="PowerAgent",
        sys_prompt="You can access external systems and search the web.",
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
        ),
        memory=InMemoryMemory(),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
    )
    # ... rest of logic

asyncio.run(main())
```

## Tool Best Practices

```python
# ✅ GOOD — Type hints + clear docstring
def calculate_roi(
    investment: float,
    returns: float,
    period_months: int = 12
) -> dict:
    """Calculate Return on Investment (ROI) metrics.

    Args:
        investment: Initial investment amount in USD.
        returns: Total returns received in USD.
        period_months: Investment period in months (default: 12).

    Returns:
        Dictionary with roi_percentage, annualized_roi, and period_months.
    """
    roi = (returns - investment) / investment * 100
    annualized = roi * (12 / period_months)
    return {
        "roi_percentage": round(roi, 2),
        "annualized_roi": round(annualized, 2),
        "period_months": period_months
    }

# ❌ BAD — Không có type hints, docstring mơ hồ
def calc(a, b, c=12):
    """Calculate something."""
    return (b - a) / a * 100
```

## MCP Client Comparison

| | `HttpStatelessClient` | `HttpStatefulClient` |
|---|---|---|
| Connection | Per-request | Persistent |
| Server state | Không cần | Cần session |
| Use case | REST-style tools | Stateful workflows |
| Setup | Đơn giản | Cần connect/close |

## Additional Resources

- **`references/agentscope-tools-full.md`** — Full tool và MCP reference
