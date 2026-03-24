# Agents

In the Agent Development Kit (ADK), an **Agent** is a self-contained execution unit designed to act autonomously to achieve specific goals. Agents can perform tasks, interact with users, utilize external tools, and coordinate with other agents.

The foundation for all agents in ADK is the `BaseAgent` class. It serves as the fundamental blueprint. To create functional agents, you typically extend `BaseAgent` in one of three main ways, catering to different needs – from intelligent reasoning to structured process control.

<img src="../assets/agent-types.png" alt="Types of agents in ADK">

## Core Agent Categories

ADK provides distinct agent categories to build sophisticated applications:

1. [**LLM Agents (`LlmAgent`, `Agent`)**](llm-agents.md): These agents utilize Large Language Models (LLMs) as their core engine to understand natural language, reason, plan, generate responses, and dynamically decide how to proceed or which tools to use, making them ideal for flexible, language-centric tasks. [Learn more about LLM Agents...](llm-agents.md)

2. [**Workflow Agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`)**](workflow-agents/index.md): These specialized agents control the execution flow of other agents in predefined, deterministic patterns (sequence, parallel, or loop) without using an LLM for the flow control itself, perfect for structured processes needing predictable execution. [Explore Workflow Agents...](workflow-agents/index.md)

3. [**Custom Agents**](custom-agents.md): Created by extending `BaseAgent` directly, these agents allow you to implement unique operational logic, specific control flows, or specialized integrations not covered by the standard types, catering to highly tailored application requirements. [Discover how to build Custom Agents...](custom-agents.md)

## Choosing the Right Agent Type

The following table provides a high-level comparison to help distinguish between the agent types. As you explore each type in more detail in the subsequent sections, these distinctions will become clearer.

| Feature              | LLM Agent (`LlmAgent`)              | Workflow Agent                              | Custom Agent (`BaseAgent` subclass)      |
| :------------------- | :---------------------------------- | :------------------------------------------ |:-----------------------------------------|
| **Primary Function** | Reasoning, Generation, Tool Use     | Controlling Agent Execution Flow            | Implementing Unique Logic/Integrations   |
| **Core Engine**  | Large Language Model (LLM)          | Predefined Logic (Sequence, Parallel, Loop) | Custom Code                              |
| **Determinism**  | Non-deterministic (Flexible)        | Deterministic (Predictable)                 | Can be either, based on implementation   |
| **Primary Use**  | Language tasks, Dynamic decisions   | Structured processes, Orchestration         | Tailored requirements, Specific workflows|

## Agents Working Together: Multi-Agent Systems

While each agent type serves a distinct purpose, the true power often comes from combining them. Complex applications frequently employ [multi-agent architectures](multi-agents.md) where:

* **LLM Agents** handle intelligent, language-based task execution.
* **Workflow Agents** manage the overall process flow using standard patterns.
* **Custom Agents** provide specialized capabilities or rules needed for unique integrations.

Understanding these core types is the first step toward building sophisticated, capable AI applications with ADK.

---

## What's Next?

Now that you have an overview of the different agent types available in ADK, dive deeper into how they work and how to use them effectively:

* [**LLM Agents:**](llm-agents.md) Explore how to configure agents powered by large language models, including setting instructions, providing tools, and enabling advanced features like planning and code execution.
* [**Workflow Agents:**](workflow-agents/index.md) Learn how to orchestrate tasks using `SequentialAgent`, `ParallelAgent`, and `LoopAgent` for structured and predictable processes.
* [**Custom Agents:**](custom-agents.md) Discover the principles of extending `BaseAgent` to build agents with unique logic and integrations tailored to your specific needs.
* [**Multi-Agents:**](multi-agents.md) Understand how to combine different agent types to create sophisticated, collaborative systems capable of tackling complex problems.
* [**Models:**](models.md) Learn about the different LLM integrations available and how to select the right model for your agents.


# LLM Agent

The `LlmAgent` (often aliased simply as `Agent`) is a core component in ADK,
acting as the "thinking" part of your application. It leverages the power of a
Large Language Model (LLM) for reasoning, understanding natural language, making
decisions, generating responses, and interacting with tools.

Unlike deterministic [Workflow Agents](workflow-agents/index.md) that follow
predefined execution paths, `LlmAgent` behavior is non-deterministic. It uses
the LLM to interpret instructions and context, deciding dynamically how to
proceed, which tools to use (if any), or whether to transfer control to another
agent.

Building an effective `LlmAgent` involves defining its identity, clearly guiding
its behavior through instructions, and equipping it with the necessary tools and
capabilities.

## Defining the Agent's Identity and Purpose

First, you need to establish what the agent *is* and what it's *for*.

* **`name` (Required):** Every agent needs a unique string identifier. This
  `name` is crucial for internal operations, especially in multi-agent systems
  where agents need to refer to or delegate tasks to each other. Choose a
  descriptive name that reflects the agent's function (e.g.,
  `customer_support_router`, `billing_inquiry_agent`). Avoid reserved names like
  `user`.

* **`description` (Optional, Recommended for Multi-Agent):** Provide a concise
  summary of the agent's capabilities. This description is primarily used by
  *other* LLM agents to determine if they should route a task to this agent.
  Make it specific enough to differentiate it from peers (e.g., "Handles
  inquiries about current billing statements," not just "Billing agent").

* **`model` (Required):** Specify the underlying LLM that will power this
  agent's reasoning. This is a string identifier like `"gemini-2.5-flash"`. The
  choice of model impacts the agent's capabilities, cost, and performance. See
  the [Models](models.md) page for available options and considerations.

=== "Python"

    ```python
    # Example: Defining the basic identity
    capital_agent = LlmAgent(
        model="gemini-2.5-flash",
        name="capital_agent",
        description="Answers user questions about the capital city of a given country."
        # instruction and tools will be added next
    )
    ```

=== "Java"

    


## Guiding the Agent: Instructions (`instruction`)

The `instruction` parameter is arguably the most critical for shaping an
`LlmAgent`'s behavior. It's a string (or a function returning a string) that
tells the agent:

* Its core task or goal.
* Its personality or persona (e.g., "You are a helpful assistant," "You are a witty pirate").
* Constraints on its behavior (e.g., "Only answer questions about X," "Never reveal Y").
* How and when to use its `tools`. You should explain the purpose of each tool and the circumstances under which it should be called, supplementing any descriptions within the tool itself.
* The desired format for its output (e.g., "Respond in JSON," "Provide a bulleted list").

**Tips for Effective Instructions:**

* **Be Clear and Specific:** Avoid ambiguity. Clearly state the desired actions and outcomes.
* **Use Markdown:** Improve readability for complex instructions using headings, lists, etc.
* **Provide Examples (Few-Shot):** For complex tasks or specific output formats, include examples directly in the instruction.
* **Guide Tool Use:** Don't just list tools; explain *when* and *why* the agent should use them.

**State:**

* The instruction is a string template, you can use the `{var}` syntax to insert dynamic values into the instruction.
* `{var}` is used to insert the value of the state variable named var.
* `{artifact.var}` is used to insert the text content of the artifact named var.
* If the state variable or artifact does not exist, the agent will raise an error. If you want to ignore the error, you can append a `?` to the variable name as in `{var?}`.

=== "Python"

    ```python
    # Example: Adding instructions
    capital_agent = LlmAgent(
        model="gemini-2.5-flash",
        name="capital_agent",
        description="Answers user questions about the capital city of a given country.",
        instruction="""You are an agent that provides the capital city of a country.
    When a user asks for the capital of a country:
    1. Identify the country name from the user's query.
    2. Use the `get_capital_city` tool to find the capital.
    3. Respond clearly to the user, stating the capital city.
    Example Query: "What's the capital of {country}?"
    Example Response: "The capital of France is Paris."
    """,
        # tools will be added next
    )
    ```

=== "Java"

    

*(Note: For instructions that apply to *all* agents in a system, consider using
`global_instruction` on the root agent, detailed further in the
[Multi-Agents](multi-agents.md) section.)*

## Equipping the Agent: Tools (`tools`)

Tools give your `LlmAgent` capabilities beyond the LLM's built-in knowledge or
reasoning. They allow the agent to interact with the outside world, perform
calculations, fetch real-time data, or execute specific actions.

* **`tools` (Optional):** Provide a list of tools the agent can use. Each item in the list can be:
    * A native function or method (wrapped as a `FunctionTool`). Python ADK automatically wraps the native function into a `FuntionTool` whereas, you must explicitly wrap your Java methods using `FunctionTool.create(...)`
    * An instance of a class inheriting from `BaseTool`.
    * An instance of another agent (`AgentTool`, enabling agent-to-agent delegation - see [Multi-Agents](multi-agents.md)).

The LLM uses the function/tool names, descriptions (from docstrings or the
`description` field), and parameter schemas to decide which tool to call based
on the conversation and its instructions.

=== "Python"

    ```python
    # Define a tool function
    def get_capital_city(country: str) -> str:
      """Retrieves the capital city for a given country."""
      # Replace with actual logic (e.g., API call, database lookup)
      capitals = {"france": "Paris", "japan": "Tokyo", "canada": "Ottawa"}
      return capitals.get(country.lower(), f"Sorry, I don't know the capital of {country}.")
    
    # Add the tool to the agent
    capital_agent = LlmAgent(
        model="gemini-2.5-flash",
        name="capital_agent",
        description="Answers user questions about the capital city of a given country.",
        instruction="""You are an agent that provides the capital city of a country... (previous instruction text)""",
        tools=[get_capital_city] # Provide the function directly
    )
    ```

=== "Java"

    

Learn more about Tools in the [Tools](../tools/index.md) section.

## Advanced Configuration & Control

Beyond the core parameters, `LlmAgent` offers several options for finer control:

### Fine-Tuning LLM Generation (`generate_content_config`)

You can adjust how the underlying LLM generates responses using `generate_content_config`.

* **`generate_content_config` (Optional):** Pass an instance of `google.genai.types.GenerateContentConfig` to control parameters like `temperature` (randomness), `max_output_tokens` (response length), `top_p`, `top_k`, and safety settings.

=== "Python"

    ```python
    from google.genai import types

    agent = LlmAgent(
        # ... other params
        generate_content_config=types.GenerateContentConfig(
            temperature=0.2, # More deterministic output
            max_output_tokens=250
        )
    )
    ```

=== "Java"

    

### Structuring Data (`input_schema`, `output_schema`, `output_key`)

For scenarios requiring structured data exchange with an `LLM Agent`, the ADK provides mechanisms to define expected input and desired output formats using schema definitions.

* **`input_schema` (Optional):** Define a schema representing the expected input structure. If set, the user message content passed to this agent *must* be a JSON string conforming to this schema. Your instructions should guide the user or preceding agent accordingly.

* **`output_schema` (Optional):** Define a schema representing the desired output structure. If set, the agent's final response *must* be a JSON string conforming to this schema.
    * **Constraint:** Using `output_schema` enables controlled generation within the LLM but **disables the agent's ability to use tools or transfer control to other agents**. Your instructions must guide the LLM to produce JSON matching the schema directly.

* **`output_key` (Optional):** Provide a string key. If set, the text content of the agent's *final* response will be automatically saved to the session's state dictionary under this key. This is useful for passing results between agents or steps in a workflow.
    * In Python, this might look like: `session.state[output_key] = agent_response_text`
    * In Java: `session.state().put(outputKey, agentResponseText)`

=== "Python"

    The input and output schema is typically a `Pydantic` BaseModel.

    ```python
    from pydantic import BaseModel, Field
    
    class CapitalOutput(BaseModel):
        capital: str = Field(description="The capital of the country.")
    
    structured_capital_agent = LlmAgent(
        # ... name, model, description
        instruction="""You are a Capital Information Agent. Given a country, respond ONLY with a JSON object containing the capital. Format: {"capital": "capital_name"}""",
        output_schema=CapitalOutput, # Enforce JSON output
        output_key="found_capital"  # Store result in state['found_capital']
        # Cannot use tools=[get_capital_city] effectively here
    )
    ```

=== "Java"

     The input and output schema is a `google.genai.types.Schema` object.

    

### Managing Context (`include_contents`)

Control whether the agent receives the prior conversation history.

* **`include_contents` (Optional, Default: `'default'`):** Determines if the `contents` (history) are sent to the LLM.
    * `'default'`: The agent receives the relevant conversation history.
    * `'none'`: The agent receives no prior `contents`. It operates based solely on its current instruction and any input provided in the *current* turn (useful for stateless tasks or enforcing specific contexts).

=== "Python"

    ```python
    stateless_agent = LlmAgent(
        # ... other params
        include_contents='none'
    )
    ```

=== "Java"

    

### Planning & Code Execution

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

For more complex reasoning involving multiple steps or executing code:

* **`planner` (Optional):** Assign a `BasePlanner` instance to enable multi-step reasoning and planning before execution. (See [Multi-Agents](multi-agents.md) patterns).
* **`code_executor` (Optional):** Provide a `BaseCodeExecutor` instance to allow the agent to execute code blocks (e.g., Python) found in the LLM's response. ([See Tools/Built-in tools](../tools/built-in-tools.md)).

## Putting It Together: Example

??? "Code"
    Here's the complete basic `capital_agent`:

    === "Python"
    
        ```python
        # --- Full example code demonstrating LlmAgent with Tools vs. Output Schema ---
        import json # Needed for pretty printing dicts

        from google.adk.agents import LlmAgent
        from google.adk.runners import Runner
        from google.adk.sessions import InMemorySessionService
        from google.genai import types
        from pydantic import BaseModel, Field

        # --- 1. Define Constants ---
        APP_NAME = "agent_comparison_app"
        USER_ID = "test_user_456"
        SESSION_ID_TOOL_AGENT = "session_tool_agent_xyz"
        SESSION_ID_SCHEMA_AGENT = "session_schema_agent_xyz"
        MODEL_NAME = "gemini-2.5-flash"

        # --- 2. Define Schemas ---

        # Input schema used by both agents
        class CountryInput(BaseModel):
            country: str = Field(description="The country to get information about.")

        # Output schema ONLY for the second agent
        class CapitalInfoOutput(BaseModel):
            capital: str = Field(description="The capital city of the country.")
            # Note: Population is illustrative; the LLM will infer or estimate this
            # as it cannot use tools when output_schema is set.
            population_estimate: str = Field(description="An estimated population of the capital city.")

        # --- 3. Define the Tool (Only for the first agent) ---
        def get_capital_city(country: str) -> str:
            """Retrieves the capital city of a given country."""
            print(f"\n-- Tool Call: get_capital_city(country='{country}') --")
            country_capitals = {
                "united states": "Washington, D.C.",
                "canada": "Ottawa",
                "france": "Paris",
                "japan": "Tokyo",
            }
            result = country_capitals.get(country.lower(), f"Sorry, I couldn't find the capital for {country}.")
            print(f"-- Tool Result: '{result}' --")
            return result

        # --- 4. Configure Agents ---

        # Agent 1: Uses a tool and output_key
        capital_agent_with_tool = LlmAgent(
            model=MODEL_NAME,
            name="capital_agent_tool",
            description="Retrieves the capital city using a specific tool.",
            instruction="""You are a helpful agent that provides the capital city of a country using a tool.
        The user will provide the country name in a JSON format like {"country": "country_name"}.
        1. Extract the country name.
        2. Use the `get_capital_city` tool to find the capital.
        3. Respond clearly to the user, stating the capital city found by the tool.
        """,
            tools=[get_capital_city],
            input_schema=CountryInput,
            output_key="capital_tool_result", # Store final text response
        )

        # Agent 2: Uses output_schema (NO tools possible)
        structured_info_agent_schema = LlmAgent(
            model=MODEL_NAME,
            name="structured_info_agent_schema",
            description="Provides capital and estimated population in a specific JSON format.",
            instruction=f"""You are an agent that provides country information.
        The user will provide the country name in a JSON format like {{"country": "country_name"}}.
        Respond ONLY with a JSON object matching this exact schema:
        {json.dumps(CapitalInfoOutput.model_json_schema(), indent=2)}
        Use your knowledge to determine the capital and estimate the population. Do not use any tools.
        """,
            # *** NO tools parameter here - using output_schema prevents tool use ***
            input_schema=CountryInput,
            output_schema=CapitalInfoOutput, # Enforce JSON output structure
            output_key="structured_info_result", # Store final JSON response
        )

        # --- 5. Set up Session Management and Runners ---
        session_service = InMemorySessionService()

        # Create separate sessions for clarity, though not strictly necessary if context is managed
        session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID_TOOL_AGENT)
        session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID_SCHEMA_AGENT)

        # Create a runner for EACH agent
        capital_runner = Runner(
            agent=capital_agent_with_tool,
            app_name=APP_NAME,
            session_service=session_service
        )
        structured_runner = Runner(
            agent=structured_info_agent_schema,
            app_name=APP_NAME,
            session_service=session_service
        )

        # --- 6. Define Agent Interaction Logic ---
        async def call_agent_and_print(
            runner_instance: Runner,
            agent_instance: LlmAgent,
            session_id: str,
            query_json: str
        ):
            """Sends a query to the specified agent/runner and prints results."""
            print(f"\n>>> Calling Agent: '{agent_instance.name}' | Query: {query_json}")

            user_content = types.Content(role='user', parts=[types.Part(text=query_json)])

            final_response_content = "No final response received."
            async for event in runner_instance.run_async(user_id=USER_ID, session_id=session_id, new_message=user_content):
                # print(f"Event: {event.type}, Author: {event.author}") # Uncomment for detailed logging
                if event.is_final_response() and event.content and event.content.parts:
                    # For output_schema, the content is the JSON string itself
                    final_response_content = event.content.parts[0].text

            print(f"<<< Agent '{agent_instance.name}' Response: {final_response_content}")

            current_session = session_service.get_session(app_name=APP_NAME,
                                                          user_id=USER_ID,
                                                          session_id=session_id)
            stored_output = current_session.state.get(agent_instance.output_key)

            # Pretty print if the stored output looks like JSON (likely from output_schema)
            print(f"--- Session State ['{agent_instance.output_key}']: ", end="")
            try:
                # Attempt to parse and pretty print if it's JSON
                parsed_output = json.loads(stored_output)
                print(json.dumps(parsed_output, indent=2))
            except (json.JSONDecodeError, TypeError):
                 # Otherwise, print as string
                print(stored_output)
            print("-" * 30)


        # --- 7. Run Interactions ---
        async def main():
            print("--- Testing Agent with Tool ---")
            await call_agent_and_print(capital_runner, capital_agent_with_tool, SESSION_ID_TOOL_AGENT, '{"country": "France"}')
            await call_agent_and_print(capital_runner, capital_agent_with_tool, SESSION_ID_TOOL_AGENT, '{"country": "Canada"}')

            print("\n\n--- Testing Agent with Output Schema (No Tool Use) ---")
            await call_agent_and_print(structured_runner, structured_info_agent_schema, SESSION_ID_SCHEMA_AGENT, '{"country": "France"}')
            await call_agent_and_print(structured_runner, structured_info_agent_schema, SESSION_ID_SCHEMA_AGENT, '{"country": "Japan"}')

        if __name__ == "__main__":
            await main()

        ```
    
    === "Java"
    
        

_(This example demonstrates the core concepts. More complex agents might incorporate schemas, context control, planning, etc.)_

## Related Concepts (Deferred Topics)

While this page covers the core configuration of `LlmAgent`, several related concepts provide more advanced control and are detailed elsewhere:

* **Callbacks:** Intercepting execution points (before/after model calls, before/after tool calls) using `before_model_callback`, `after_model_callback`, etc. See [Callbacks](../callbacks/types-of-callbacks.md).
* **Multi-Agent Control:** Advanced strategies for agent interaction, including planning (`planner`), controlling agent transfer (`disallow_transfer_to_parent`, `disallow_transfer_to_peers`), and system-wide instructions (`global_instruction`). See [Multi-Agents](multi-agents.md).


# Using Different Models with ADK

!!! Note
    Java ADK currently supports Gemini and Anthropic models. More model support coming soon.

The Agent Development Kit (ADK) is designed for flexibility, allowing you to
integrate various Large Language Models (LLMs) into your agents. While the setup
for Google Gemini models is covered in the
[Setup Foundation Models](../get-started/installation.md) guide, this page
details how to leverage Gemini effectively and integrate other popular models,
including those hosted externally or running locally.

ADK primarily uses two mechanisms for model integration:

1. **Direct String / Registry:** For models tightly integrated with Google Cloud
   (like Gemini models accessed via Google AI Studio or Vertex AI) or models
   hosted on Vertex AI endpoints. You typically provide the model name or
   endpoint resource string directly to the `LlmAgent`. ADK's internal registry
   resolves this string to the appropriate backend client, often utilizing the
   `google-genai` library.
2. **Wrapper Classes:** For broader compatibility, especially with models
   outside the Google ecosystem or those requiring specific client
   configurations (like models accessed via LiteLLM). You instantiate a specific
   wrapper class (e.g., `LiteLlm`) and pass this object as the `model` parameter
   to your `LlmAgent`.

The following sections guide you through using these methods based on your needs.

## Using Google Gemini Models

This is the most direct way to use Google's flagship models within ADK.

**Integration Method:** Pass the model's identifier string directly to the
`model` parameter of `LlmAgent` (or its alias, `Agent`).

**Backend Options & Setup:**

The `google-genai` library, used internally by ADK for Gemini, can connect
through either Google AI Studio or Vertex AI.

!!!note "Model support for voice/video streaming"

    In order to use voice/video streaming in ADK, you will need to use Gemini
    models that support the Live API. You can find the **model ID(s)** that
    support the Gemini Live API in the documentation:

    - [Google AI Studio: Gemini Live API](https://ai.google.dev/gemini-api/docs/models#live-api)
    - [Vertex AI: Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api)

### Google AI Studio

* **Use Case:** Google AI Studio is the easiest way to get started with Gemini.
  All you need is the [API key](https://aistudio.google.com/app/apikey). Best
  for rapid prototyping and development.
* **Setup:** Typically requires an API key:
     * Set as an environment variable or 
     * Passed during the model initialization via the `Client` (see example below)

```shell
export GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY"
export GOOGLE_GENAI_USE_VERTEXAI=FALSE
```

* **Models:** Find all available models on the
  [Google AI for Developers site](https://ai.google.dev/gemini-api/docs/models).

### Vertex AI

* **Use Case:** Recommended for production applications, leveraging Google Cloud
  infrastructure. Gemini on Vertex AI supports enterprise-grade features,
  security, and compliance controls.
* **Setup:**
    * Authenticate using Application Default Credentials (ADC):

        ```shell
        gcloud auth application-default login
        ```

    * Configure these variables either as environment variables or by providing them directly when initializing the Model.
            
         Set your Google Cloud project and location:
    
         ```shell
         export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"
         export GOOGLE_CLOUD_LOCATION="YOUR_VERTEX_AI_LOCATION" # e.g., us-central1
         ```     
    
         Explicitly tell the library to use Vertex AI:
    
         ```shell
         export GOOGLE_GENAI_USE_VERTEXAI=TRUE
         ```

* **Models:** Find available model IDs in the
  [Vertex AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models).

**Example:**

=== "Python"

    ```python
    from google.adk.agents import LlmAgent
    
    # --- Example using a stable Gemini Flash model ---
    agent_gemini_flash = LlmAgent(
        # Use the latest stable Flash model identifier
        model="gemini-2.5-flash",
        name="gemini_flash_agent",
        instruction="You are a fast and helpful Gemini assistant.",
        # ... other agent parameters
    )
    
    # --- Example using a powerful Gemini Pro model ---
    # Note: Always check the official Gemini documentation for the latest model names,
    # including specific preview versions if needed. Preview models might have
    # different availability or quota limitations.
    agent_gemini_pro = LlmAgent(
        # Use the latest generally available Pro model identifier
        model="gemini-2.5-pro",
        name="gemini_pro_agent",
        instruction="You are a powerful and knowledgeable Gemini assistant.",
        # ... other agent parameters
    )
    ```

=== "Java"

    

## Using Anthropic models

![java_only](https://img.shields.io/badge/Supported_in-Java-orange){ title="This feature is currently available for Java. Python support for direct Anthropic API (non-Vertex) is via LiteLLM."}

You can integrate Anthropic's Claude models directly using their API key or from a Vertex AI backend into your Java ADK applications by using the ADK's `Claude` wrapper class.

For Vertex AI backend, see the [Third-Party Models on Vertex AI](#third-party-models-on-vertex-ai-eg-anthropic-claude) section.

**Prerequisites:**

1.  **Dependencies:**
    *   **Anthropic SDK Classes (Transitive):** The Java ADK's `com.google.adk.models.Claude` wrapper relies on classes from Anthropic's official Java SDK. These are typically included as **transitive dependencies**.

2.  **Anthropic API Key:**
    *   Obtain an API key from Anthropic. Securely manage this key using a secret manager.

**Integration:**

Instantiate `com.google.adk.models.Claude`, providing the desired Claude model name and an `AnthropicOkHttpClient` configured with your API key. Then, pass this `Claude` instance to your `LlmAgent`.

**Example:**





## Using Cloud & Proprietary Models via LiteLLM

![python_only](https://img.shields.io/badge/Supported_in-Python-blue)

To access a vast range of LLMs from providers like OpenAI, Anthropic (non-Vertex
AI), Cohere, and many others, ADK offers integration through the LiteLLM
library.

**Integration Method:** Instantiate the `LiteLlm` wrapper class and pass it to
the `model` parameter of `LlmAgent`.

**LiteLLM Overview:** [LiteLLM](https://docs.litellm.ai/) acts as a translation
layer, providing a standardized, OpenAI-compatible interface to over 100+ LLMs.

**Setup:**

1. **Install LiteLLM:**
        ```shell
        pip install litellm
        ```
2. **Set Provider API Keys:** Configure API keys as environment variables for
   the specific providers you intend to use.

    * *Example for OpenAI:*

        ```shell
        export OPENAI_API_KEY="YOUR_OPENAI_API_KEY"
        ```

    * *Example for Anthropic (non-Vertex AI):*

        ```shell
        export ANTHROPIC_API_KEY="YOUR_ANTHROPIC_API_KEY"
        ```

    * *Consult the
      [LiteLLM Providers Documentation](https://docs.litellm.ai/docs/providers)
      for the correct environment variable names for other providers.*

        **Example:**

        ```python
        from google.adk.agents import LlmAgent
        from google.adk.models.lite_llm import LiteLlm

        # --- Example Agent using OpenAI's GPT-4o ---
        # (Requires OPENAI_API_KEY)
        agent_openai = LlmAgent(
            model=LiteLlm(model="openai/gpt-4o"), # LiteLLM model string format
            name="openai_agent",
            instruction="You are a helpful assistant powered by GPT-4o.",
            # ... other agent parameters
        )

        # --- Example Agent using Anthropic's Claude Haiku (non-Vertex) ---
        # (Requires ANTHROPIC_API_KEY)
        agent_claude_direct = LlmAgent(
            model=LiteLlm(model="anthropic/claude-3-haiku-20240307"),
            name="claude_direct_agent",
            instruction="You are an assistant powered by Claude Haiku.",
            # ... other agent parameters
        )
        ```

!!!info "Note for Windows users"

    ### Avoiding LiteLLM UnicodeDecodeError on Windows
    When using ADK agents with LiteLlm on Windows, users might encounter the following error:
    ```
    UnicodeDecodeError: 'charmap' codec can't decode byte...
    ```
    This issue occurs because `litellm` (used by LiteLlm) reads cached files (e.g., model pricing information) using the default Windows encoding (`cp1252`) instead of UTF-8.
    Windows users can prevent this issue by setting the `PYTHONUTF8` environment variable to `1`. This forces Python to use UTF-8 globally.
    **Example (PowerShell):**
    ```powershell
    # Set for current session
    $env:PYTHONUTF8 = "1"
    # Set persistently for the user
    [System.Environment]::SetEnvironmentVariable('PYTHONUTF8', '1', [System.EnvironmentVariableTarget]::User)
    Applying this setting ensures that Python reads cached files using UTF-8, avoiding the decoding error.
    ```


## Using Open & Local Models via LiteLLM

![python_only](https://img.shields.io/badge/Supported_in-Python-blue)

For maximum control, cost savings, privacy, or offline use cases, you can run
open-source models locally or self-host them and integrate them using LiteLLM.

**Integration Method:** Instantiate the `LiteLlm` wrapper class, configured to
point to your local model server.

### Ollama Integration

[Ollama](https://ollama.com/) allows you to easily run open-source models
locally.

#### Model choice

If your agent is relying on tools, please make sure that you select a model with
tool support from [Ollama website](https://ollama.com/search?c=tools).

For reliable results, we recommend using a decent-sized model with tool support.

The tool support for the model can be checked with the following command:

```bash
ollama show mistral-small3.1
  Model
    architecture        mistral3
    parameters          24.0B
    context length      131072
    embedding length    5120
    quantization        Q4_K_M

  Capabilities
    completion
    vision
    tools
