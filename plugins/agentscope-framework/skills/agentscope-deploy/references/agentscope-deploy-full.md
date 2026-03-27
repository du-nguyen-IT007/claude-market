# AgentScope Framework
> **Mục tiêu:** Tài liệu tham khảo cho Developer và AI (vibecode) để xây dựng Agent applications sử dụng AgentScope Framework một cách nhanh chóng và hiệu quả.

---

## 1. Tổng Quan (Overview)

**AgentScope** là production-ready agent framework được thiết kế để xây dựng các ứng dụng agentic với LLM ngày càng mạnh mẽ. Triết lý cốt lõi: *làm việc cùng với khả năng của model, không ép buộc chúng qua các prompt cứng nhắc.*

### Điểm mạnh chính
- **Built-in ReAct Agent** — Tools, Skills, Memory, Planning, Human-in-the-loop ngay từ đầu
- **Multi-Agent Orchestration** — MsgHub + Pipeline cho phép agent giao tiếp linh hoạt
- **MCP & A2A Protocol** — Tích hợp Model Context Protocol và Agent-to-Agent protocol
- **Production Ready** — OpenTelemetry tracing, Docker/Kubernetes deployment, Serverless support
- **Voice Agent** — Hỗ trợ realtime voice interaction
- **Reinforcement Learning** — Fine-tune agent thông qua RL

---

## 2. Cài Đặt (Installation)

### Yêu cầu
- Python **3.10+**

### Cài từ PyPI
```bash
pip install agentscope

# Hoặc dùng uv (nhanh hơn)
uv pip install agentscope
```

### Cài từ source (để contribute hoặc dùng features mới nhất)
```bash
git clone -b main https://github.com/agentscope-ai/agentscope.git
cd agentscope
pip install -e .
```

---

## 3. Core Concepts — 4 Module Nền Tảng

AgentScope xây dựng mọi thứ trên 4 abstractions cốt lõi:

```
┌─────────────────────────────────────────────────────┐
│                    AGENT (ReActAgent)                │
│                                                     │
│  ┌───────────┐  ┌───────────┐  ┌────────────────┐  │
│  │  MESSAGE  │  │   MODEL   │  │    MEMORY      │  │
│  │  (Msg)    │  │ (LLM API) │  │ (In/LongTerm)  │  │
│  └───────────┘  └───────────┘  └────────────────┘  │
│                     ┌───────────────┐               │
│                     │  TOOL/SKILL   │               │
│                     │  (Toolkit)    │               │
│                     └───────────────┘               │
└─────────────────────────────────────────────────────┘
```

### 3.1 Message (Msg)
Đơn vị giao tiếp cơ bản giữa các agent.

```python
from agentscope.message import Msg

# Tạo message cơ bản
msg = Msg(name="user", content="Hello, what can you do?", role="user")

# Lấy nội dung text
text = msg.get_text_content()
```

### 3.2 Model (LLM Integration)
AgentScope hỗ trợ nhiều LLM provider:

```python
import os
from agentscope.model import (
    DashScopeChatModel,   # Alibaba DashScope (Qwen)
    OpenAIChatModel,      # OpenAI (GPT-4o, ...)
    AnthropicChatModel,   # Anthropic (Claude)
    GeminiChatModel,      # Google Gemini
    OllamaChatModel,      # Local models via Ollama
)

# DashScope (Qwen)
model = DashScopeChatModel(
    model_name="qwen-max",
    api_key=os.environ["DASHSCOPE_API_KEY"],
    stream=True,  # Streaming response
)

# OpenAI
model = OpenAIChatModel(
    model_name="gpt-4o",
    api_key=os.environ["OPENAI_API_KEY"],
)

# Anthropic Claude
model = AnthropicChatModel(
    model_name="claude-sonnet-4-20250514",
    api_key=os.environ["ANTHROPIC_API_KEY"],
)
```

### 3.3 Memory
Quản lý lịch sử hội thoại và ngữ cảnh của agent.

```python
from agentscope.memory import InMemoryMemory, Mem0LongTermMemory

# Short-term memory (trong RAM, reset khi restart)
memory = InMemoryMemory()

# Long-term memory (persistent, dùng mem0 library)
long_memory = Mem0LongTermMemory(
    agent_id="agent_001",      # ID định danh agent
    user_id="user_123",        # ID user
)
```

**Hai chế độ Long-term Memory:**
- `agent_control` — Agent tự quản lý memory thông qua tool calls
- `static_control` — Developer kiểm soát memory operations trực tiếp

### 3.4 Tool & Toolkit
Công cụ mà agent có thể gọi để thực hiện actions.

