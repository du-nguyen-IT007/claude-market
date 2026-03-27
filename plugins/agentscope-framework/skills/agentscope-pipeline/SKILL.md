---
name: agentscope-pipeline
description: >
  Orchestrate multi-agent workflows using AgentScope MsgHub and pipelines.
  Use when the user asks to "create a pipeline", "orchestrate multiple agents",
  "sequential agent workflow", "parallel agents", "broadcast message to agents",
  "multi-agent debate", "agent workflow A→B→C", or needs MsgHub and pipeline patterns.
metadata:
  version: "0.1.0"
  framework: "agentscope-python"
---

# AgentScope Pipeline & MsgHub

Orchestrate multi-agent workflows với MsgHub broadcast system và pipeline patterns.

## MsgHub — Core Concept

MsgHub là message broadcast hub. Khi một agent reply, tất cả participants khác **tự động nhận message** qua `observe()`. Developer không cần manually route messages.

```python
from agentscope.pipeline import MsgHub
from agentscope.message import Msg

# Tạo agents (xem agentscope-agent-architect)
# ...

async with MsgHub(
    participants=[agent_a, agent_b, agent_c],
    announcement=Msg(  # Optional: message gửi cho tất cả lúc bắt đầu
        name="system",
        content="You are participating in a technical review.",
        role="system",
    ),
):
    await agent_a()  # Reply tự động đến agent_b và agent_c
    await agent_b()  # Đã nhận message của agent_a
    await agent_c()  # Đã nhận messages của agent_a và agent_b
```

## Sequential Pipeline

Thực thi agents theo thứ tự, output của agent trước là input của agent sau.

```python
from agentscope.pipeline import MsgHub, sequential_pipeline

async with MsgHub([researcher, analyst, writer]) as hub:
    # Chạy tuần tự: researcher → analyst → writer
    await sequential_pipeline([researcher, analyst, writer])
```

**Use cases:** Research → Analysis → Report, Code → Review → Fix

## Fanout Pipeline (Parallel)

Nhiều agents xử lý song song từ cùng một input.

```python
from agentscope.pipeline import fanout_pipeline
from agentscope.message import Msg

initial_msg = Msg("user", "Analyze this dataset: ...", "user")

# Ba agents xử lý song song
results = await fanout_pipeline(
    agents=[statistical_analyst, visual_analyst, domain_expert],
    msg=initial_msg,
)
# results = list of Msg từ mỗi agent
```

**Use cases:** Parallel research, multi-perspective analysis, A/B evaluation

## Kết hợp Sequential + Parallel

```python
# Pattern: Parallel research → Sequential analysis → Final report

# Phase 1: Thu thập data song song
initial = Msg("user", "Research topic: AI in healthcare", "user")
research_results = await fanout_pipeline(
    agents=[researcher_a, researcher_b, researcher_c],
    msg=initial,
)

# Aggregate results thành một message
combined = Msg(
    "system",
    "\n".join([r.get_text_content() for r in research_results]),
    "system",
)

# Phase 2: Sequential analysis và writing
async with MsgHub([analyst, writer]) as hub:
    await sequential_pipeline([analyst, writer], initial_msg=combined)
```

## Multi-Agent Debate Pattern

```python
async def run_debate(topic: str, rounds: int = 3):
    announcement = Msg(
        "system",
        f"Debate topic: {topic}. You are academic debaters. Be concise.",
        "system",
    )

    async with MsgHub(
        [proposer, opponent, moderator],
        announcement=announcement,
    ):
        for round_num in range(rounds):
            print(f"\n--- Round {round_num + 1} ---")
            await proposer()   # Đề xuất lập luận
            await opponent()   # Phản biện
        await moderator()      # Kết luận cuối
```

## Dynamic Hub với Điều kiện

```python
async with MsgHub(participants=[agent_a]) as hub:
    result = await agent_a(initial_msg)

    # Quyết định có cần review không
    if "complex" in result.get_text_content():
        hub.add(reviewer)
        await reviewer()

    # Broadcast thông báo đến tất cả
    await hub.broadcast(
        Msg("system", "Task complete. Provide summary.", "system")
    )
    await agent_a()
```

## Formatter Selection

| Scenario | Formatter |
|---|---|
| Single agent chat | `DashScopeChatFormatter` |
| Multi-agent (MsgHub) | `DashScopeMultiAgentFormatter` |
| OpenAI single | `OpenAIChatFormatter` |
| OpenAI multi-agent | `OpenAIMultiAgentFormatter` |

```python
from agentscope.formatter import (
    DashScopeChatFormatter,
    DashScopeMultiAgentFormatter,
    OpenAIChatFormatter,
    OpenAIMultiAgentFormatter,
)

# CRITICAL: Dùng đúng formatter cho multi-agent
formatter = DashScopeMultiAgentFormatter()  # ← cho MsgHub
# KHÔNG dùng DashScopeChatFormatter cho multi-agent
```

## Common Pipeline Patterns

```python
# Pattern 1: Code Review Pipeline
async with MsgHub([coder, reviewer, tester]):
    await sequential_pipeline([coder, reviewer, tester])

# Pattern 2: Content Creation
async with MsgHub([researcher, writer, editor]):
    await sequential_pipeline([researcher, writer, editor])

# Pattern 3: Customer Support Triage
async with MsgHub([classifier, specialist_a, specialist_b]) as hub:
    classification = await classifier(user_msg)
    if "billing" in classification.get_text_content():
        hub.delete(specialist_b)
        await specialist_a()
    else:
        hub.delete(specialist_a)
        await specialist_b()
```

## Additional Resources

- **`references/agentscope-pipeline-full.md`** — Full pipeline reference với advanced patterns