```

You are supposed to see `tools` listed under capabilities.

You can also look at the template the model is using and tweak it based on your
needs.

```bash
ollama show --modelfile llama3.2 > model_file_to_modify
```

For instance, the default template for the above model inherently suggests that
the model shall call a function all the time. This may result in an infinite
loop of function calls.

```
Given the following functions, please respond with a JSON for a function call
with its proper arguments that best answers the given prompt.

Respond in the format {"name": function name, "parameters": dictionary of
argument name and its value}. Do not use variables.
```

You can swap such prompts with a more descriptive one to prevent infinite tool
call loops.

For instance:

```
Review the user's prompt and the available functions listed below.
First, determine if calling one of these functions is the most appropriate way to respond. A function call is likely needed if the prompt asks for a specific action, requires external data lookup, or involves calculations handled by the functions. If the prompt is a general question or can be answered directly, a function call is likely NOT needed.

If you determine a function call IS required: Respond ONLY with a JSON object in the format {"name": "function_name", "parameters": {"argument_name": "value"}}. Ensure parameter values are concrete, not variables.

If you determine a function call IS NOT required: Respond directly to the user's prompt in plain text, providing the answer or information requested. Do not output any JSON.
```

Then you can create a new model with the following command:

```bash
ollama create llama3.2-modified -f model_file_to_modify
```

#### Using ollama_chat provider

Our LiteLLM wrapper can be used to create agents with Ollama models.

```py
root_agent = Agent(
    model=LiteLlm(model="ollama_chat/mistral-small3.1"),
    name="dice_agent",
    description=(
        "hello world agent that can roll a dice of 8 sides and check prime"
        " numbers."
    ),
    instruction="""
      You roll dice and answer questions about the outcome of the dice rolls.
    """,
    tools=[
        roll_die,
        check_prime,
    ],
)
```

**It is important to set the provider `ollama_chat` instead of `ollama`. Using
`ollama` will result in unexpected behaviors such as infinite tool call loops
and ignoring previous context.**

While `api_base` can be provided inside LiteLLM for generation, LiteLLM library
is calling other APIs relying on the env variable instead as of v1.65.5 after
completion. So at this time, we recommend setting the env variable
`OLLAMA_API_BASE` to point to the ollama server.

```bash
export OLLAMA_API_BASE="http://localhost:11434"
adk web
```

#### Using openai provider

Alternatively, `openai` can be used as the provider name. But this will also
require setting the `OPENAI_API_BASE=http://localhost:11434/v1` and
`OPENAI_API_KEY=anything` env variables instead of `OLLAMA_API_BASE`. **Please
note that api base now has `/v1` at the end.**