```python
from agentscope.tool import Toolkit, execute_python_code, execute_shell_command

# Tạo toolkit và đăng ký tools
toolkit = Toolkit()
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(execute_shell_command)

# Đăng ký custom tool function
def search_web(query: str) -> str:
    """Search the web for information.

    Args:
        query: The search query string

    Returns:
        Search results as string
    """
    # Implementation...
    return results

toolkit.register_tool_function(search_web)
```

> **Lưu ý:** AgentScope tự động generate tool schema từ docstring và type hints. Hãy viết docstring rõ ràng.

---

## 4. ReActAgent — Agent Cơ Bản

`ReActAgent` là implementation mặc định theo pattern Reason + Act.

```python
import os
import asyncio
from agentscope.agent import ReActAgent, UserAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit, execute_python_code, execute_shell_command

async def main():
    # 1. Khởi tạo toolkit
    toolkit = Toolkit()
    toolkit.register_tool_function(execute_python_code)
    toolkit.register_tool_function(execute_shell_command)

    # 2. Tạo agent
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

    # 3. Tạo user agent để lấy input từ console
    user = UserAgent(name="user")

    # 4. Vòng lặp hội thoại
    msg = None
    while True:
        msg = await agent(msg)     # Agent xử lý và trả lời
        msg = await user(msg)      # User nhập input
        if msg.get_text_content() == "exit":
            break

asyncio.run(main())
```

### Agent với Human-in-the-Loop
```python
# Cho phép interrupt realtime
agent = ReActAgent(
    name="Friday",
    sys_prompt="...",
    model=model,
    memory=InMemoryMemory(),
    toolkit=toolkit,
    # Human-in-the-loop được hỗ trợ built-in
)

# Agent có thể bị interrupt và resume seamlessly
```

---

## 5. Multi-Agent với MsgHub

`MsgHub` là message broadcast hub — khi một agent reply, tất cả participants khác tự động nhận được message đó.

### Ví dụ Multi-Agent Conversation
```python
import asyncio
from agentscope.agent import ReActAgent
from agentscope.pipeline import MsgHub
from agentscope.message import Msg
from agentscope.formatter import DashScopeMultiAgentFormatter

# Dùng chung model và formatter cho nhiều agent
model = DashScopeChatModel(
    model_name="qwen-max",
    api_key=os.environ["DASHSCOPE_API_KEY"],
)
formatter = DashScopeMultiAgentFormatter()

# Tạo các agent
alice = ReActAgent(
    name="Alice",
    sys_prompt="You're a student named Alice discussing AI trends.",
    model=model,
    formatter=formatter,
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
)

bob = ReActAgent(
    name="Bob",
    sys_prompt="You're a student named Bob, skeptical about AI claims.",
    model=model,
    formatter=formatter,
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
)

charlie = ReActAgent(
    name="Charlie",
    sys_prompt="You're a moderator named Charlie who summarizes debates.",
    model=model,
    formatter=formatter,
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
)

async def run_debate():
    async with MsgHub(
        [alice, bob, charlie],
        announcement=Msg(
            "system",
            "Debate topic: Will AGI be achieved by 2030?",
            "system",
        ),
    ):
        await alice()   # Alice speaks first
        await bob()     # Bob responds (đã nhận message của Alice)
        await charlie() # Charlie summarizes

asyncio.run(run_debate())
```

### Dynamic Participant Management
```python
async with MsgHub(participants=[alice, bob]) as hub:
    await alice()

    # Thêm participant mới
    hub.add(charlie)

    # Xóa participant
    hub.delete(alice)

    # Broadcast message đến tất cả
    await hub.broadcast(
        Msg("system", "New topic: Let's discuss safety.", "system")
    )

    await bob()
    await charlie()
```

---

## 6. Pipeline — Điều Phối Luồng Thực Thi

### Sequential Pipeline
```python
from agentscope.pipeline import sequential_pipeline, MsgHub

async def sequential_workflow():
    async with MsgHub(participants=[agent1, agent2, agent3]) as hub:
        # Thực thi tuần tự
        await sequential_pipeline([agent1, agent2, agent3])
```

### Fanout Pipeline (Parallel)
```python
from agentscope.pipeline import fanout_pipeline

async def parallel_workflow(initial_msg: Msg):
    # Nhiều agent xử lý song song
    results = await fanout_pipeline(
        agents=[researcher, analyst, critic],
        msg=initial_msg,
    )
    return results
```

---

## 7. MCP Integration

AgentScope hỗ trợ Model Context Protocol với **dual-client design**:
- `HttpStatefulClient` — Persistent connection (cho server cần session state)
- `HttpStatelessClient` — Stateless connection (nhẹ hơn, không cần duy trì kết nối)

