---
name: agentscope-session-memory
description: >
  Manage memory and context for AgentScope agents — short-term and long-term.
  Use when the user asks to "add memory to agent", "persist conversation history",
  "use long-term memory", "manage agent context", "implement ReMe memory",
  "agent nhớ thông tin", or needs guidance on InMemoryMemory and Mem0LongTermMemory.
metadata:
  version: "0.1.0"
  framework: "agentscope-python"
---

# AgentScope Session & Memory

Quản lý memory cho agents — từ in-session đến persistent long-term memory.

## InMemoryMemory (Short-term)

Lưu trong RAM, reset khi process khởi động lại. Dùng cho development và stateless services.

```python
from agentscope.memory import InMemoryMemory

memory = InMemoryMemory()

# Gán vào agent
from agentscope.agent import ReActAgent
agent = ReActAgent(
    name="Assistant",
    sys_prompt="...",
    model=model,
    memory=InMemoryMemory(),  # Fresh memory mỗi session
    toolkit=toolkit,
)
```

## Mem0LongTermMemory (Long-term / Persistent)

Persistent memory giữa các sessions. Cần cài `mem0ai`:

```bash
pip install mem0ai
```

```python
from agentscope.memory import Mem0LongTermMemory

# Khởi tạo với IDs để identify agent và user
memory = Mem0LongTermMemory(
    agent_id="customer_support_agent",  # Định danh agent
    user_id="user_12345",               # Định danh user
    # config={}  # Optional: mem0 config (API key, vector DB, etc.)
)

agent = ReActAgent(
    name="SupportAgent",
    sys_prompt="You remember past conversations with this user.",
    model=model,
    memory=memory,  # Persistent across sessions
    toolkit=toolkit,
)
```

### Hai chế độ LongTermMemory

**`agent_control`** — Agent tự quyết định khi nào save/retrieve memory qua tool calls:
```python
memory = Mem0LongTermMemory(
    agent_id="agent_001",
    user_id="user_001",
    mode="agent_control",  # Agent tự quản lý
)
```

**`static_control`** — Developer kiểm soát trực tiếp:
```python
memory = Mem0LongTermMemory(
    agent_id="agent_001",
    user_id="user_001",
    mode="static_control",  # Developer kiểm soát
)

# Thêm memory thủ công
await memory.add("User prefers Python over JavaScript")
await memory.add("User is building a recommendation system")

# Retrieve relevant memories
memories = await memory.get("programming language preference")
```

## ReMe (Long-term Memory Kit)

AgentScope ReMe là advanced long-term memory với compression và database support.

```bash
pip install agentscope[reme]
# hoặc: pip install agentscope-reme
```

```python
# ReMe tích hợp qua Mem0LongTermMemory với backend config
memory = Mem0LongTermMemory(
    agent_id="agent_001",
    user_id="user_001",
    config={
        "vector_store": {
            "provider": "chroma",  # hoặc "qdrant", "pgvector"
            "config": {"collection_name": "agent_memories"}
        }
    }
)
```

## Memory trong Multi-Agent Setup

Mỗi agent có memory riêng — không share:

```python
# Mỗi agent có memory độc lập
alice = ReActAgent(
    name="Alice",
    memory=InMemoryMemory(),  # Alice's memory
    ...
)

bob = ReActAgent(
    name="Bob",
    memory=InMemoryMemory(),  # Bob's memory — riêng biệt
    ...
)

# Khi dùng MsgHub, messages được broadcast
# và mỗi agent lưu vào memory của mình qua `observe()`
async with MsgHub([alice, bob]):
    await alice()  # Bob tự động nhận và observe message của Alice
    await bob()
```

## Memory Lifecycle

```
User sends message
        ↓
Agent receives Msg
        ↓
Agent adds to memory (observe)
        ↓
Agent formats memory → context window
        ↓
Model generates response
        ↓
Agent saves response to memory
        ↓
Response sent to next agent / user
```

## Khi nào dùng gì?

| Scenario | Memory Type |
|---|---|
| Development, prototyping | `InMemoryMemory` |
| Production single session | `InMemoryMemory` |
| Multi-session (nhớ user) | `Mem0LongTermMemory` |
| Chatbot với lịch sử lâu dài | `Mem0LongTermMemory` + compression |
| Agent tự quản lý memory | `mode="agent_control"` |
| Developer kiểm soát memory | `mode="static_control"` |

## Additional Resources

- **`references/agentscope-memory-full.md`** — Full memory reference với config options
- [ReMe GitHub](https://github.com/agentscope-ai/ReMe)