```py
root_agent = Agent(
    model=LiteLlm(model="openai/mistral-small3.1"),
    name="dice_agent",
    description=(
        "hello world agent that can roll a dice of 8 sides and check prime"
        " numbers."
    ),
    instruction="""
      You roll dice and answer questions about the outcome of the dice rolls.
    """,
    tools=[
        roll_die,
        check_prime,
    ],
)
```

```bash
export OPENAI_API_BASE=http://localhost:11434/v1
export OPENAI_API_KEY=anything
adk web
```

#### Debugging

You can see the request sent to the Ollama server by adding the following in
your agent code just after imports.

```py
import litellm
litellm._turn_on_debug()
```

Look for a line like the following:

```bash
Request Sent from LiteLLM:
curl -X POST \
http://localhost:11434/api/chat \
-d '{'model': 'mistral-small3.1', 'messages': [{'role': 'system', 'content': ...
```

### Self-Hosted Endpoint (e.g., vLLM)

![python_only](https://img.shields.io/badge/Supported_in-Python-blue)

Tools such as [vLLM](https://github.com/vllm-project/vllm) allow you to host
models efficiently and often expose an OpenAI-compatible API endpoint.

**Setup:**

1. **Deploy Model:** Deploy your chosen model using vLLM (or a similar tool).
   Note the API base URL (e.g., `https://your-vllm-endpoint.run.app/v1`).
    * *Important for ADK Tools:* When deploying, ensure the serving tool
      supports and enables OpenAI-compatible tool/function calling. For vLLM,
      this might involve flags like `--enable-auto-tool-choice` and potentially
      a specific `--tool-call-parser`, depending on the model. Refer to the vLLM
      documentation on Tool Use.
2. **Authentication:** Determine how your endpoint handles authentication (e.g.,
   API key, bearer token).

    **Integration Example:**

    ```python
    import subprocess
    from google.adk.agents import LlmAgent
    from google.adk.models.lite_llm import LiteLlm

    # --- Example Agent using a model hosted on a vLLM endpoint ---

    # Endpoint URL provided by your vLLM deployment
    api_base_url = "https://your-vllm-endpoint.run.app/v1"

    # Model name as recognized by *your* vLLM endpoint configuration
    model_name_at_endpoint = "hosted_vllm/google/gemma-3-4b-it" # Example from vllm_test.py

    # Authentication (Example: using gcloud identity token for a Cloud Run deployment)
    # Adapt this based on your endpoint's security
    try:
        gcloud_token = subprocess.check_output(
            ["gcloud", "auth", "print-identity-token", "-q"]
        ).decode().strip()
        auth_headers = {"Authorization": f"Bearer {gcloud_token}"}
    except Exception as e:
        print(f"Warning: Could not get gcloud token - {e}. Endpoint might be unsecured or require different auth.")
        auth_headers = None # Or handle error appropriately

    agent_vllm = LlmAgent(
        model=LiteLlm(
            model=model_name_at_endpoint,
            api_base=api_base_url,
            # Pass authentication headers if needed
            extra_headers=auth_headers
            # Alternatively, if endpoint uses an API key:
            # api_key="YOUR_ENDPOINT_API_KEY"
        ),
        name="vllm_agent",
        instruction="You are a helpful assistant running on a self-hosted vLLM endpoint.",
        # ... other agent parameters
    )
    ```

## Using Hosted & Tuned Models on Vertex AI

For enterprise-grade scalability, reliability, and integration with Google
Cloud's MLOps ecosystem, you can use models deployed to Vertex AI Endpoints.
This includes models from Model Garden or your own fine-tuned models.

**Integration Method:** Pass the full Vertex AI Endpoint resource string
(`projects/PROJECT_ID/locations/LOCATION/endpoints/ENDPOINT_ID`) directly to the
`model` parameter of `LlmAgent`.

**Vertex AI Setup (Consolidated):**

Ensure your environment is configured for Vertex AI:

1. **Authentication:** Use Application Default Credentials (ADC):

    ```shell
    gcloud auth application-default login
    ```

2. **Environment Variables:** Set your project and location:

    ```shell
    export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"
    export GOOGLE_CLOUD_LOCATION="YOUR_VERTEX_AI_LOCATION" # e.g., us-central1
    ```

3. **Enable Vertex Backend:** Crucially, ensure the `google-genai` library
   targets Vertex AI:

    ```shell
    export GOOGLE_GENAI_USE_VERTEXAI=TRUE
    ```

### Model Garden Deployments

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

You can deploy various open and proprietary models from the
[Vertex AI Model Garden](https://console.cloud.google.com/vertex-ai/model-garden)
to an endpoint.

**Example:**

```python
from google.adk.agents import LlmAgent
from google.genai import types # For config objects

# --- Example Agent using a Llama 3 model deployed from Model Garden ---

# Replace with your actual Vertex AI Endpoint resource name
llama3_endpoint = "projects/YOUR_PROJECT_ID/locations/us-central1/endpoints/YOUR_LLAMA3_ENDPOINT_ID"

agent_llama3_vertex = LlmAgent(
    model=llama3_endpoint,
    name="llama3_vertex_agent",
    instruction="You are a helpful assistant based on Llama 3, hosted on Vertex AI.",
    generate_content_config=types.GenerateContentConfig(max_output_tokens=2048),
    # ... other agent parameters
)
```

### Fine-tuned Model Endpoints

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

Deploying your fine-tuned models (whether based on Gemini or other architectures
supported by Vertex AI) results in an endpoint that can be used directly.

**Example:**

```python
from google.adk.agents import LlmAgent

# --- Example Agent using a fine-tuned Gemini model endpoint ---

# Replace with your fine-tuned model's endpoint resource name
finetuned_gemini_endpoint = "projects/YOUR_PROJECT_ID/locations/us-central1/endpoints/YOUR_FINETUNED_ENDPOINT_ID"

agent_finetuned_gemini = LlmAgent(
    model=finetuned_gemini_endpoint,
    name="finetuned_gemini_agent",
    instruction="You are a specialized assistant trained on specific data.",
    # ... other agent parameters
)
```

### Third-Party Models on Vertex AI (e.g., Anthropic Claude)

Some providers, like Anthropic, make their models available directly through
Vertex AI.

=== "Python"

    **Integration Method:** Uses the direct model string (e.g.,
    `"claude-3-sonnet@20240229"`), *but requires manual registration* within ADK.
    
    **Why Registration?** ADK's registry automatically recognizes `gemini-*` strings
    and standard Vertex AI endpoint strings (`projects/.../endpoints/...`) and
    routes them via the `google-genai` library. For other model types used directly
    via Vertex AI (like Claude), you must explicitly tell the ADK registry which
    specific wrapper class (`Claude` in this case) knows how to handle that model
    identifier string with the Vertex AI backend.
    
    **Setup:**
    
    1. **Vertex AI Environment:** Ensure the consolidated Vertex AI setup (ADC, Env
       Vars, `GOOGLE_GENAI_USE_VERTEXAI=TRUE`) is complete.
    
    2. **Install Provider Library:** Install the necessary client library configured
       for Vertex AI.
    
        ```shell
        pip install "anthropic[vertex]"
        ```
    
    3. **Register Model Class:** Add this code near the start of your application,
       *before* creating an agent using the Claude model string:
    
        ```python
        # Required for using Claude model strings directly via Vertex AI with LlmAgent
        from google.adk.models.anthropic_llm import Claude
        from google.adk.models.registry import LLMRegistry
    
        LLMRegistry.register(Claude)
        ```
    
       **Example:**

       ```python
       from google.adk.agents import LlmAgent
       from google.adk.models.anthropic_llm import Claude # Import needed for registration
       from google.adk.models.registry import LLMRegistry # Import needed for registration
       from google.genai import types
        
       # --- Register Claude class (do this once at startup) ---
       LLMRegistry.register(Claude)
        
       # --- Example Agent using Claude 3 Sonnet on Vertex AI ---
        
       # Standard model name for Claude 3 Sonnet on Vertex AI
       claude_model_vertexai = "claude-3-sonnet@20240229"
        
       agent_claude_vertexai = LlmAgent(
           model=claude_model_vertexai, # Pass the direct string after registration
           name="claude_vertexai_agent",
           instruction="You are an assistant powered by Claude 3 Sonnet on Vertex AI.",
           generate_content_config=types.GenerateContentConfig(max_output_tokens=4096),
           # ... other agent parameters
       )
       ```

=== "Java"

    **Integration Method:** Directly instantiate the provider-specific model class (e.g., `com.google.adk.models.Claude`) and configure it with a Vertex AI backend.
    
    **Why Direct Instantiation?** The Java ADK's `LlmRegistry` primarily handles Gemini models by default. For third-party models like Claude on Vertex AI, you directly provide an instance of the ADK's wrapper class (e.g., `Claude`) to the `LlmAgent`. This wrapper class is responsible for interacting with the model via its specific client library, configured for Vertex AI.
    
    **Setup:**
    
    1.  **Vertex AI Environment:**
        *   Ensure your Google Cloud project and region are correctly set up.
        *   **Application Default Credentials (ADC):** Make sure ADC is configured correctly in your environment. This is typically done by running `gcloud auth application-default login`. The Java client libraries will use these credentials to authenticate with Vertex AI. Follow the [Google Cloud Java documentation on ADC](https://cloud.google.com/java/docs/reference/google-auth-library/latest/com.google.auth.oauth2.GoogleCredentials#com_google_auth_oauth2_GoogleCredentials_getApplicationDefault__) for detailed setup.
    
    2.  **Provider Library Dependencies:**
        *   **Third-Party Client Libraries (Often Transitive):** The ADK core library often includes the necessary client libraries for common third-party models on Vertex AI (like Anthropic's required classes) as **transitive dependencies**. This means you might not need to explicitly add a separate dependency for the Anthropic Vertex SDK in your `pom.xml` or `build.gradle`.

    3.  **Instantiate and Configure the Model:**
        When creating your `LlmAgent`, instantiate the `Claude` class (or the equivalent for another provider) and configure its `VertexBackend`.
    
    **Example:**

    

# Multi-Agent Systems in ADK

As agentic applications grow in complexity, structuring them as a single, monolithic agent can become challenging to develop, maintain, and reason about. The Agent Development Kit (ADK) supports building sophisticated applications by composing multiple, distinct `BaseAgent` instances into a **Multi-Agent System (MAS)**.

In ADK, a multi-agent system is an application where different agents, often forming a hierarchy, collaborate or coordinate to achieve a larger goal. Structuring your application this way offers significant advantages, including enhanced modularity, specialization, reusability, maintainability, and the ability to define structured control flows using dedicated workflow agents.

You can compose various types of agents derived from `BaseAgent` to build these systems:

* **LLM Agents:** Agents powered by large language models. (See [LLM Agents](llm-agents.md))
* **Workflow Agents:** Specialized agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`) designed to manage the execution flow of their sub-agents. (See [Workflow Agents](workflow-agents/index.md))
* **Custom agents:** Your own agents inheriting from `BaseAgent` with specialized, non-LLM logic. (See [Custom Agents](custom-agents.md))

The following sections detail the core ADK primitives—such as agent hierarchy, workflow agents, and interaction mechanisms—that enable you to construct and manage these multi-agent systems effectively.

## 1. ADK Primitives for Agent Composition

ADK provides core building blocks—primitives—that enable you to structure and manage interactions within your multi-agent system.

!!! Note
    The specific parameters or method names for the primitives may vary slightly by SDK language (e.g., `sub_agents` in Python, `subAgents` in Java). Refer to the language-specific API documentation for details.

### 1.1. Agent Hierarchy (Parent agent, Sub Agents)

The foundation for structuring multi-agent systems is the parent-child relationship defined in `BaseAgent`.

* **Establishing Hierarchy:** You create a tree structure by passing a list of agent instances to the `sub_agents` argument when initializing a parent agent. ADK automatically sets the `parent_agent` attribute on each child agent during initialization.
* **Single Parent Rule:** An agent instance can only be added as a sub-agent once. Attempting to assign a second parent will result in a `ValueError`.
* **Importance:** This hierarchy defines the scope for [Workflow Agents](#12-workflow-agents-as-orchestrators) and influences the potential targets for LLM-Driven Delegation. You can navigate the hierarchy using `agent.parent_agent` or find descendants using `agent.find_agent(name)`.

=== "Python"

    ```python
    # Conceptual Example: Defining Hierarchy
    from google.adk.agents import LlmAgent, BaseAgent
    
    # Define individual agents
    greeter = LlmAgent(name="Greeter", model="gemini-2.5-flash")
    task_doer = BaseAgent(name="TaskExecutor") # Custom non-LLM agent
    
    # Create parent agent and assign children via sub_agents
    coordinator = LlmAgent(
        name="Coordinator",
        model="gemini-2.5-flash",
        description="I coordinate greetings and tasks.",
        sub_agents=[ # Assign sub_agents here
            greeter,
            task_doer
        ]
    )
    
    # Framework automatically sets:
    # assert greeter.parent_agent == coordinator
    # assert task_doer.parent_agent == coordinator
    ```

=== "Java"

    

### 1.2. Workflow Agents as Orchestrators

ADK includes specialized agents derived from `BaseAgent` that don't perform tasks themselves but orchestrate the execution flow of their `sub_agents`.

* **[`SequentialAgent`](workflow-agents/sequential-agents.md):** Executes its `sub_agents` one after another in the order they are listed.
    * **Context:** Passes the *same* [`InvocationContext`](../runtime/index.md) sequentially, allowing agents to easily pass results via shared state.

=== "Python"

    ```python
    # Conceptual Example: Sequential Pipeline
    from google.adk.agents import SequentialAgent, LlmAgent

    step1 = LlmAgent(name="Step1_Fetch", output_key="data") # Saves output to state['data']
    step2 = LlmAgent(name="Step2_Process", instruction="Process data from state key 'data'.")

    pipeline = SequentialAgent(name="MyPipeline", sub_agents=[step1, step2])
    # When pipeline runs, Step2 can access the state['data'] set by Step1.
    ```

=== "Java"

    

* **[`ParallelAgent`](workflow-agents/parallel-agents.md):** Executes its `sub_agents` in parallel. Events from sub-agents may be interleaved.
    * **Context:** Modifies the `InvocationContext.branch` for each child agent (e.g., `ParentBranch.ChildName`), providing a distinct contextual path which can be useful for isolating history in some memory implementations.
    * **State:** Despite different branches, all parallel children access the *same shared* `session.state`, enabling them to read initial state and write results (use distinct keys to avoid race conditions).

=== "Python"

    ```python
    # Conceptual Example: Parallel Execution
    from google.adk.agents import ParallelAgent, LlmAgent

    fetch_weather = LlmAgent(name="WeatherFetcher", output_key="weather")
    fetch_news = LlmAgent(name="NewsFetcher", output_key="news")

    gatherer = ParallelAgent(name="InfoGatherer", sub_agents=[fetch_weather, fetch_news])
    # When gatherer runs, WeatherFetcher and NewsFetcher run concurrently.
    # A subsequent agent could read state['weather'] and state['news'].
    ```
  
=== "Java"

    

  * **[`LoopAgent`](workflow-agents/loop-agents.md):** Executes its `sub_agents` sequentially in a loop.
      * **Termination:** The loop stops if the optional `max_iterations` is reached, or if any sub-agent returns an [`Event`](../events/index.md) with `escalate=True` in it's Event Actions.
      * **Context & State:** Passes the *same* `InvocationContext` in each iteration, allowing state changes (e.g., counters, flags) to persist across loops.

=== "Python"

      ```python
      # Conceptual Example: Loop with Condition
      from google.adk.agents import LoopAgent, LlmAgent, BaseAgent
      from google.adk.events import Event, EventActions
      from google.adk.agents.invocation_context import InvocationContext
      from typing import AsyncGenerator

      class CheckCondition(BaseAgent): # Custom agent to check state
          async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
              status = ctx.session.state.get("status", "pending")
              is_done = (status == "completed")
              yield Event(author=self.name, actions=EventActions(escalate=is_done)) # Escalate if done

      process_step = LlmAgent(name="ProcessingStep") # Agent that might update state['status']

      poller = LoopAgent(
          name="StatusPoller",
          max_iterations=10,
          sub_agents=[process_step, CheckCondition(name="Checker")]
      )
      # When poller runs, it executes process_step then Checker repeatedly
      # until Checker escalates (state['status'] == 'completed') or 10 iterations pass.
      ```
    
=== "Java"

    

### 1.3. Interaction & Communication Mechanisms

Agents within a system often need to exchange data or trigger actions in one another. ADK facilitates this through:

#### a) Shared Session State (`session.state`)

The most fundamental way for agents operating within the same invocation (and thus sharing the same [`Session`](../sessions/session.md) object via the `InvocationContext`) to communicate passively.

* **Mechanism:** One agent (or its tool/callback) writes a value (`context.state['data_key'] = processed_data`), and a subsequent agent reads it (`data = context.state.get('data_key')`). State changes are tracked via [`CallbackContext`](../callbacks/index.md).
* **Convenience:** The `output_key` property on [`LlmAgent`](llm-agents.md) automatically saves the agent's final response text (or structured output) to the specified state key.
* **Nature:** Asynchronous, passive communication. Ideal for pipelines orchestrated by `SequentialAgent` or passing data across `LoopAgent` iterations.
* **See Also:** [State Management](../sessions/state.md)

=== "Python"

    ```python
    # Conceptual Example: Using output_key and reading state
    from google.adk.agents import LlmAgent, SequentialAgent
    
    agent_A = LlmAgent(name="AgentA", instruction="Find the capital of France.", output_key="capital_city")
    agent_B = LlmAgent(name="AgentB", instruction="Tell me about the city stored in state key 'capital_city'.")
    
    pipeline = SequentialAgent(name="CityInfo", sub_agents=[agent_A, agent_B])
    # AgentA runs, saves "Paris" to state['capital_city'].
    # AgentB runs, its instruction processor reads state['capital_city'] to get "Paris".
    ```

=== "Java"

    

#### b) LLM-Driven Delegation (Agent Transfer)

Leverages an [`LlmAgent`](llm-agents.md)'s understanding to dynamically route tasks to other suitable agents within the hierarchy.

* **Mechanism:** The agent's LLM generates a specific function call: `transfer_to_agent(agent_name='target_agent_name')`.
* **Handling:** The `AutoFlow`, used by default when sub-agents are present or transfer isn't disallowed, intercepts this call. It identifies the target agent using `root_agent.find_agent()` and updates the `InvocationContext` to switch execution focus.
* **Requires:** The calling `LlmAgent` needs clear `instructions` on when to transfer, and potential target agents need distinct `description`s for the LLM to make informed decisions. Transfer scope (parent, sub-agent, siblings) can be configured on the `LlmAgent`.
* **Nature:** Dynamic, flexible routing based on LLM interpretation.

=== "Python"

    ```python
    # Conceptual Setup: LLM Transfer
    from google.adk.agents import LlmAgent
    
    booking_agent = LlmAgent(name="Booker", description="Handles flight and hotel bookings.")
    info_agent = LlmAgent(name="Info", description="Provides general information and answers questions.")
    
    coordinator = LlmAgent(
        name="Coordinator",
        model="gemini-2.5-flash",
        instruction="You are an assistant. Delegate booking tasks to Booker and info requests to Info.",
        description="Main coordinator.",
        # AutoFlow is typically used implicitly here
        sub_agents=[booking_agent, info_agent]
    )
    # If coordinator receives "Book a flight", its LLM should generate:
    # FunctionCall(name='transfer_to_agent', args={'agent_name': 'Booker'})
    # ADK framework then routes execution to booking_agent.
    ```

=== "Java"

    

#### c) Explicit Invocation (`AgentTool`)

Allows an [`LlmAgent`](llm-agents.md) to treat another `BaseAgent` instance as a callable function or [Tool](../tools/index.md).

* **Mechanism:** Wrap the target agent instance in `AgentTool` and include it in the parent `LlmAgent`'s `tools` list. `AgentTool` generates a corresponding function declaration for the LLM.
* **Handling:** When the parent LLM generates a function call targeting the `AgentTool`, the framework executes `AgentTool.run_async`. This method runs the target agent, captures its final response, forwards any state/artifact changes back to the parent's context, and returns the response as the tool's result.
* **Nature:** Synchronous (within the parent's flow), explicit, controlled invocation like any other tool.
* **(Note:** `AgentTool` needs to be imported and used explicitly).

=== "Python"

    ```python
    # Conceptual Setup: Agent as a Tool
    from google.adk.agents import LlmAgent, BaseAgent
    from google.adk.tools import agent_tool
    from pydantic import BaseModel
    
    # Define a target agent (could be LlmAgent or custom BaseAgent)
    class ImageGeneratorAgent(BaseAgent): # Example custom agent
        name: str = "ImageGen"
        description: str = "Generates an image based on a prompt."
        # ... internal logic ...
        async def _run_async_impl(self, ctx): # Simplified run logic
            prompt = ctx.session.state.get("image_prompt", "default prompt")
            # ... generate image bytes ...
            image_bytes = b"..."
            yield Event(author=self.name, content=types.Content(parts=[types.Part.from_bytes(image_bytes, "image/png")]))
    
    image_agent = ImageGeneratorAgent()
    image_tool = agent_tool.AgentTool(agent=image_agent) # Wrap the agent
    
    # Parent agent uses the AgentTool
    artist_agent = LlmAgent(
        name="Artist",
        model="gemini-2.5-flash",
        instruction="Create a prompt and use the ImageGen tool to generate the image.",
        tools=[image_tool] # Include the AgentTool
    )
    # Artist LLM generates a prompt, then calls:
    # FunctionCall(name='ImageGen', args={'image_prompt': 'a cat wearing a hat'})
    # Framework calls image_tool.run_async(...), which runs ImageGeneratorAgent.
    # The resulting image Part is returned to the Artist agent as the tool result.
    ```

=== "Java"

    

These primitives provide the flexibility to design multi-agent interactions ranging from tightly coupled sequential workflows to dynamic, LLM-driven delegation networks.

## 2. Common Multi-Agent Patterns using ADK Primitives

By combining ADK's composition primitives, you can implement various established patterns for multi-agent collaboration.

### Coordinator/Dispatcher Pattern

* **Structure:** A central [`LlmAgent`](llm-agents.md) (Coordinator) manages several specialized `sub_agents`.
* **Goal:** Route incoming requests to the appropriate specialist agent.
* **ADK Primitives Used:**
    * **Hierarchy:** Coordinator has specialists listed in `sub_agents`.
    * **Interaction:** Primarily uses **LLM-Driven Delegation** (requires clear `description`s on sub-agents and appropriate `instruction` on Coordinator) or **Explicit Invocation (`AgentTool`)** (Coordinator includes `AgentTool`-wrapped specialists in its `tools`).

=== "Python"

    ```python
    # Conceptual Code: Coordinator using LLM Transfer
    from google.adk.agents import LlmAgent
    
    billing_agent = LlmAgent(name="Billing", description="Handles billing inquiries.")
    support_agent = LlmAgent(name="Support", description="Handles technical support requests.")
    
    coordinator = LlmAgent(
        name="HelpDeskCoordinator",
        model="gemini-2.5-flash",
        instruction="Route user requests: Use Billing agent for payment issues, Support agent for technical problems.",
        description="Main help desk router.",
        # allow_transfer=True is often implicit with sub_agents in AutoFlow
        sub_agents=[billing_agent, support_agent]
    )
    # User asks "My payment failed" -> Coordinator's LLM should call transfer_to_agent(agent_name='Billing')
    # User asks "I can't log in" -> Coordinator's LLM should call transfer_to_agent(agent_name='Support')
    ```

=== "Java"

    

### Sequential Pipeline Pattern

* **Structure:** A [`SequentialAgent`](workflow-agents/sequential-agents.md) contains `sub_agents` executed in a fixed order.
* **Goal:** Implement a multi-step process where the output of one step feeds into the next.
* **ADK Primitives Used:**
    * **Workflow:** `SequentialAgent` defines the order.
    * **Communication:** Primarily uses **Shared Session State**. Earlier agents write results (often via `output_key`), later agents read those results from `context.state`.

=== "Python"

    ```python
    # Conceptual Code: Sequential Data Pipeline
    from google.adk.agents import SequentialAgent, LlmAgent
    
    validator = LlmAgent(name="ValidateInput", instruction="Validate the input.", output_key="validation_status")
    processor = LlmAgent(name="ProcessData", instruction="Process data if state key 'validation_status' is 'valid'.", output_key="result")
    reporter = LlmAgent(name="ReportResult", instruction="Report the result from state key 'result'.")
    
    data_pipeline = SequentialAgent(
        name="DataPipeline",
        sub_agents=[validator, processor, reporter]
    )
    # validator runs -> saves to state['validation_status']
    # processor runs -> reads state['validation_status'], saves to state['result']
    # reporter runs -> reads state['result']
    ```

=== "Java"

    

### Parallel Fan-Out/Gather Pattern

* **Structure:** A [`ParallelAgent`](workflow-agents/parallel-agents.md) runs multiple `sub_agents` concurrently, often followed by a later agent (in a `SequentialAgent`) that aggregates results.
* **Goal:** Execute independent tasks simultaneously to reduce latency, then combine their outputs.
* **ADK Primitives Used:**
    * **Workflow:** `ParallelAgent` for concurrent execution (Fan-Out). Often nested within a `SequentialAgent` to handle the subsequent aggregation step (Gather).
    * **Communication:** Sub-agents write results to distinct keys in **Shared Session State**. The subsequent "Gather" agent reads multiple state keys.

=== "Python"

    ```python
    # Conceptual Code: Parallel Information Gathering
    from google.adk.agents import SequentialAgent, ParallelAgent, LlmAgent
    
    fetch_api1 = LlmAgent(name="API1Fetcher", instruction="Fetch data from API 1.", output_key="api1_data")
    fetch_api2 = LlmAgent(name="API2Fetcher", instruction="Fetch data from API 2.", output_key="api2_data")
    
    gather_concurrently = ParallelAgent(
        name="ConcurrentFetch",
        sub_agents=[fetch_api1, fetch_api2]
    )
    
    synthesizer = LlmAgent(
        name="Synthesizer",
        instruction="Combine results from state keys 'api1_data' and 'api2_data'."
    )
    
    overall_workflow = SequentialAgent(
        name="FetchAndSynthesize",
        sub_agents=[gather_concurrently, synthesizer] # Run parallel fetch, then synthesize
    )
    # fetch_api1 and fetch_api2 run concurrently, saving to state.
    # synthesizer runs afterwards, reading state['api1_data'] and state['api2_data'].
    ```
=== "Java"

    


### Hierarchical Task Decomposition

* **Structure:** A multi-level tree of agents where higher-level agents break down complex goals and delegate sub-tasks to lower-level agents.
* **Goal:** Solve complex problems by recursively breaking them down into simpler, executable steps.
* **ADK Primitives Used:**
    * **Hierarchy:** Multi-level `parent_agent`/`sub_agents` structure.
    * **Interaction:** Primarily **LLM-Driven Delegation** or **Explicit Invocation (`AgentTool`)** used by parent agents to assign tasks to subagents. Results are returned up the hierarchy (via tool responses or state).

=== "Python"

    ```python
    # Conceptual Code: Hierarchical Research Task
    from google.adk.agents import LlmAgent
    from google.adk.tools import agent_tool
    
    # Low-level tool-like agents
    web_searcher = LlmAgent(name="WebSearch", description="Performs web searches for facts.")
    summarizer = LlmAgent(name="Summarizer", description="Summarizes text.")
    
    # Mid-level agent combining tools
    research_assistant = LlmAgent(
        name="ResearchAssistant",
        model="gemini-2.5-flash",
        description="Finds and summarizes information on a topic.",
        tools=[agent_tool.AgentTool(agent=web_searcher), agent_tool.AgentTool(agent=summarizer)]
    )
    
    # High-level agent delegating research
    report_writer = LlmAgent(
        name="ReportWriter",
        model="gemini-2.5-flash",
        instruction="Write a report on topic X. Use the ResearchAssistant to gather information.",
        tools=[agent_tool.AgentTool(agent=research_assistant)]
        # Alternatively, could use LLM Transfer if research_assistant is a sub_agent
    )
    # User interacts with ReportWriter.
    # ReportWriter calls ResearchAssistant tool.
    # ResearchAssistant calls WebSearch and Summarizer tools.
    # Results flow back up.
    ```

=== "Java"

    

### Review/Critique Pattern (Generator-Critic)

* **Structure:** Typically involves two agents within a [`SequentialAgent`](workflow-agents/sequential-agents.md): a Generator and a Critic/Reviewer.
* **Goal:** Improve the quality or validity of generated output by having a dedicated agent review it.
* **ADK Primitives Used:**
    * **Workflow:** `SequentialAgent` ensures generation happens before review.
    * **Communication:** **Shared Session State** (Generator uses `output_key` to save output; Reviewer reads that state key). The Reviewer might save its feedback to another state key for subsequent steps.

=== "Python"

    ```python
    # Conceptual Code: Generator-Critic
    from google.adk.agents import SequentialAgent, LlmAgent
    
    generator = LlmAgent(
        name="DraftWriter",
        instruction="Write a short paragraph about subject X.",
        output_key="draft_text"
    )
    
    reviewer = LlmAgent(
        name="FactChecker",
        instruction="Review the text in state key 'draft_text' for factual accuracy. Output 'valid' or 'invalid' with reasons.",
        output_key="review_status"
    )
    
    # Optional: Further steps based on review_status
    
    review_pipeline = SequentialAgent(
        name="WriteAndReview",
        sub_agents=[generator, reviewer]
    )
    # generator runs -> saves draft to state['draft_text']
    # reviewer runs -> reads state['draft_text'], saves status to state['review_status']
    ```

=== "Java"

    

### Iterative Refinement Pattern

* **Structure:** Uses a [`LoopAgent`](workflow-agents/loop-agents.md) containing one or more agents that work on a task over multiple iterations.
* **Goal:** Progressively improve a result (e.g., code, text, plan) stored in the session state until a quality threshold is met or a maximum number of iterations is reached.
* **ADK Primitives Used:**
    * **Workflow:** `LoopAgent` manages the repetition.
    * **Communication:** **Shared Session State** is essential for agents to read the previous iteration's output and save the refined version.
    * **Termination:** The loop typically ends based on `max_iterations` or a dedicated checking agent setting `escalate=True` in the `Event Actions` when the result is satisfactory.

=== "Python"

    ```python
    # Conceptual Code: Iterative Code Refinement
    from google.adk.agents import LoopAgent, LlmAgent, BaseAgent
    from google.adk.events import Event, EventActions
    from google.adk.agents.invocation_context import InvocationContext
    from typing import AsyncGenerator
    
    # Agent to generate/refine code based on state['current_code'] and state['requirements']
    code_refiner = LlmAgent(
        name="CodeRefiner",
        instruction="Read state['current_code'] (if exists) and state['requirements']. Generate/refine Python code to meet requirements. Save to state['current_code'].",
        output_key="current_code" # Overwrites previous code in state
    )
    
    # Agent to check if the code meets quality standards
    quality_checker = LlmAgent(
        name="QualityChecker",
        instruction="Evaluate the code in state['current_code'] against state['requirements']. Output 'pass' or 'fail'.",
        output_key="quality_status"
    )
    
    # Custom agent to check the status and escalate if 'pass'
    class CheckStatusAndEscalate(BaseAgent):
        async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
            status = ctx.session.state.get("quality_status", "fail")
            should_stop = (status == "pass")
            yield Event(author=self.name, actions=EventActions(escalate=should_stop))
    
    refinement_loop = LoopAgent(
        name="CodeRefinementLoop",
        max_iterations=5,
        sub_agents=[code_refiner, quality_checker, CheckStatusAndEscalate(name="StopChecker")]
    )
    # Loop runs: Refiner -> Checker -> StopChecker
    # State['current_code'] is updated each iteration.
    # Loop stops if QualityChecker outputs 'pass' (leading to StopChecker escalating) or after 5 iterations.
    ```

=== "Java"

    

### Human-in-the-Loop Pattern

* **Structure:** Integrates human intervention points within an agent workflow.
* **Goal:** Allow for human oversight, approval, correction, or tasks that AI cannot perform.
* **ADK Primitives Used (Conceptual):**
    * **Interaction:** Can be implemented using a custom **Tool** that pauses execution and sends a request to an external system (e.g., a UI, ticketing system) waiting for human input. The tool then returns the human's response to the agent.
    * **Workflow:** Could use **LLM-Driven Delegation** (`transfer_to_agent`) targeting a conceptual "Human Agent" that triggers the external workflow, or use the custom tool within an `LlmAgent`.
    * **State/Callbacks:** State can hold task details for the human; callbacks can manage the interaction flow.
    * **Note:** ADK doesn't have a built-in "Human Agent" type, so this requires custom integration.

=== "Python"

    ```python
    # Conceptual Code: Using a Tool for Human Approval
    from google.adk.agents import LlmAgent, SequentialAgent
    from google.adk.tools import FunctionTool
    
    # --- Assume external_approval_tool exists ---
    # This tool would:
    # 1. Take details (e.g., request_id, amount, reason).
    # 2. Send these details to a human review system (e.g., via API).
    # 3. Poll or wait for the human response (approved/rejected).
    # 4. Return the human's decision.
    # async def external_approval_tool(amount: float, reason: str) -> str: ...
    approval_tool = FunctionTool(func=external_approval_tool)
    
    # Agent that prepares the request
    prepare_request = LlmAgent(
        name="PrepareApproval",
        instruction="Prepare the approval request details based on user input. Store amount and reason in state.",
        # ... likely sets state['approval_amount'] and state['approval_reason'] ...
    )
    
    # Agent that calls the human approval tool
    request_approval = LlmAgent(
        name="RequestHumanApproval",
        instruction="Use the external_approval_tool with amount from state['approval_amount'] and reason from state['approval_reason'].",
        tools=[approval_tool],
        output_key="human_decision"
    )
    
    # Agent that proceeds based on human decision
    process_decision = LlmAgent(
        name="ProcessDecision",
        instruction="Check state key 'human_decision'. If 'approved', proceed. If 'rejected', inform user."
    )
    
    approval_workflow = SequentialAgent(
        name="HumanApprovalWorkflow",
        sub_agents=[prepare_request, request_approval, process_decision]
    )
    ```

=== "Java"

    

These patterns provide starting points for structuring your multi-agent systems. You can mix and match them as needed to create the most effective architecture for your specific application.


# Workflow Agents

This section introduces "*workflow agents*" - **specialized agents that control the execution flow of its sub-agents**.  

Workflow agents are specialized components in ADK designed purely for **orchestrating the execution flow of sub-agents**. Their primary role is to manage *how* and *when* other agents run, defining the control flow of a process.

Unlike [LLM Agents](../llm-agents.md), which use Large Language Models for dynamic reasoning and decision-making, Workflow Agents operate based on **predefined logic**. They determine the execution sequence according to their type (e.g., sequential, parallel, loop) without consulting an LLM for the orchestration itself. This results in **deterministic and predictable execution patterns**.

ADK provides three core workflow agent types, each implementing a distinct execution pattern:

<div class="grid cards" markdown>

- :material-console-line: **Sequential Agents**

    ---

    Executes sub-agents one after another, in **sequence**.

    [:octicons-arrow-right-24: Learn more](sequential-agents.md)

- :material-console-line: **Loop Agents**

    ---

    **Repeatedly** executes its sub-agents until a specific termination condition is met.

    [:octicons-arrow-right-24: Learn more](loop-agents.md)

- :material-console-line: **Parallel Agents**

    ---

    Executes multiple sub-agents in **parallel**.

    [:octicons-arrow-right-24: Learn more](parallel-agents.md)

</div>

## Why Use Workflow Agents?

Workflow agents are essential when you need explicit control over how a series of tasks or agents are executed. They provide:

* **Predictability:** The flow of execution is guaranteed based on the agent type and configuration.
* **Reliability:** Ensures tasks run in the required order or pattern consistently.
* **Structure:** Allows you to build complex processes by composing agents within clear control structures.

While the workflow agent manages the control flow deterministically, the sub-agents it orchestrates can themselves be any type of agent, including intelligent LLM Agent instances. This allows you to combine structured process control with flexible, LLM-powered task execution.


# Loop agents

## The `LoopAgent`

The `LoopAgent` is a workflow agent that executes its sub-agents in a loop (i.e. iteratively). It **_repeatedly runs_ a sequence of agents** for a specified number of iterations or until a termination condition is met.

Use the `LoopAgent` when your workflow involves repetition or iterative refinement, such as like revising code.

### Example

* You want to build an agent that can generate images of food, but sometimes when you want to generate a specific number of items (e.g. 5 bananas), it generates a different number of those items in the image (e.g. an image of 7 bananas). You have two tools: `Generate Image`, `Count Food Items`. Because you want to keep generating images until it either correctly generates the specified number of items, or after a certain number of iterations, you should build your agent using a `LoopAgent`.

As with other [workflow agents](index.md), the `LoopAgent` is not powered by an LLM, and is thus deterministic in how it executes. That being said, workflow agents are only concerned only with their execution (i.e. in a loop), and not their internal logic; the tools or sub-agents of a workflow agent may or may not utilize LLMs.

### How it Works

When the `LoopAgent`'s `Run Async` method is called, it performs the following actions:

1. **Sub-Agent Execution:**  It iterates through the Sub Agents list _in order_. For _each_ sub-agent, it calls the agent's `Run Async` method.
2. **Termination Check:**

    _Crucially_, the `LoopAgent` itself does _not_ inherently decide when to stop looping. You _must_ implement a termination mechanism to prevent infinite loops.  Common strategies include:

    * **Max Iterations**: Set a maximum number of iterations in the `LoopAgent`. **The loop will terminate after that many iterations**.
    * **Escalation from sub-agent**: Design one or more sub-agents to evaluate a condition (e.g., "Is the document quality good enough?", "Has a consensus been reached?").  If the condition is met, the sub-agent can signal termination (e.g., by raising a custom event, setting a flag in a shared context, or returning a specific value).

![Loop Agent](../../assets/loop-agent.png)

### Full Example: Iterative Document Improvement

Imagine a scenario where you want to iteratively improve a document:

* **Writer Agent:** An `LlmAgent` that generates or refines a draft on a topic.
* **Critic Agent:** An `LlmAgent` that critiques the draft, identifying areas for improvement.

    ```py
    LoopAgent(sub_agents=[WriterAgent, CriticAgent], max_iterations=5)
    ```

In this setup, the `LoopAgent` would manage the iterative process.  The `CriticAgent` could be **designed to return a "STOP" signal when the document reaches a satisfactory quality level**, preventing further iterations. Alternatively, the `max iterations` parameter could be used to limit the process to a fixed number of cycles, or external logic could be implemented to make stop decisions. The **loop would run at most five times**, ensuring the iterative refinement doesn't continue indefinitely.

???+ "Full Code"

    === "Python"
        ```py
        # Part of agent.py --> Follow https://google.github.io/adk-docs/get-started/quickstart/ to learn the setup
        import asyncio
        import os
        from google.adk.agents import LoopAgent, LlmAgent, BaseAgent, SequentialAgent
        from google.genai import types
        from google.adk.runners import InMemoryRunner
        from google.adk.agents.invocation_context import InvocationContext
        from google.adk.tools.tool_context import ToolContext
        from typing import AsyncGenerator, Optional
        from google.adk.events import Event, EventActions
        # --- Constants ---
        APP_NAME = "doc_writing_app_v3" # New App Name
        USER_ID = "dev_user_01"
        SESSION_ID_BASE = "loop_exit_tool_session" # New Base Session ID
        GEMINI_MODEL = "gemini-2.5-flash"
        STATE_INITIAL_TOPIC = "initial_topic"
        # --- State Keys ---
        STATE_CURRENT_DOC = "current_document"
        STATE_CRITICISM = "criticism"
        # Define the exact phrase the Critic should use to signal completion
        COMPLETION_PHRASE = "No major issues found."
        # --- Tool Definition ---
        def exit_loop(tool_context: ToolContext):
          """Call this function ONLY when the critique indicates no further changes are needed, signaling the iterative process should end."""
          print(f"  [Tool Call] exit_loop triggered by {tool_context.agent_name}")
          tool_context.actions.escalate = True
          # Return empty dict as tools should typically return JSON-serializable output
          return {}
        # --- Agent Definitions ---
        # STEP 1: Initial Writer Agent (Runs ONCE at the beginning)
        initial_writer_agent = LlmAgent(
            name="InitialWriterAgent",
            model=GEMINI_MODEL,
            include_contents='none',
            # MODIFIED Instruction: Ask for a slightly more developed start
            instruction=f"""You are a Creative Writing Assistant tasked with starting a story.
            Write the *first draft* of a short story (aim for 2-4 sentences).
            Base the content *only* on the topic provided below. Try to introduce a specific element (like a character, a setting detail, or a starting action) to make it engaging.
            Topic: {{initial_topic}}
            Output *only* the story/document text. Do not add introductions or explanations.
        """,
            description="Writes the initial document draft based on the topic, aiming for some initial substance.",
            output_key=STATE_CURRENT_DOC
        )
        # STEP 2a: Critic Agent (Inside the Refinement Loop)
        critic_agent_in_loop = LlmAgent(
            name="CriticAgent",
            model=GEMINI_MODEL,
            include_contents='none',
            # MODIFIED Instruction: More nuanced completion criteria, look for clear improvement paths.
            instruction=f"""You are a Constructive Critic AI reviewing a short document draft (typically 2-6 sentences). Your goal is balanced feedback.
            **Document to Review:**
            ```
            {{current_document}}
            ```
            **Task:**
            Review the document for clarity, engagement, and basic coherence according to the initial topic (if known).
            IF you identify 1-2 *clear and actionable* ways the document could be improved to better capture the topic or enhance reader engagement (e.g., "Needs a stronger opening sentence", "Clarify the character's goal"):
            Provide these specific suggestions concisely. Output *only* the critique text.
            ELSE IF the document is coherent, addresses the topic adequately for its length, and has no glaring errors or obvious omissions:
            Respond *exactly* with the phrase "{COMPLETION_PHRASE}" and nothing else. It doesn't need to be perfect, just functionally complete for this stage. Avoid suggesting purely subjective stylistic preferences if the core is sound.
            Do not add explanations. Output only the critique OR the exact completion phrase.
        """,
            description="Reviews the current draft, providing critique if clear improvements are needed, otherwise signals completion.",
            output_key=STATE_CRITICISM
        )
        # STEP 2b: Refiner/Exiter Agent (Inside the Refinement Loop)
        refiner_agent_in_loop = LlmAgent(
            name="RefinerAgent",
            model=GEMINI_MODEL,
            # Relies solely on state via placeholders
            include_contents='none',
            instruction=f"""You are a Creative Writing Assistant refining a document based on feedback OR exiting the process.
            **Current Document:**
            ```
            {{current_document}}
            ```
            **Critique/Suggestions:**
            {{criticism}}
            **Task:**
            Analyze the 'Critique/Suggestions'.
            IF the critique is *exactly* "{COMPLETION_PHRASE}":
            You MUST call the 'exit_loop' function. Do not output any text.
            ELSE (the critique contains actionable feedback):
            Carefully apply the suggestions to improve the 'Current Document'. Output *only* the refined document text.
            Do not add explanations. Either output the refined document OR call the exit_loop function.
        """,
            description="Refines the document based on critique, or calls exit_loop if critique indicates completion.",
            tools=[exit_loop], # Provide the exit_loop tool
            output_key=STATE_CURRENT_DOC # Overwrites state['current_document'] with the refined version
        )
        # STEP 2: Refinement Loop Agent
        refinement_loop = LoopAgent(
            name="RefinementLoop",
            # Agent order is crucial: Critique first, then Refine/Exit
            sub_agents=[
                critic_agent_in_loop,
                refiner_agent_in_loop,
            ],
            max_iterations=5 # Limit loops
        )
        # STEP 3: Overall Sequential Pipeline
        # For ADK tools compatibility, the root agent must be named `root_agent`
        root_agent = SequentialAgent(
            name="IterativeWritingPipeline",
            sub_agents=[
                initial_writer_agent, # Run first to create initial doc
                refinement_loop       # Then run the critique/refine loop
            ],
            description="Writes an initial document and then iteratively refines it with critique using an exit tool."
        )
        ```
    === "Java"
        



# Parallel agents

The `ParallelAgent` is a [workflow agent](index.md) that executes its sub-agents *concurrently*. This dramatically speeds up workflows where tasks can be performed independently.

Use `ParallelAgent` when: For scenarios prioritizing speed and involving independent, resource-intensive tasks, a `ParallelAgent` facilitates efficient parallel execution. **When sub-agents operate without dependencies, their tasks can be performed concurrently**, significantly reducing overall processing time.

As with other [workflow agents](index.md), the `ParallelAgent` is not powered by an LLM, and is thus deterministic in how it executes. That being said, workflow agents are only concerned with their execution (i.e. executing sub-agents in parallel), and not their internal logic; the tools or sub-agents of a workflow agent may or may not utilize LLMs.

### Example

This approach is particularly beneficial for operations like multi-source data retrieval or heavy computations, where parallelization yields substantial performance gains. Importantly, this strategy assumes no inherent need for shared state or direct information exchange between the concurrently executing agents.

### How it works

When the `ParallelAgent`'s `run_async()` method is called:

1. **Concurrent Execution:** It initiates the `run_async()` method of *each* sub-agent present in the `sub_agents` list *concurrently*.  This means all the agents start running at (approximately) the same time.
2. **Independent Branches:**  Each sub-agent operates in its own execution branch.  There is ***no* automatic sharing of conversation history or state between these branches** during execution.
3. **Result Collection:** The `ParallelAgent` manages the parallel execution and, typically, provides a way to access the results from each sub-agent after they have completed (e.g., through a list of results or events). The order of results may not be deterministic.

### Independent Execution and State Management

It's *crucial* to understand that sub-agents within a `ParallelAgent` run independently.  If you *need* communication or data sharing between these agents, you must implement it explicitly.  Possible approaches include:

* **Shared `InvocationContext`:** You could pass a shared `InvocationContext` object to each sub-agent.  This object could act as a shared data store.  However, you'd need to manage concurrent access to this shared context carefully (e.g., using locks) to avoid race conditions.
* **External State Management:**  Use an external database, message queue, or other mechanism to manage shared state and facilitate communication between agents.
* **Post-Processing:** Collect results from each branch, and then implement logic to coordinate data afterwards.

![Parallel Agent](../../assets/parallel-agent.png){: width="600"}

### Full Example: Parallel Web Research

Imagine researching multiple topics simultaneously:

1. **Researcher Agent 1:**  An `LlmAgent` that researches "renewable energy sources."
2. **Researcher Agent 2:**  An `LlmAgent` that researches "electric vehicle technology."
3. **Researcher Agent 3:**  An `LlmAgent` that researches "carbon capture methods."

    ```py
    ParallelAgent(sub_agents=[ResearcherAgent1, ResearcherAgent2, ResearcherAgent3])
    ```

These research tasks are independent.  Using a `ParallelAgent` allows them to run concurrently, potentially reducing the total research time significantly compared to running them sequentially. The results from each agent would be collected separately after they finish.

???+ "Full Code"

    === "Python"
        ```py
         # Part of agent.py --> Follow https://google.github.io/adk-docs/get-started/quickstart/ to learn the setup
         # --- 1. Define Researcher Sub-Agents (to run in parallel) ---
         # Researcher 1: Renewable Energy
         researcher_agent_1 = LlmAgent(
             name="RenewableEnergyResearcher",
             model=GEMINI_MODEL,
             instruction="""You are an AI Research Assistant specializing in energy.
         Research the latest advancements in 'renewable energy sources'.
         Use the Google Search tool provided.
         Summarize your key findings concisely (1-2 sentences).
         Output *only* the summary.
         """,
             description="Researches renewable energy sources.",
             tools=[google_search],
             # Store result in state for the merger agent
             output_key="renewable_energy_result"
         )
         # Researcher 2: Electric Vehicles
         researcher_agent_2 = LlmAgent(
             name="EVResearcher",
             model=GEMINI_MODEL,
             instruction="""You are an AI Research Assistant specializing in transportation.
         Research the latest developments in 'electric vehicle technology'.
         Use the Google Search tool provided.
         Summarize your key findings concisely (1-2 sentences).
         Output *only* the summary.
         """,
             description="Researches electric vehicle technology.",
             tools=[google_search],
             # Store result in state for the merger agent
             output_key="ev_technology_result"
         )
         # Researcher 3: Carbon Capture
         researcher_agent_3 = LlmAgent(
             name="CarbonCaptureResearcher",
             model=GEMINI_MODEL,
             instruction="""You are an AI Research Assistant specializing in climate solutions.
         Research the current state of 'carbon capture methods'.
         Use the Google Search tool provided.
         Summarize your key findings concisely (1-2 sentences).
         Output *only* the summary.
         """,
             description="Researches carbon capture methods.",
             tools=[google_search],
             # Store result in state for the merger agent
             output_key="carbon_capture_result"
         )
         # --- 2. Create the ParallelAgent (Runs researchers concurrently) ---
         # This agent orchestrates the concurrent execution of the researchers.
         # It finishes once all researchers have completed and stored their results in state.
         parallel_research_agent = ParallelAgent(
             name="ParallelWebResearchAgent",
             sub_agents=[researcher_agent_1, researcher_agent_2, researcher_agent_3],
             description="Runs multiple research agents in parallel to gather information."
         )
         # --- 3. Define the Merger Agent (Runs *after* the parallel agents) ---
         # This agent takes the results stored in the session state by the parallel agents
         # and synthesizes them into a single, structured response with attributions.
         merger_agent = LlmAgent(
             name="SynthesisAgent",
             model=GEMINI_MODEL,  # Or potentially a more powerful model if needed for synthesis
             instruction="""You are an AI Assistant responsible for combining research findings into a structured report.
         Your primary task is to synthesize the following research summaries, clearly attributing findings to their source areas. Structure your response using headings for each topic. Ensure the report is coherent and integrates the key points smoothly.
         **Crucially: Your entire response MUST be grounded *exclusively* on the information provided in the 'Input Summaries' below. Do NOT add any external knowledge, facts, or details not present in these specific summaries.**
         **Input Summaries:**
         *   **Renewable Energy:**
             {renewable_energy_result}
         *   **Electric Vehicles:**
             {ev_technology_result}
         *   **Carbon Capture:**
             {carbon_capture_result}
         **Output Format:**
         ## Summary of Recent Sustainable Technology Advancements
         ### Renewable Energy Findings
         (Based on RenewableEnergyResearcher's findings)
         [Synthesize and elaborate *only* on the renewable energy input summary provided above.]
         ### Electric Vehicle Findings
         (Based on EVResearcher's findings)
         [Synthesize and elaborate *only* on the EV input summary provided above.]
         ### Carbon Capture Findings
         (Based on CarbonCaptureResearcher's findings)
         [Synthesize and elaborate *only* on the carbon capture input summary provided above.]
         ### Overall Conclusion
         [Provide a brief (1-2 sentence) concluding statement that connects *only* the findings presented above.]
         Output *only* the structured report following this format. Do not include introductory or concluding phrases outside this structure, and strictly adhere to using only the provided input summary content.
         """,
             description="Combines research findings from parallel agents into a structured, cited report, strictly grounded on provided inputs.",
             # No tools needed for merging
             # No output_key needed here, as its direct response is the final output of the sequence
         )
         # --- 4. Create the SequentialAgent (Orchestrates the overall flow) ---
         # This is the main agent that will be run. It first executes the ParallelAgent
         # to populate the state, and then executes the MergerAgent to produce the final output.
         sequential_pipeline_agent = SequentialAgent(
             name="ResearchAndSynthesisPipeline",
             # Run parallel research first, then merge
             sub_agents=[parallel_research_agent, merger_agent],
             description="Coordinates parallel research and synthesizes the results."
         )
         root_agent = sequential_pipeline_agent
        ```
    === "Java"
        


# Sequential agents

## The `SequentialAgent`

The `SequentialAgent` is a [workflow agent](index.md) that executes its sub-agents in the order they are specified in the list.

Use the `SequentialAgent` when you want the execution to occur in a fixed, strict order.

### Example

* You want to build an agent that can summarize any webpage, using two tools: `Get Page Contents` and `Summarize Page`. Because the agent must always call `Get Page Contents` before calling `Summarize Page` (you can't summarize from nothing!), you should build your agent using a `SequentialAgent`.

As with other [workflow agents](index.md), the `SequentialAgent` is not powered by an LLM, and is thus deterministic in how it executes. That being said, workflow agents are concerned only with their execution (i.e. in sequence), and not their internal logic; the tools or sub-agents of a workflow agent may or may not utilize LLMs.

### How it works

When the `SequentialAgent`'s `Run Async` method is called, it performs the following actions:

1. **Iteration:** It iterates through the sub agents list in the order they were provided.
2. **Sub-Agent Execution:** For each sub-agent in the list, it calls the sub-agent's `Run Async` method.

![Sequential Agent](../../assets/sequential-agent.png){: width="600"}

### Full Example: Code Development Pipeline

Consider a simplified code development pipeline:

* **Code Writer Agent:**  An LLM Agent that generates initial code based on a specification.
* **Code Reviewer Agent:**  An LLM Agent that reviews the generated code for errors, style issues, and adherence to best practices.  It receives the output of the Code Writer Agent.
* **Code Refactorer Agent:** An LLM Agent that takes the reviewed code (and the reviewer's comments) and refactors it to improve quality and address issues.

A `SequentialAgent` is perfect for this:

```py
SequentialAgent(sub_agents=[CodeWriterAgent, CodeReviewerAgent, CodeRefactorerAgent])
```

This ensures the code is written, *then* reviewed, and *finally* refactored, in a strict, dependable order. **The output from each sub-agent is passed to the next by storing them in state via [Output Key](../llm-agents.md#structuring-data-input_schema-output_schema-output_key)**.

???+ "Code"

    === "Python"
        ```py
        # Part of agent.py --> Follow https://google.github.io/adk-docs/get-started/quickstart/ to learn the setup
        # --- 1. Define Sub-Agents for Each Pipeline Stage ---
        # Code Writer Agent
        # Takes the initial specification (from user query) and writes code.
        code_writer_agent = LlmAgent(
            name="CodeWriterAgent",
            model=GEMINI_MODEL,
            # Change 3: Improved instruction
            instruction="""You are a Python Code Generator.
        Based *only* on the user's request, write Python code that fulfills the requirement.
        Output *only* the complete Python code block, enclosed in triple backticks (```python ... ```). 
        Do not add any other text before or after the code block.
        """,
            description="Writes initial Python code based on a specification.",
            output_key="generated_code" # Stores output in state['generated_code']
        )
        # Code Reviewer Agent
        # Takes the code generated by the previous agent (read from state) and provides feedback.
        code_reviewer_agent = LlmAgent(
            name="CodeReviewerAgent",
            model=GEMINI_MODEL,
            # Change 3: Improved instruction, correctly using state key injection
            instruction="""You are an expert Python Code Reviewer. 
            Your task is to provide constructive feedback on the provided code.
            **Code to Review:**
            ```python
            {generated_code}
            ```
        **Review Criteria:**
        1.  **Correctness:** Does the code work as intended? Are there logic errors?
        2.  **Readability:** Is the code clear and easy to understand? Follows PEP 8 style guidelines?
        3.  **Efficiency:** Is the code reasonably efficient? Any obvious performance bottlenecks?
        4.  **Edge Cases:** Does the code handle potential edge cases or invalid inputs gracefully?
        5.  **Best Practices:** Does the code follow common Python best practices?
        **Output:**
        Provide your feedback as a concise, bulleted list. Focus on the most important points for improvement.
        If the code is excellent and requires no changes, simply state: "No major issues found."
        Output *only* the review comments or the "No major issues" statement.
        """,
            description="Reviews code and provides feedback.",
            output_key="review_comments", # Stores output in state['review_comments']
        )
        # Code Refactorer Agent
        # Takes the original code and the review comments (read from state) and refactors the code.
        code_refactorer_agent = LlmAgent(
            name="CodeRefactorerAgent",
            model=GEMINI_MODEL,
            # Change 3: Improved instruction, correctly using state key injection
            instruction="""You are a Python Code Refactoring AI.
        Your goal is to improve the given Python code based on the provided review comments.
          **Original Code:**
          ```python
          {generated_code}
          ```
          **Review Comments:**
          {review_comments}
        **Task:**
        Carefully apply the suggestions from the review comments to refactor the original code.
        If the review comments state "No major issues found," return the original code unchanged.
        Ensure the final code is complete, functional, and includes necessary imports and docstrings.
        **Output:**
        Output *only* the final, refactored Python code block, enclosed in triple backticks (```python ... ```). 
        Do not add any other text before or after the code block.
        """,
            description="Refactors code based on review comments.",
            output_key="refactored_code", # Stores output in state['refactored_code']
        )
        # --- 2. Create the SequentialAgent ---
        # This agent orchestrates the pipeline by running the sub_agents in order.
        code_pipeline_agent = SequentialAgent(
            name="CodePipelineAgent",
            sub_agents=[code_writer_agent, code_reviewer_agent, code_refactorer_agent],
            description="Executes a sequence of code writing, reviewing, and refactoring.",
            # The agents will run in the order provided: Writer -> Reviewer -> Refactorer
        )
        # For ADK tools compatibility, the root agent must be named `root_agent`
        root_agent = code_pipeline_agent
        ```

    === "Java"
        

    


# API Reference

The Agent Development Kit (ADK) provides comprehensive API references for both Python and Java, allowing you to dive deep into all available classes, methods, and functionalities.