```python
from agentscope.mcp import HttpStatelessClient, HttpStatefulClient

# Stateless client (dùng cho most cases)
client = HttpStatelessClient(
    name="my_mcp_server",
    url="http://localhost:8000/mcp",
)

# Lấy tool function và gọi trực tiếp
func = await client.get_callable_function(func_name="search_documents")
result = await func(query="AgentScope examples")

# Đăng ký MCP tools vào Toolkit
toolkit = Toolkit()
await toolkit.register_mcp_client(client)

# Bây giờ agent có thể dùng MCP tools
agent = ReActAgent(
    name="Agent",
    sys_prompt="...",
    model=model,
    toolkit=toolkit,
    memory=InMemoryMemory(),
)
```

### Stateful Client
```python
client = HttpStatefulClient(
    name="stateful_server",
    url="http://localhost:8001/mcp",
)

# Cần explicit connect/close
await client.connect()
try:
    func = await client.get_callable_function("process_document")
    result = await func(doc_id="123")
finally:
    await client.close()
```

---

## 8. Agent Skills

**Agent Skills** là lightweight instruction system — cung cấp reusable prompt templates và reference materials để augment agent's system prompt. Khác với tools, skills **không execute code** — chúng cung cấp specialized knowledge.

```python
from agentscope.tool import Toolkit

# Skills được load qua Toolkit
toolkit = Toolkit()

# Đăng ký skill từ file SKILL.md
toolkit.register_skill_from_file(
    skill_name="data_analysis",
    skill_path="./skills/data_analysis/SKILL.md",
)

# Agent sẽ tự động load skill khi cần
agent = ReActAgent(
    name="DataAnalyst",
    sys_prompt="You're a data analyst with specialized skills.",
    model=model,
    toolkit=toolkit,
    memory=InMemoryMemory(),
)
```

**Cấu trúc SKILL.md:**
```markdown
# Data Analysis Skill

## Overview
Specialized knowledge for statistical analysis and visualization.

## When to Use
Use when asked to analyze datasets, create charts, or interpret statistics.

## Instructions
1. Always start by understanding the data structure
2. Identify missing values and outliers
3. Choose appropriate statistical methods
...

## Reference
- Use pandas for data manipulation
- Use matplotlib/seaborn for visualization
```

---

## 9. Tracing & Observability

AgentScope sử dụng **OpenTelemetry** — tương thích với mọi OTel-compatible backend.

```python
import agentscope

# Khởi tạo với tracing
agentscope.init(
    tracing_url="http://localhost:4317",  # OTLP endpoint
    # Hoặc Langfuse:
    # tracing_url="https://cloud.langfuse.com/api/public/otel"
)

# Built-in tracing cho:
# - LLM calls
# - Tool executions
# - Agent steps
# - Formatter operations
```

**Supported platforms:**
- Alibaba Cloud CloudMonitor
- Arize-Phoenix
- Langfuse
- Bất kỳ OTel-compatible backend nào

---

## 10. Deployment

### Local Development
```bash
# Chạy trực tiếp
python my_agent.py
```

### Docker
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install agentscope
COPY . .
CMD ["python", "main.py"]
```

### Kubernetes (với OpenTelemetry)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agentscope-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: agent
        image: my-agent:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4317"
        - name: DASHSCOPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: dashscope
```

### Serverless (Cloud Run)
```bash
gcloud run deploy my-agent \
  --image gcr.io/my-project/my-agent \
  --set-env-vars DASHSCOPE_API_KEY=$DASHSCOPE_API_KEY
```

---

## 11. Structured Output

Đảm bảo agent trả về output đúng format với **self-correcting parser**:

```python
from pydantic import BaseModel
from agentscope.agent import ReActAgent

class AnalysisResult(BaseModel):
    summary: str
    key_findings: list[str]
    confidence_score: float
    recommendations: list[str]

# Agent với structured output
agent = ReActAgent(
    name="Analyst",
    sys_prompt="Analyze data and return structured results.",
    model=model,
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
    output_format=AnalysisResult,  # Type-safe output
)

result = await agent(msg)
# result là AnalysisResult object — guaranteed format
```

---

## 12. Voice Agent

```python
from agentscope.agent import VoiceAgent

voice_agent = VoiceAgent(
    name="VoiceAssistant",
    sys_prompt="You're a helpful voice assistant.",
    model=model,
    memory=InMemoryMemory(),
    tts_model=...,   # Text-to-Speech model
    stt_model=...,   # Speech-to-Text model
)

# Realtime voice interaction
await voice_agent.start_realtime_session()
```

---

## 13. Patterns & Best Practices cho Vibecoding

### Pattern 1: Single Agent với Tools
```python
# Use case: Coding assistant, research agent, data analyst
agent = ReActAgent(
    name="Assistant",
    sys_prompt="""You are an expert assistant.
    Use tools to help users effectively.
    Always explain your reasoning before using tools.""",
    model=model,
    toolkit=toolkit_with_relevant_tools,
    memory=InMemoryMemory(),
)
```

### Pattern 2: Multi-Agent Debate/Review
```python
# Use case: Code review, decision making, quality assurance
async with MsgHub([proposer, critic, judge]) as hub:
    proposal_msg = Msg("user", "Review this code: ...", "user")
    await proposer(proposal_msg)  # Đề xuất giải pháp
    await critic()                 # Phê bình
    await judge()                  # Kết luận cuối
```

### Pattern 3: Pipeline với Specialization
```python
# Use case: Content pipeline, data processing, research workflow
research_agent = ReActAgent("Researcher", "Search and gather info", ...)
analysis_agent = ReActAgent("Analyst", "Analyze gathered data", ...)
writer_agent = ReActAgent("Writer", "Write final report", ...)

async with MsgHub([research_agent, analysis_agent, writer_agent]):
    await sequential_pipeline([research_agent, analysis_agent, writer_agent])
```

### Pattern 4: MCP-Powered Agent
```python
# Use case: Agent kết nối external services (DB, APIs, file systems)
client = HttpStatelessClient("external_tools", "http://mcp-server:8000")
toolkit = Toolkit()
await toolkit.register_mcp_client(client)

agent = ReActAgent(
    "PowerAgent",
    "You can access external systems via tools.",
    model=model,
    toolkit=toolkit,
    memory=InMemoryMemory(),
)
```

### Tips quan trọng cho AI Vibecoding
1. **Luôn dùng `async/await`** — AgentScope hoàn toàn async
2. **`asyncio.run(main())`** để chạy từ sync context
3. **Docstring chi tiết cho tools** — AgentScope tự generate schema từ docstring
4. **`InMemoryMemory` cho dev**, `Mem0LongTermMemory` cho production
5. **Formatter phải match với model** — `DashScopeChatFormatter` cho DashScope, etc.
6. **Multi-agent dùng `DashScopeMultiAgentFormatter`** thay vì `DashScopeChatFormatter`
7. **Environment variables** cho API keys — không hardcode
8. **Dùng `MsgHub` context manager** (`async with`) để auto-cleanup

---

## 14. Environment Variables Reference

| Variable | Provider | Required |
|---|---|---|
| `DASHSCOPE_API_KEY` | Alibaba DashScope (Qwen) | Nếu dùng DashScope |
| `OPENAI_API_KEY` | OpenAI | Nếu dùng OpenAI |
| `ANTHROPIC_API_KEY` | Anthropic Claude | Nếu dùng Anthropic |
| `GEMINI_API_KEY` | Google Gemini | Nếu dùng Gemini |

---

## 15. Quick Reference — Imports

```python
# Agents
from agentscope.agent import ReActAgent, UserAgent, VoiceAgent

# Models
from agentscope.model import (
    DashScopeChatModel,
    OpenAIChatModel,
    AnthropicChatModel,
    GeminiChatModel,
    OllamaChatModel,
)

# Formatters
from agentscope.formatter import (
    DashScopeChatFormatter,
    DashScopeMultiAgentFormatter,
)

# Memory
from agentscope.memory import InMemoryMemory, Mem0LongTermMemory

# Tools
from agentscope.tool import Toolkit, execute_python_code, execute_shell_command

# Pipeline & MsgHub
from agentscope.pipeline import MsgHub, sequential_pipeline, fanout_pipeline

# Message
from agentscope.message import Msg

# MCP
from agentscope.mcp import HttpStatelessClient, HttpStatefulClient

# Init (for tracing)
import agentscope
```

---

## 16. Resources

- **GitHub:** [agentscope-ai/agentscope](https://github.com/agentscope-ai/agentscope)
- **Documentation:** [doc.agentscope.io](https://doc.agentscope.io)
- **Tutorial:** [doc.agentscope.io/tutorial](https://doc.agentscope.io/tutorial/)
- **API Docs:** [doc.agentscope.io/api](https://doc.agentscope.io/api/agentscope.html)
- **Examples:** [`/examples`](https://github.com/agentscope-ai/agentscope/tree/main/examples) trong repo
- **Paper (ArXiv):** [AgentScope 1.0 — 2508.16279](https://arxiv.org/abs/2508.16279)
- **ReMe (Long-term Memory):** [agentscope-ai/ReMe](https://github.com/agentscope-ai/ReMe)
- **AgentScope Runtime:** [agentscope-ai/agentscope-runtime](https://github.com/agentscope-ai/agentscope-runtime)

---

*Tài liệu được tạo: 2026-03-27 | Framework: AgentScope (Python) | License: Apache 2.0*
