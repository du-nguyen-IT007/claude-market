# Built-in tools

These built-in tools provide ready-to-use functionality such as Google Search or
code executors that provide agents with common capabilities. For instance, an
agent that needs to retrieve information from the web can directly use the
**google\_search** tool without any additional setup.

## How to Use

1. **Import:** Import the desired tool from the tools module. This is `agents.tools` in Python or `com.google.adk.tools` in Java.
2. **Configure:** Initialize the tool, providing required parameters if any.
3. **Register:** Add the initialized tool to the **tools** list of your Agent.

Once added to an agent, the agent can decide to use the tool based on the **user
prompt** and its **instructions**. The framework handles the execution of the
tool when the agent calls it. Important: check the ***Limitations*** section of this page.

## Available Built-in tools

Note: Java only supports Google Search and Code Execution tools currently.

### Google Search

The `google_search` tool allows the agent to perform web searches using Google
Search. The `google_search` tool is only compatible with Gemini 2 models.

!!! warning "Additional requirements when using the `google_search` tool"
    When you use grounding with Google Search, and you receive Search suggestions in your response, you must display the Search suggestions in production and in your applications.
    For more information on grounding with Google Search, see Grounding with Google Search documentation for [Google AI Studio](https://ai.google.dev/gemini-api/docs/grounding/search-suggestions) or [Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-search-suggestions). The UI code (HTML) is returned in the Gemini response as `renderedContent`, and you will need to show the HTML in your app, in accordance with the policy.

=== "Python"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    from google.adk.agents import Agent
    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from google.adk.tools import google_search
    from google.genai import types

    APP_NAME="google_search_agent"
    USER_ID="user1234"
    SESSION_ID="1234"


    root_agent = Agent(
        name="basic_search_agent",
        model="gemini-2.5-flash",
        description="Agent to answer questions using Google Search.",
        instruction="I can answer your questions by searching the internet. Just ask me anything!",
        # google_search is a pre-built tool which allows the agent to perform Google searches.
        tools=[google_search]
    )

    # Session and Runner
    async def setup_session_and_runner():
        session_service = InMemorySessionService()
        session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
        runner = Runner(agent=root_agent, app_name=APP_NAME, session_service=session_service)
        return session, runner

    # Agent Interaction
    async def call_agent_async(query):
        content = types.Content(role='user', parts=[types.Part(text=query)])
        session, runner = await setup_session_and_runner()
        events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

        async for event in events:
            if event.is_final_response():
                final_response = event.content.parts[0].text
                print("Agent Response: ", final_response)

    # Note: In Colab, you can directly use 'await' at the top level.
    # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
    await call_agent_async("what's the latest ai news?")

    ```

=== "Java"

    

### Code Execution

The `built_in_code_execution` tool enables the agent to execute code,
specifically when using Gemini 2 models. This allows the model to perform tasks
like calculations, data manipulation, or running small scripts.

=== "Python"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    import asyncio
    from google.adk.agents import LlmAgent
    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from google.adk.code_executors import BuiltInCodeExecutor
    from google.genai import types

    AGENT_NAME = "calculator_agent"
    APP_NAME = "calculator"
    USER_ID = "user1234"
    SESSION_ID = "session_code_exec_async"
    GEMINI_MODEL = "gemini-2.5-flash"

    # Agent Definition
    code_agent = LlmAgent(
        name=AGENT_NAME,
        model=GEMINI_MODEL,
        executor=[BuiltInCodeExecutor],
        instruction="""You are a calculator agent.
        When given a mathematical expression, write and execute Python code to calculate the result.
        Return only the final numerical result as plain text, without markdown or code blocks.
        """,
        description="Executes Python code to perform calculations.",
    )

    # Session and Runner
    session_service = InMemorySessionService()
    session = session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
    )
    runner = Runner(agent=code_agent, app_name=APP_NAME, session_service=session_service)


    # Agent Interaction (Async)
    async def call_agent_async(query):
        content = types.Content(role="user", parts=[types.Part(text=query)])
        print(f"\n--- Running Query: {query} ---")
        final_response_text = "No final text response captured."
        try:
            # Use run_async
            async for event in runner.run_async(
                user_id=USER_ID, session_id=SESSION_ID, new_message=content
            ):
                print(f"Event ID: {event.id}, Author: {event.author}")

                # --- Check for specific parts FIRST ---
                has_specific_part = False
                if event.content and event.content.parts:
                    for part in event.content.parts:  # Iterate through all parts
                        if part.executable_code:
                            # Access the actual code string via .code
                            print(
                                f"  Debug: Agent generated code:\n```python\n{part.executable_code.code}\n```"
                            )
                            has_specific_part = True
                        elif part.code_execution_result:
                            # Access outcome and output correctly
                            print(
                                f"  Debug: Code Execution Result: {part.code_execution_result.outcome} - Output:\n{part.code_execution_result.output}"
                            )
                            has_specific_part = True
                        # Also print any text parts found in any event for debugging
                        elif part.text and not part.text.isspace():
                            print(f"  Text: '{part.text.strip()}'")
                            # Do not set has_specific_part=True here, as we want the final response logic below

                # --- Check for final response AFTER specific parts ---
                # Only consider it final if it doesn't have the specific code parts we just handled
                if not has_specific_part and event.is_final_response():
                    if (
                        event.content
                        and event.content.parts
                        and event.content.parts[0].text
                    ):
                        final_response_text = event.content.parts[0].text.strip()
                        print(f"==> Final Agent Response: {final_response_text}")
                    else:
                        print("==> Final Agent Response: [No text content in final event]")

        except Exception as e:
            print(f"ERROR during agent run: {e}")
        print("-" * 30)


    # Main async function to run the examples
    async def main():
        await call_agent_async("Calculate the value of (5 + 7) * 3")
        await call_agent_async("What is 10 factorial?")


    # Execute the main async function
    try:
        asyncio.run(main())
    except RuntimeError as e:
        # Handle specific error when running asyncio.run in an already running loop (like Jupyter/Colab)
        if "cannot be called from a running event loop" in str(e):
            print("\nRunning in an existing event loop (like Colab/Jupyter).")
            print("Please run `await main()` in a notebook cell instead.")
            # If in an interactive environment like a notebook, you might need to run:
            # await main()
        else:
            raise e  # Re-raise other runtime errors

    ```

=== "Java"

    


### Vertex AI Search

The `vertex_ai_search_tool` uses Google Cloud's Vertex AI Search, enabling the
agent to search across your private, configured data stores (e.g., internal
documents, company policies, knowledge bases). This built-in tool requires you
to provide the specific data store ID during configuration.



```py
import asyncio

from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from google.adk.tools import VertexAiSearchTool

# Replace with your actual Vertex AI Search Datastore ID
# Format: projects/<PROJECT_ID>/locations/<LOCATION>/collections/default_collection/dataStores/<DATASTORE_ID>
# e.g., "projects/12345/locations/us-central1/collections/default_collection/dataStores/my-datastore-123"
YOUR_DATASTORE_ID = "YOUR_DATASTORE_ID_HERE"

# Constants
APP_NAME_VSEARCH = "vertex_search_app"
USER_ID_VSEARCH = "user_vsearch_1"
SESSION_ID_VSEARCH = "session_vsearch_1"
AGENT_NAME_VSEARCH = "doc_qa_agent"
GEMINI_2_FLASH = "gemini-2.5-flash"

# Tool Instantiation
# You MUST provide your datastore ID here.
vertex_search_tool = VertexAiSearchTool(data_store_id=YOUR_DATASTORE_ID)

# Agent Definition
doc_qa_agent = LlmAgent(
    name=AGENT_NAME_VSEARCH,
    model=GEMINI_2_FLASH, # Requires Gemini model
    tools=[vertex_search_tool],
    instruction=f"""You are a helpful assistant that answers questions based on information found in the document store: {YOUR_DATASTORE_ID}.
    Use the search tool to find relevant information before answering.
    If the answer isn't in the documents, say that you couldn't find the information.
    """,
    description="Answers questions using a specific Vertex AI Search datastore.",
)

# Session and Runner Setup
session_service_vsearch = InMemorySessionService()
runner_vsearch = Runner(
    agent=doc_qa_agent, app_name=APP_NAME_VSEARCH, session_service=session_service_vsearch
)
session_vsearch = session_service_vsearch.create_session(
    app_name=APP_NAME_VSEARCH, user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH
)

# Agent Interaction Function
async def call_vsearch_agent_async(query):
    print("\n--- Running Vertex AI Search Agent ---")
    print(f"Query: {query}")
    if "YOUR_DATASTORE_ID_HERE" in YOUR_DATASTORE_ID:
        print("Skipping execution: Please replace YOUR_DATASTORE_ID_HERE with your actual datastore ID.")
        print("-" * 30)
        return

    content = types.Content(role='user', parts=[types.Part(text=query)])
    final_response_text = "No response received."
    try:
        async for event in runner_vsearch.run_async(
            user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH, new_message=content
        ):
            # Like Google Search, results are often embedded in the model's response.
            if event.is_final_response() and event.content and event.content.parts:
                final_response_text = event.content.parts[0].text.strip()
                print(f"Agent Response: {final_response_text}")
                # You can inspect event.grounding_metadata for source citations
                if event.grounding_metadata:
                    print(f"  (Grounding metadata found with {len(event.grounding_metadata.grounding_attributions)} attributions)")

    except Exception as e:
        print(f"An error occurred: {e}")
        print("Ensure your datastore ID is correct and the service account has permissions.")
    print("-" * 30)

# --- Run Example ---
async def run_vsearch_example():
    # Replace with a question relevant to YOUR datastore content
    await call_vsearch_agent_async("Summarize the main points about the Q2 strategy document.")
    await call_vsearch_agent_async("What safety procedures are mentioned for lab X?")

# Execute the example
# await run_vsearch_example()

# Running locally due to potential colab asyncio issues with multiple awaits
try:
    asyncio.run(run_vsearch_example())
except RuntimeError as e:
    if "cannot be called from a running event loop" in str(e):
        print("Skipping execution in running event loop (like Colab/Jupyter). Run locally.")
    else:
        raise e

```


### BigQuery

These are a set of tools aimed to provide integration with BigQuery, namely:

* **`list_dataset_ids`**: Fetches BigQuery dataset ids present in a GCP project.
* **`get_dataset_info`**: Fetches metadata about a BigQuery dataset.
* **`list_table_ids`**: Fetches table ids present in a BigQuery dataset.
* **`get_table_info`**: Fetches metadata about a BigQuery table.
* **`execute_sql`**: Runs a SQL query in BigQuery and fetch the result.

They are packaged in the toolset `BigQueryToolset`.



```py
# Copyright 2026 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.bigquery import BigQueryCredentialsConfig
from google.adk.tools.bigquery import BigQueryToolset
from google.adk.tools.bigquery.config import BigQueryToolConfig
from google.adk.tools.bigquery.config import WriteMode
from google.genai import types
import google.auth

# Define constants for this example agent
AGENT_NAME = "bigquery_agent"
APP_NAME = "bigquery_app"
USER_ID = "user1234"
SESSION_ID = "1234"
GEMINI_MODEL = "gemini-2.5-flash"

# Define a tool configuration to block any write operations
tool_config = BigQueryToolConfig(write_mode=WriteMode.BLOCKED)

# Define a credentials config - in this example we are using application default
# credentials
# https://cloud.google.com/docs/authentication/provide-credentials-adc
application_default_credentials, _ = google.auth.default()
credentials_config = BigQueryCredentialsConfig(
    credentials=application_default_credentials
)

# Instantiate a BigQuery toolset
bigquery_toolset = BigQueryToolset(
    credentials_config=credentials_config, bigquery_tool_config=tool_config
)

# Agent Definition
bigquery_agent = Agent(
    model=GEMINI_MODEL,
    name=AGENT_NAME,
    description=(
        "Agent to answer questions about BigQuery data and models and execute"
        " SQL queries."
    ),
    instruction="""\
        You are a data science agent with access to several BigQuery tools.
        Make use of those tools to answer the user's questions.
    """,
    tools=[bigquery_toolset],
)

# Session and Runner
session_service = InMemorySessionService()
session = asyncio.run(session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID))
runner = Runner(agent=bigquery_agent, app_name=APP_NAME, session_service=session_service)

# Agent Interaction
def call_agent(query):
    """
    Helper function to call the agent with a query.
    """
    content = types.Content(role='user', parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    print("USER:", query)
    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("AGENT:", final_response)

call_agent("Are there any ml datasets in bigquery-public-data project?")
call_agent("Tell me more about ml_datasets.")
call_agent("Which all tables does it have?")
call_agent("Tell me more about the census_adult_income table.")
call_agent("How many rows are there per income bracket?")

```

## Use Built-in tools with other tools

The following code sample demonstrates how to use multiple built-in tools or how
to use built-in tools with other tools by using multiple agents:

=== "Python"

    ```py
    from google.adk.tools import agent_tool
    from google.adk.agents import Agent
    from google.adk.tools import google_search
    from google.adk.code_executors import BuiltInCodeExecutor
    

    search_agent = Agent(
        model='gemini-2.5-flash',
        name='SearchAgent',
        instruction="""
        You're a specialist in Google Search
        """,
        tools=[google_search],
    )
    coding_agent = Agent(
        model='gemini-2.5-flash',
        name='CodeAgent',
        instruction="""
        You're a specialist in Code Execution
        """,
        code_executor=[BuiltInCodeExecutor],
    )
    root_agent = Agent(
        name="RootAgent",
        model="gemini-2.5-flash",
        description="Root Agent",
        tools=[agent_tool.AgentTool(agent=search_agent), agent_tool.AgentTool(agent=coding_agent)],
    )
    ```

=== "Java"

    


### Limitations

!!! warning

    Currently, for each root agent or single agent, only one built-in tool is
    supported. No other tools of any type can be used in the same agent.

 For example, the following approach that uses ***a built-in tool along with
 other tools*** within a single agent is **not** currently supported:

=== "Python"

    ```py
    root_agent = Agent(
        name="RootAgent",
        model="gemini-2.5-flash",
        description="Root Agent",
        tools=[custom_function], 
        executor=[BuiltInCodeExecutor] # <-- not supported when used with tools
    )
    ```

=== "Java"

    

!!! warning

    Built-in tools cannot be used within a sub-agent.

For example, the following approach that uses built-in tools within sub-agents
is **not** currently supported:

=== "Python"

    ```py
    search_agent = Agent(
        model='gemini-2.5-flash',
        name='SearchAgent',
        instruction="""
        You're a specialist in Google Search
        """,
        tools=[google_search],
    )
    coding_agent = Agent(
        model='gemini-2.5-flash',
        name='CodeAgent',
        instruction="""
        You're a specialist in Code Execution
        """,
        executor=[BuiltInCodeExecutor],
    )
    root_agent = Agent(
        name="RootAgent",
        model="gemini-2.5-flash",
        description="Root Agent",
        sub_agents=[
            search_agent,
            coding_agent
        ],
    )
    ```

=== "Java"

    


# Function tools

## What are function tools?

When out-of-the-box tools don't fully meet specific requirements, developers can create custom function tools. This allows for **tailored functionality**, such as connecting to proprietary databases or implementing unique algorithms.

*For example,* a function tool, "myfinancetool", might be a function that calculates a specific financial metric. ADK also supports long running functions, so if that calculation takes a while, the agent can continue working on other tasks.

ADK offers several ways to create functions tools, each suited to different levels of complexity and control:

1. Function Tool
2. Long Running Function Tool
3. Agents-as-a-Tool

## 1. Function Tool

Transforming a function into a tool is a straightforward way to integrate custom logic into your agents. In fact, when you assign a function to an agent’s tools list, the framework will automatically wrap it as a Function Tool for you. This approach offers flexibility and quick integration.

### Parameters

Define your function parameters using standard **JSON-serializable types** (e.g., string, integer, list, dictionary). It's important to avoid setting default values for parameters, as the language model (LLM) does not currently support interpreting them.

### Return Type

The preferred return type for a Function Tool is a **dictionary** in Python or **Map** in Java. This allows you to structure the response with key-value pairs, providing context and clarity to the LLM. If your function returns a type other than a dictionary, the framework automatically wraps it into a dictionary with a single key named **"result"**.

Strive to make your return values as descriptive as possible. *For example,* instead of returning a numeric error code, return a dictionary with an "error\_message" key containing a human-readable explanation. **Remember that the LLM**, not a piece of code, needs to understand the result. As a best practice, include a "status" key in your return dictionary to indicate the overall outcome (e.g., "success", "error", "pending"), providing the LLM with a clear signal about the operation's state.

### Docstring / Source code comments

The docstring (or comments above) your function serve as the tool's description and is sent to the LLM. Therefore, a well-written and comprehensive docstring is crucial for the LLM to understand how to use the tool effectively. Clearly explain the purpose of the function, the meaning of its parameters, and the expected return values.

??? "Example"

    === "Python"
    
        This tool is a python function which obtains the Stock price of a given Stock ticker/ symbol.
    
        <u>Note</u>: You need to `pip install yfinance` library before using this tool.
    
        ```py
        # Copyright 2026 Google LLC
        #
        # Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        #
        #     http://www.apache.org/licenses/LICENSE-2.0
        #
        # Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.

        from google.adk.agents import Agent
        from google.adk.runners import Runner
        from google.adk.sessions import InMemorySessionService
        from google.genai import types

        import yfinance as yf


        APP_NAME = "stock_app"
        USER_ID = "1234"
        SESSION_ID = "session1234"

        def get_stock_price(symbol: str):
            """
            Retrieves the current stock price for a given symbol.

            Args:
                symbol (str): The stock symbol (e.g., "AAPL", "GOOG").

            Returns:
                float: The current stock price, or None if an error occurs.
            """
            try:
                stock = yf.Ticker(symbol)
                historical_data = stock.history(period="1d")
                if not historical_data.empty:
                    current_price = historical_data['Close'].iloc[-1]
                    return current_price
                else:
                    return None
            except Exception as e:
                print(f"Error retrieving stock price for {symbol}: {e}")
                return None


        stock_price_agent = Agent(
            model='gemini-2.5-flash',
            name='stock_agent',
            instruction= 'You are an agent who retrieves stock prices. If a ticker symbol is provided, fetch the current price. If only a company name is given, first perform a Google search to find the correct ticker symbol before retrieving the stock price. If the provided ticker symbol is invalid or data cannot be retrieved, inform the user that the stock price could not be found.',
            description='This agent specializes in retrieving real-time stock prices. Given a stock ticker symbol (e.g., AAPL, GOOG, MSFT) or the stock name, use the tools and reliable data sources to provide the most up-to-date price.',
            tools=[get_stock_price], # You can add Python functions directly to the tools list; they will be automatically wrapped as FunctionTools.
        )


        # Session and Runner
        async def setup_session_and_runner():
            session_service = InMemorySessionService()
            session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
            runner = Runner(agent=stock_price_agent, app_name=APP_NAME, session_service=session_service)
            return session, runner

        # Agent Interaction
        async def call_agent_async(query):
            content = types.Content(role='user', parts=[types.Part(text=query)])
            session, runner = await setup_session_and_runner()
            events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

            async for event in events:
                if event.is_final_response():
                    final_response = event.content.parts[0].text
                    print("Agent Response: ", final_response)


        # Note: In Colab, you can directly use 'await' at the top level.
        # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
        await call_agent_async("stock price of GOOG")

        ```
    
        The return value from this tool will be wrapped into a dictionary.
    
        ```json
        {"result": "$123"}
        ```
    
    === "Java"
    
        This tool retrieves the mocked value of a stock price.
    
        
    
        The return value from this tool will be wrapped into a Map<String, Object>.
    
        ```json
        For input `GOOG`: {"symbol": "GOOG", "price": "1.0"}
        ```

### Best Practices

While you have considerable flexibility in defining your function, remember that simplicity enhances usability for the LLM. Consider these guidelines:

* **Fewer Parameters are Better:** Minimize the number of parameters to reduce complexity.  
* **Simple Data Types:** Favor primitive data types like `str` and `int` over custom classes whenever possible.  
* **Meaningful Names:** The function's name and parameter names significantly influence how the LLM interprets and utilizes the tool. Choose names that clearly reflect the function's purpose and the meaning of its inputs. Avoid generic names like `do_stuff()` or `beAgent()`.  

## 2. Long Running Function Tool

Designed for tasks that require a significant amount of processing time without blocking the agent's execution. This tool is a subclass of `FunctionTool`.

When using a `LongRunningFunctionTool`, your function can initiate the long-running operation and optionally return an **initial result** (e.g. the long-running operation id). Once a long running function tool is invoked the agent runner will pause the agent run and let the agent client to decide whether to continue or wait until the long-running operation finishes. The agent client can query the progress of the long-running operation and send back an intermediate or final response. The agent can then continue with other tasks. An example is the human-in-the-loop scenario where the agent needs human approval before proceeding with a task.

### How it Works

In Python, you wrap a function with `LongRunningFunctionTool`. In Java, you pass a Method name to `LongRunningFunctionTool.create()`.


1. **Initiation:** When the LLM calls the tool, your function starts the long-running operation.

2. **Initial Updates:** Your function should optionally return an initial result (e.g. the long-running operation id). The ADK framework takes the result and sends it back to the LLM packaged within a `FunctionResponse`. This allows the LLM to inform the user (e.g., status, percentage complete, messages). And then the agent run is ended / paused.

3. **Continue or Wait:** After each agent run is completed. Agent client can query the progress of the long-running operation and decide whether to continue the agent run with an intermediate response (to update the progress) or wait until a final response is retrieved. Agent client should send the intermediate or final response back to the agent for the next run.

4. **Framework Handling:** The ADK framework manages the execution. It sends the intermediate or final `FunctionResponse` sent by agent client to the LLM to generate a user friendly message.

### Creating the Tool

Define your tool function and wrap it using the `LongRunningFunctionTool` class:

=== "Python"

    ```py
    # 1. Define the long running function
    def ask_for_approval(
        purpose: str, amount: float
    ) -> dict[str, Any]:
        """Ask for approval for the reimbursement."""
        # create a ticket for the approval
        # Send a notification to the approver with the link of the ticket
        return {'status': 'pending', 'approver': 'Sean Zhou', 'purpose' : purpose, 'amount': amount, 'ticket-id': 'approval-ticket-1'}
    def reimburse(purpose: str, amount: float) -> str:
        """Reimburse the amount of money to the employee."""
        # send the reimbrusement request to payment vendor
        return {'status': 'ok'}
    # 2. Wrap the function with LongRunningFunctionTool
    long_running_tool = LongRunningFunctionTool(func=ask_for_approval)
    ```

=== "Java"

    

### Intermediate / Final result Updates

Agent client received an event with long running function calls and check the status of the ticket. Then Agent client can send the intermediate or final response back to update the progress. The framework packages this value (even if it's None) into the content of the `FunctionResponse` sent back to the LLM.

!!! Tip "Applies to only Java ADK"

    When passing `ToolContext` with Function Tools, ensure that one of the following is true:

    * The Schema is passed with the ToolContext parameter in the function signature, like:
      ```
      @com.google.adk.tools.Annotations.Schema(name = "toolContext") ToolContext toolContext
      ```
    OR

    * The following `-parameters` flag is set to the mvn compiler plugin

    ```
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.14.0</version> <!-- or newer -->
                <configuration>
                    <compilerArgs>
                        <arg>-parameters</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
    ```
    This constraint is temporary and will be removed.


=== "Python"

    ```py
    --8<-- "examples/python/snippets/tools/function-tools/human_in_the_loop.py:call_reimbursement_tool"
    ```

=== "Java"

    


??? "Python complete example: File Processing Simulation"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    import asyncio
    from typing import Any
    from google.adk.agents import Agent
    from google.adk.events import Event
    from google.adk.runners import Runner
    from google.adk.tools import LongRunningFunctionTool
    from google.adk.sessions import InMemorySessionService
    from google.genai import types

    # --8<-- [start:define_long_running_function]

    # 1. Define the long running function
    def ask_for_approval(
        purpose: str, amount: float
    ) -> dict[str, Any]:
        """Ask for approval for the reimbursement."""
        # create a ticket for the approval
        # Send a notification to the approver with the link of the ticket
        return {'status': 'pending', 'approver': 'Sean Zhou', 'purpose' : purpose, 'amount': amount, 'ticket-id': 'approval-ticket-1'}

    def reimburse(purpose: str, amount: float) -> str:
        """Reimburse the amount of money to the employee."""
        # send the reimbrusement request to payment vendor
        return {'status': 'ok'}

    # 2. Wrap the function with LongRunningFunctionTool
    long_running_tool = LongRunningFunctionTool(func=ask_for_approval)

    # --8<-- [end:define_long_running_function]

    # 3. Use the tool in an Agent
    file_processor_agent = Agent(
        # Use a model compatible with function calling
        model="gemini-2.5-flash",
        name='reimbursement_agent',
        instruction="""
          You are an agent whose job is to handle the reimbursement process for
          the employees. If the amount is less than $100, you will automatically
          approve the reimbursement.

          If the amount is greater than $100, you will
          ask for approval from the manager. If the manager approves, you will
          call reimburse() to reimburse the amount to the employee. If the manager
          rejects, you will inform the employee of the rejection.
        """,
        tools=[reimburse, long_running_tool]
    )


    APP_NAME = "human_in_the_loop"
    USER_ID = "1234"
    SESSION_ID = "session1234"

    # Session and Runner
    async def setup_session_and_runner():
        session_service = InMemorySessionService()
        session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
        runner = Runner(agent=file_processor_agent, app_name=APP_NAME, session_service=session_service)
        return session, runner

    # --8<-- [start: call_reimbursement_tool]

    # Agent Interaction
    async def call_agent_async(query):

        def get_long_running_function_call(event: Event) -> types.FunctionCall:
            # Get the long running function call from the event
            if not event.long_running_tool_ids or not event.content or not event.content.parts:
                return
            for part in event.content.parts:
                if (
                    part
                    and part.function_call
                    and event.long_running_tool_ids
                    and part.function_call.id in event.long_running_tool_ids
                ):
                    return part.function_call

        def get_function_response(event: Event, function_call_id: str) -> types.FunctionResponse:
            # Get the function response for the function call with specified id.
            if not event.content or not event.content.parts:
                return
            for part in event.content.parts:
                if (
                    part
                    and part.function_response
                    and part.function_response.id == function_call_id
                ):
                    return part.function_response

        content = types.Content(role='user', parts=[types.Part(text=query)])
        session, runner = await setup_session_and_runner()
        events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

        print("\nRunning agent...")
        events_async = runner.run_async(
            session_id=session.id, user_id=USER_ID, new_message=content
        )


        long_running_function_call, long_running_function_response, ticket_id = None, None, None
        async for event in events_async:
            # Use helper to check for the specific auth request event
            if not long_running_function_call:
                long_running_function_call = get_long_running_function_call(event)
            else:
                long_running_function_response = get_function_response(event, long_running_function_call.id)
                if long_running_function_response:
                    ticket_id = long_running_function_response.response['ticket-id']
            if event.content and event.content.parts:
                if text := ''.join(part.text or '' for part in event.content.parts):
                    print(f'[{event.author}]: {text}')


        if long_running_function_response:
            # query the status of the correpsonding ticket via tciket_id
            # send back an intermediate / final response
            updated_response = long_running_function_response.model_copy(deep=True)
            updated_response.response = {'status': 'approved'}
            async for event in runner.run_async(
              session_id=session.id, user_id=USER_ID, new_message=types.Content(parts=[types.Part(function_response = updated_response)], role='user')
            ):
                if event.content and event.content.parts:
                    if text := ''.join(part.text or '' for part in event.content.parts):
                        print(f'[{event.author}]: {text}')
          
    # --8<-- [end:call_reimbursement_tool]          

    # Note: In Colab, you can directly use 'await' at the top level.
    # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
                   
    # reimbursement that doesn't require approval
    # asyncio.run(call_agent_async("Please reimburse 50$ for meals"))
    await call_agent_async("Please reimburse 50$ for meals") # For Notebooks, uncomment this line and comment the above line
    # reimbursement that requires approval
    # asyncio.run(call_agent_async("Please reimburse 200$ for meals"))
    await call_agent_async("Please reimburse 200$ for meals") # For Notebooks, uncomment this line and comment the above line

    ```

#### Key aspects of this example

* **`LongRunningFunctionTool`**: Wraps the supplied method/function; the framework handles sending yielded updates and the final return value as sequential FunctionResponses.

* **Agent instruction**: Directs the LLM to use the tool and understand the incoming FunctionResponse stream (progress vs. completion) for user updates.

* **Final return**: The function returns the final result dictionary, which is sent in the concluding FunctionResponse to indicate completion.

## 3. Agent-as-a-Tool

This powerful feature allows you to leverage the capabilities of other agents within your system by calling them as tools. The Agent-as-a-Tool enables you to invoke another agent to perform a specific task, effectively **delegating responsibility**. This is conceptually similar to creating a Python function that calls another agent and uses the agent's response as the function's return value.

### Key difference from sub-agents

It's important to distinguish an Agent-as-a-Tool from a Sub-Agent.

* **Agent-as-a-Tool:** When Agent A calls Agent B as a tool (using Agent-as-a-Tool), Agent B's answer is **passed back** to Agent A, which then summarizes the answer and generates a response to the user. Agent A retains control and continues to handle future user input.  

* **Sub-agent:** When Agent A calls Agent B as a sub-agent, the responsibility of answering the user is completely **transferred to Agent B**. Agent A is effectively out of the loop. All subsequent user input will be answered by Agent B.

### Usage

To use an agent as a tool, wrap the agent with the AgentTool class.

=== "Python"

    ```py
    tools=[AgentTool(agent=agent_b)]
    ```

=== "Java"

    

### Customization

The `AgentTool` class provides the following attributes for customizing its behavior:

* **skip\_summarization: bool:** If set to True, the framework will **bypass the LLM-based summarization** of the tool agent's response. This can be useful when the tool's response is already well-formatted and requires no further processing.

??? "Example"

    === "Python"

        ```py
        # Copyright 2026 Google LLC
        #
        # Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        #
        #     http://www.apache.org/licenses/LICENSE-2.0
        #
        # Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.

        from google.adk.agents import Agent
        from google.adk.runners import Runner
        from google.adk.sessions import InMemorySessionService
        from google.adk.tools.agent_tool import AgentTool
        from google.genai import types

        APP_NAME="summary_agent"
        USER_ID="user1234"
        SESSION_ID="1234"

        summary_agent = Agent(
            model="gemini-2.5-flash",
            name="summary_agent",
            instruction="""You are an expert summarizer. Please read the following text and provide a concise summary.""",
            description="Agent to summarize text",
        )

        root_agent = Agent(
            model='gemini-2.5-flash',
            name='root_agent',
            instruction="""You are a helpful assistant. When the user provides a text, use the 'summarize' tool to generate a summary. Always forward the user's message exactly as received to the 'summarize' tool, without modifying or summarizing it yourself. Present the response from the tool to the user.""",
            tools=[AgentTool(agent=summary_agent)]
        )

        # Session and Runner
        async def setup_session_and_runner():
            session_service = InMemorySessionService()
            session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
            runner = Runner(agent=root_agent, app_name=APP_NAME, session_service=session_service)
            return session, runner


        # Agent Interaction
        async def call_agent_async(query):
            content = types.Content(role='user', parts=[types.Part(text=query)])
            session, runner = await setup_session_and_runner()
            events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

            async for event in events:
                if event.is_final_response():
                    final_response = event.content.parts[0].text
                    print("Agent Response: ", final_response)


        long_text = """Quantum computing represents a fundamentally different approach to computation, 
        leveraging the bizarre principles of quantum mechanics to process information. Unlike classical computers 
        that rely on bits representing either 0 or 1, quantum computers use qubits which can exist in a state of superposition - effectively 
        being 0, 1, or a combination of both simultaneously. Furthermore, qubits can become entangled, 
        meaning their fates are intertwined regardless of distance, allowing for complex correlations. This parallelism and 
        interconnectedness grant quantum computers the potential to solve specific types of incredibly complex problems - such 
        as drug discovery, materials science, complex system optimization, and breaking certain types of cryptography - far 
        faster than even the most powerful classical supercomputers could ever achieve, although the technology is still largely in its developmental stages."""


        # Note: In Colab, you can directly use 'await' at the top level.
        # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
        await call_agent_async(long_text)

        ```
  
    === "Java"

        

### How it works

1. When the `main_agent` receives the long text, its instruction tells it to use the 'summarize' tool for long texts.  
2. The framework recognizes 'summarize' as an `AgentTool` that wraps the `summary_agent`.  
3. Behind the scenes, the `main_agent` will call the `summary_agent` with the long text as input.  
4. The `summary_agent` will process the text according to its instruction and generate a summary.  
5. **The response from the `summary_agent` is then passed back to the `main_agent`.**  
6. The `main_agent` can then take the summary and formulate its final response to the user (e.g., "Here's a summary of the text: ...")



# Google Cloud Tools

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

Google Cloud tools make it easier to connect your agents to Google Cloud’s
products and services. With just a few lines of code you can use these tools to
connect your agents with:

* **Any custom APIs** that developers host in Apigee.
* **100s** of **prebuilt connectors** to enterprise systems such as Salesforce,
  Workday, and SAP.
* **Automation workflows** built using application integration.
* **Databases** such as Spanner, AlloyDB, Postgres and more using the MCP Toolbox for
  databases.

![Google Cloud Tools](../assets/google_cloud_tools.svg)

## Apigee API Hub Tools

**ApiHubToolset** lets you turn any documented API from Apigee API hub into a
tool with a few lines of code. This section shows you the step by step
instructions including setting up authentication for a secure connection to your
APIs.

**Prerequisites**

1. [Install ADK](../get-started/installation.md)
2. Install the
   [Google Cloud CLI](https://cloud.google.com/sdk/docs/install?db=bigtable-docs#installation_instructions).
3. [Apigee API hub](https://cloud.google.com/apigee/docs/apihub/what-is-api-hub)
    instance with documented (i.e. OpenAPI spec) APIs
4. Set up your project structure and create required files

```console
project_root_folder
 |
 `-- my_agent
     |-- .env
     |-- __init__.py
     |-- agent.py
     `__ tool.py
```

### Create an API Hub Toolset

Note: This tutorial includes an agent creation. If you already have an agent,
you only need to follow a subset of these steps.

1. Get your access token, so that APIHubToolset can fetch spec from API Hub API.
   In your terminal run the following command

    ```shell
    gcloud auth print-access-token
    # Prints your access token like 'ya29....'
    ```

2. Ensure that the account used has the required permissions. You can use the
   pre-defined role `roles/apihub.viewer` or assign the following permissions:

    1. **apihub.specs.get (required)**
    2. apihub.apis.get (optional)
    3. apihub.apis.list (optional)
    4. apihub.versions.get (optional)
    5. apihub.versions.list (optional)
    6. apihub.specs.list (optional)

3. Create a tool with `APIHubToolset`. Add the below to `tools.py`

    If your API requires authentication, you must configure authentication for
    the tool. The following code sample demonstrates how to configure an API
    key. ADK supports token based auth (API Key, Bearer token), service account,
    and OpenID Connect. We will soon add support for various OAuth2 flows.

    ```py
    from google.adk.tools.openapi_tool.auth.auth_helpers import token_to_scheme_credential
    from google.adk.tools.apihub_tool.apihub_toolset import APIHubToolset

    # Provide authentication for your APIs. Not required if your APIs don't required authentication.
    auth_scheme, auth_credential = token_to_scheme_credential(
        "apikey", "query", "apikey", apikey_credential_str
    )

    sample_toolset_with_auth = APIHubToolset(
        name="apihub-sample-tool",
        description="Sample Tool",
        access_token="...",  # Copy your access token generated in step 1
        apihub_resource_name="...", # API Hub resource name
        auth_scheme=auth_scheme,
        auth_credential=auth_credential,
    )
    ```

    For production deployment we recommend using a service account instead of an
    access token. In the code snippet above, use
    `service_account_json=service_account_cred_json_str` and provide your
    security account credentials instead of the token.

    For apihub\_resource\_name, if you know the specific ID of the OpenAPI Spec
    being used for your API, use
    `` `projects/my-project-id/locations/us-west1/apis/my-api-id/versions/version-id/specs/spec-id` ``.
    If you would like the Toolset to automatically pull the first available spec
    from the API, use
    `` `projects/my-project-id/locations/us-west1/apis/my-api-id` ``

4. Create your agent file Agent.py and add the created tools to your agent
   definition:

    ```py
    from google.adk.agents.llm_agent import LlmAgent
    from .tools import sample_toolset

    root_agent = LlmAgent(
        model='gemini-2.5-flash',
        name='enterprise_assistant',
        instruction='Help user, leverage the tools you have access to',
        tools=sample_toolset.get_tools(),
    )
    ```

5. Configure your `__init__.py` to expose your agent

    ```py
    from . import agent
    ```

6. Start the Google ADK Web UI and try your agent:

    ```shell
    # make sure to run `adk web` from your project_root_folder
    adk web
    ```

   Then go to [http://localhost:8000](http://localhost:8000) to try your agent from the Web UI.

---

## Application Integration Tools

With **ApplicationIntegrationToolset** you can seamlessly give your agents a
secure and governed to enterprise applications using Integration Connector’s
100+ pre-built connectors for systems like Salesforce, ServiceNow, JIRA, SAP,
and more. Support for both on-prem and SaaS applications. In addition you can
turn your existing Application Integration process automations into agentic
workflows by providing application integration workflows as tools to your ADK
agents.

**Prerequisites**

1. [Install ADK](../get-started/installation.md)
2. An existing
   [Application Integration](https://cloud.google.com/application-integration/docs/overview)
   workflow or
   [Integrations Connector](https://cloud.google.com/integration-connectors/docs/overview)
   connection you want to use with your agent
3. To use tool with default credentials: have Google Cloud CLI installed. See
   [installation guide](https://cloud.google.com/sdk/docs/install#installation_instructions)*.*

   *Run:*

   ```shell
   gcloud config set project <project-id>
   gcloud auth application-default login
   gcloud auth application-default set-quota-project <project-id>
   ```

5. Set up your project structure and create required files

    ```console
    project_root_folder
    |-- .env
    `-- my_agent
        |-- __init__.py
        |-- agent.py
        `__ tools.py
    ```

When running the agent, make sure to run adk web in project\_root\_folder

### Use Integration Connectors

Connect your agent to enterprise applications using
[Integration Connectors](https://cloud.google.com/integration-connectors/docs/overview).

**Prerequisites**

1. To use a connector from Integration Connectors, you need to [provision](https://console.cloud.google.com/integrations)
   Application Integration in the same region as your connection by clicking on "QUICK SETUP" button.


   ![Google Cloud Tools](../assets/application-integration-overview.png)

2. Go to [Connection Tool](https://console.cloud.google.com/integrations/templates/connection-tool/locations/us-central1)
   template from the template library and click on "USE TEMPLATE" button.


    ![Google Cloud Tools](../assets/use-connection-tool-template.png)

3. Fill the Integration Name as **ExecuteConnection** (It is mandatory to use this integration name only) and
   select the region same as the connection region. Click on "CREATE".

4. Publish the integration by using the "PUBLISH" button on the Application Integration Editor.


    ![Google Cloud Tools](../assets/publish-integration.png)

**Steps:**

1.  Create a tool with `ApplicationIntegrationToolset` within your `tools.py` file

    ```py
    from google.adk.tools.application_integration_tool.application_integration_toolset import ApplicationIntegrationToolset

    connector_tool = ApplicationIntegrationToolset(
        project="test-project", # TODO: replace with GCP project of the connection
        location="us-central1", #TODO: replace with location of the connection
        connection="test-connection", #TODO: replace with connection name
        entity_operations={"Entity_One": ["LIST","CREATE"], "Entity_Two": []},#empty list for actions means all operations on the entity are supported.
        actions=["action1"], #TODO: replace with actions
        service_account_credentials='{...}', # optional. Stringified json for service account key
        tool_name_prefix="tool_prefix2",
        tool_instructions="..."
    )
    ```

    **Note:**

    * You can provide service account to be used instead of using default credentials by generating [Service Account Key](https://cloud.google.com/iam/docs/keys-create-delete#creating) and providing right Application Integration and Integration Connector IAM roles to the service account.
    * To find the list of supported entities and actions for a connection, use the connectors apis: [listActions](https://cloud.google.com/integration-connectors/docs/reference/rest/v1/projects.locations.connections.connectionSchemaMetadata/listActions) or [listEntityTypes](https://cloud.google.com/integration-connectors/docs/reference/rest/v1/projects.locations.connections.connectionSchemaMetadata/listEntityTypes)


    `ApplicationIntegrationToolset` now also supports providing auth_scheme and auth_credential for dynamic OAuth2 authentication for Integration Connectors. To use it, create a tool similar to this within your `tools.py` file:

    ```py
    from google.adk.tools.application_integration_tool.application_integration_toolset import ApplicationIntegrationToolset
    from google.adk.tools.openapi_tool.auth.auth_helpers import dict_to_auth_scheme
    from google.adk.auth import AuthCredential
    from google.adk.auth import AuthCredentialTypes
    from google.adk.auth import OAuth2Auth

    oauth2_data_google_cloud = {
      "type": "oauth2",
      "flows": {
          "authorizationCode": {
              "authorizationUrl": "https://accounts.google.com/o/oauth2/auth",
              "tokenUrl": "https://oauth2.googleapis.com/token",
              "scopes": {
                  "https://www.googleapis.com/auth/cloud-platform": (
                      "View and manage your data across Google Cloud Platform"
                      " services"
                  ),
                  "https://www.googleapis.com/auth/calendar.readonly": "View your calendars"
              },
          }
      },
    }

    oauth_scheme = dict_to_auth_scheme(oauth2_data_google_cloud)

    auth_credential = AuthCredential(
      auth_type=AuthCredentialTypes.OAUTH2,
      oauth2=OAuth2Auth(
          client_id="...", #TODO: replace with client_id
          client_secret="...", #TODO: replace with client_secret
      ),
    )

    connector_tool = ApplicationIntegrationToolset(
        project="test-project", # TODO: replace with GCP project of the connection
        location="us-central1", #TODO: replace with location of the connection
        connection="test-connection", #TODO: replace with connection name
        entity_operations={"Entity_One": ["LIST","CREATE"], "Entity_Two": []},#empty list for actions means all operations on the entity are supported.
        actions=["GET_calendars/%7BcalendarId%7D/events"], #TODO: replace with actions. this one is for list events
        service_account_credentials='{...}', # optional. Stringified json for service account key
        tool_name_prefix="tool_prefix2",
        tool_instructions="...",
        auth_scheme=oauth_scheme,
        auth_credential=auth_credential
    )
    ```


2. Add the tool to your agent. Update your `agent.py` file

    ```py
    from google.adk.agents.llm_agent import LlmAgent
    from .tools import connector_tool

    root_agent = LlmAgent(
        model='gemini-2.5-flash',
        name='connector_agent',
        instruction="Help user, leverage the tools you have access to",
        tools=[connector_tool],
    )
    ```

3. Configure your  `__init__.py` to expose your agent

    ```py
    from . import agent
    ```

4. Start the Google ADK Web UI and try your agent.

    ```shell
    # make sure to run `adk web` from your project_root_folder
    adk web
    ```

   Then go to [http://localhost:8000](http://localhost:8000), and choose
   my\_agent agent (same as the agent folder name)

### Use App Integration Workflows

Use existing
[Application Integration](https://cloud.google.com/application-integration/docs/overview)
workflow as a tool for your agent or create a new one.

**Steps:**

1. Create a tool with `ApplicationIntegrationToolset` within your `tools.py` file

    ```py
    integration_tool = ApplicationIntegrationToolset(
        project="test-project", # TODO: replace with GCP project of the connection
        location="us-central1", #TODO: replace with location of the connection
        integration="test-integration", #TODO: replace with integration name
        triggers=["api_trigger/test_trigger"],#TODO: replace with trigger id(s). Empty list would mean all api triggers in the integration to be considered.
        service_account_credentials='{...}', #optional. Stringified json for service account key
        tool_name_prefix="tool_prefix1",
        tool_instructions="..."
    )
    ```

    Note: You can provide service account to be used instead of using default
        credentials by generating [Service Account Key](https://cloud.google.com/iam/docs/keys-create-delete#creating) and providing right Application Integration and Integration Connector IAM roles to the service account.

2. Add the tool to your agent. Update your `agent.py` file

    ```py
    from google.adk.agents.llm_agent import LlmAgent
    from .tools import integration_tool, connector_tool

    root_agent = LlmAgent(
        model='gemini-2.5-flash',
        name='integration_agent',
        instruction="Help user, leverage the tools you have access to",
        tools=[integration_tool],
    )
    ```

3. Configure your \`\_\_init\_\_.py\` to expose your agent

    ```py
    from . import agent
    ```

4. Start the Google ADK Web UI and try your agent.

    ```shell
    # make sure to run `adk web` from your project_root_folder
    adk web
    ```

    Then go to [http://localhost:8000](http://localhost:8000), and choose
    my\_agent agent (same as the agent folder name)

---

## Toolbox Tools for Databases

[MCP Toolbox for Databases](https://github.com/googleapis/genai-toolbox) is an
open source MCP server for databases. It was designed with enterprise-grade and
production-quality in mind. It enables you to develop tools easier, faster, and
more securely by handling the complexities such as connection pooling,
authentication, and more.

Google’s Agent Development Kit (ADK) has built in support for Toolbox. For more
information on
[getting started](https://googleapis.github.io/genai-toolbox/getting-started) or
[configuring](https://googleapis.github.io/genai-toolbox/getting-started/configure/)
Toolbox, see the
[documentation](https://googleapis.github.io/genai-toolbox/getting-started/introduction/).

![GenAI Toolbox](../assets/mcp_db_toolbox.png)

### Configure and deploy

Toolbox is an open source server that you deploy and manage yourself. For more
instructions on deploying and configuring, see the official Toolbox
documentation:

* [Installing the Server](https://googleapis.github.io/genai-toolbox/getting-started/introduction/#installing-the-server)
* [Configuring Toolbox](https://googleapis.github.io/genai-toolbox/getting-started/configure/)

### Install client SDK

ADK relies on the `toolbox-core` python package to use Toolbox. Install the
package before getting started:

```shell
pip install toolbox-core
```

### Loading Toolbox Tools

Once you’re Toolbox server is configured and up and running, you can load tools
from your server using ADK:

```python
from google.adk.agents import Agent
from toolbox_core import ToolboxSyncClient

toolbox = ToolboxSyncClient("https://127.0.0.1:5000")

# Load a specific set of tools
tools = toolbox.load_toolset('my-toolset-name'),
# Load single tool
tools = toolbox.load_tool('my-tool-name'),

root_agent = Agent(
    ...,
    tools=tools # Provide the list of tools to the Agent

)
```

### Advanced Toolbox Features

Toolbox has a variety of features to make developing Gen AI tools for databases.
For more information, read more about the following features:

* [Authenticated Parameters](https://googleapis.github.io/genai-toolbox/resources/tools/#authenticated-parameters): bind tool inputs to values from OIDC tokens automatically, making it easy to run sensitive queries without potentially leaking data
* [Authorized Invocations:](https://googleapis.github.io/genai-toolbox/resources/tools/#authorized-invocations)  restrict access to use a tool based on the users Auth token
* [OpenTelemetry](https://googleapis.github.io/genai-toolbox/how-to/export_telemetry/): get metrics and tracing from Toolbox with OpenTelemetry


# Tools

## What is a Tool?

In the context of ADK, a Tool represents a specific
capability provided to an AI agent, enabling it to perform actions and interact
with the world beyond its core text generation and reasoning abilities. What
distinguishes capable agents from basic language models is often their effective
use of tools.

Technically, a tool is typically a modular code component—**like a Python/ Java
function**, a class method, or even another specialized agent—designed to
execute a distinct, predefined task. These tasks often involve interacting with
external systems or data.

<img src="../assets/agent-tool-call.png" alt="Agent tool call">

### Key Characteristics

**Action-Oriented:** Tools perform specific actions, such as:

* Querying databases
* Making API requests (e.g., fetching weather data, booking systems)
* Searching the web
* Executing code snippets
* Retrieving information from documents (RAG)
* Interacting with other software or services

**Extends Agent capabilities:** They empower agents to access real-time information, affect external systems, and overcome the knowledge limitations inherent in their training data.

**Execute predefined logic:** Crucially, tools execute specific, developer-defined logic. They do not possess their own independent reasoning capabilities like the agent's core Large Language Model (LLM). The LLM reasons about which tool to use, when, and with what inputs, but the tool itself just executes its designated function.

## How Agents Use Tools

Agents leverage tools dynamically through mechanisms often involving function calling. The process generally follows these steps:

1. **Reasoning:** The agent's LLM analyzes its system instruction, conversation history, and user request.
2. **Selection:** Based on the analysis, the LLM decides on which tool, if any, to execute, based on the tools available to the agent and the docstrings that describes each tool.
3. **Invocation:** The LLM generates the required arguments (inputs) for the selected tool and triggers its execution.
4. **Observation:** The agent receives the output (result) returned by the tool.
5. **Finalization:** The agent incorporates the tool's output into its ongoing reasoning process to formulate the next response, decide the subsequent step, or determine if the goal has been achieved.

Think of the tools as a specialized toolkit that the agent's intelligent core (the LLM) can access and utilize as needed to accomplish complex tasks.

## Tool Types in ADK

ADK offers flexibility by supporting several types of tools:

1. **[Function Tools](../tools/function-tools.md):** Tools created by you, tailored to your specific application's needs.
    * **[Functions/Methods](../tools/function-tools.md#1-function-tool):** Define standard synchronous functions or methods in your code (e.g., Python def).
    * **[Agents-as-Tools](../tools/function-tools.md#3-agent-as-a-tool):** Use another, potentially specialized, agent as a tool for a parent agent.
    * **[Long Running Function Tools](../tools/function-tools.md#2-long-running-function-tool):** Support for tools that perform asynchronous operations or take significant time to complete.
2. **[Built-in Tools](../tools/built-in-tools.md):** Ready-to-use tools provided by the framework for common tasks.
        Examples: Google Search, Code Execution, Retrieval-Augmented Generation (RAG).
3. **[Third-Party Tools](../tools/third-party-tools.md):** Integrate tools seamlessly from popular external libraries.
        Examples: LangChain Tools, CrewAI Tools.

Navigate to the respective documentation pages linked above for detailed information and examples for each tool type.

## Referencing Tool in Agent’s Instructions

Within an agent's instructions, you can directly reference a tool by using its **function name.** If the tool's **function name** and **docstring** are sufficiently descriptive, your instructions can primarily focus on **when the Large Language Model (LLM) should utilize the tool**. This promotes clarity and helps the model understand the intended use of each tool.

It is **crucial to clearly instruct the agent on how to handle different return values** that a tool might produce. For example, if a tool returns an error message, your instructions should specify whether the agent should retry the operation, give up on the task, or request additional information from the user.

Furthermore, ADK supports the sequential use of tools, where the output of one tool can serve as the input for another. When implementing such workflows, it's important to **describe the intended sequence of tool usage** within the agent's instructions to guide the model through the necessary steps.

### Example

The following example showcases how an agent can use tools by **referencing their function names in its instructions**. It also demonstrates how to guide the agent to **handle different return values from tools**, such as success or error messages, and how to orchestrate the **sequential use of multiple tools** to accomplish a task.

=== "Python"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    from google.adk.agents import Agent
    from google.adk.tools import FunctionTool
    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from google.genai import types

    APP_NAME="weather_sentiment_agent"
    USER_ID="user1234"
    SESSION_ID="1234"
    MODEL_ID="gemini-2.5-flash"

    # Tool 1
    def get_weather_report(city: str) -> dict:
        """Retrieves the current weather report for a specified city.

        Returns:
            dict: A dictionary containing the weather information with a 'status' key ('success' or 'error') and a 'report' key with the weather details if successful, or an 'error_message' if an error occurred.
        """
        if city.lower() == "london":
            return {"status": "success", "report": "The current weather in London is cloudy with a temperature of 18 degrees Celsius and a chance of rain."}
        elif city.lower() == "paris":
            return {"status": "success", "report": "The weather in Paris is sunny with a temperature of 25 degrees Celsius."}
        else:
            return {"status": "error", "error_message": f"Weather information for '{city}' is not available."}

    weather_tool = FunctionTool(func=get_weather_report)


    # Tool 2
    def analyze_sentiment(text: str) -> dict:
        """Analyzes the sentiment of the given text.

        Returns:
            dict: A dictionary with 'sentiment' ('positive', 'negative', or 'neutral') and a 'confidence' score.
        """
        if "good" in text.lower() or "sunny" in text.lower():
            return {"sentiment": "positive", "confidence": 0.8}
        elif "rain" in text.lower() or "bad" in text.lower():
            return {"sentiment": "negative", "confidence": 0.7}
        else:
            return {"sentiment": "neutral", "confidence": 0.6}

    sentiment_tool = FunctionTool(func=analyze_sentiment)


    # Agent
    weather_sentiment_agent = Agent(
        model=MODEL_ID,
        name='weather_sentiment_agent',
        instruction="""You are a helpful assistant that provides weather information and analyzes the sentiment of user feedback.
    **If the user asks about the weather in a specific city, use the 'get_weather_report' tool to retrieve the weather details.**
    **If the 'get_weather_report' tool returns a 'success' status, provide the weather report to the user.**
    **If the 'get_weather_report' tool returns an 'error' status, inform the user that the weather information for the specified city is not available and ask if they have another city in mind.**
    **After providing a weather report, if the user gives feedback on the weather (e.g., 'That's good' or 'I don't like rain'), use the 'analyze_sentiment' tool to understand their sentiment.** Then, briefly acknowledge their sentiment.
    You can handle these tasks sequentially if needed.""",
        tools=[weather_tool, sentiment_tool]
    )

    # Session and Runner
    async def setup_session_and_runner():
        session_service = InMemorySessionService()
        session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
        runner = Runner(agent=weather_sentiment_agent, app_name=APP_NAME, session_service=session_service)
        return session, runner


    # Agent Interaction
    async def call_agent_async(query):
        content = types.Content(role='user', parts=[types.Part(text=query)])
        session, runner = await setup_session_and_runner()
        events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

        async for event in events:
            if event.is_final_response():
                final_response = event.content.parts[0].text
                print("Agent Response: ", final_response)

    # Note: In Colab, you can directly use 'await' at the top level.
    # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
    await call_agent_async("weather in london?")

    ```

=== "Java"

    

## Tool Context

For more advanced scenarios, ADK allows you to access additional contextual information within your tool function by including the special parameter `tool_context: ToolContext`. By including this in the function signature, ADK will **automatically** provide an **instance of the ToolContext** class when your tool is called during agent execution.

The **ToolContext** provides access to several key pieces of information and control levers:

* `state: State`: Read and modify the current session's state. Changes made here are tracked and persisted.

* `actions: EventActions`: Influence the agent's subsequent actions after the tool runs (e.g., skip summarization, transfer to another agent).

* `function_call_id: str`: The unique identifier assigned by the framework to this specific invocation of the tool. Useful for tracking and correlating with authentication responses. This can also be helpful when multiple tools are called within a single model response.

* `function_call_event_id: str`: This attribute provides the unique identifier of the **event** that triggered the current tool call. This can be useful for tracking and logging purposes.

* `auth_response: Any`: Contains the authentication response/credentials if an authentication flow was completed before this tool call.

* Access to Services: Methods to interact with configured services like Artifacts and Memory.

Note that you shouldn't include the `tool_context` parameter in the tool function docstring. Since `ToolContext` is automatically injected by the ADK framework *after* the LLM decides to call the tool function, it is not relevant for the LLM's decision-making and including it can confuse the LLM.

### **State Management**

The `tool_context.state` attribute provides direct read and write access to the state associated with the current session. It behaves like a dictionary but ensures that any modifications are tracked as deltas and persisted by the session service. This enables tools to maintain and share information across different interactions and agent steps.

* **Reading State**: Use standard dictionary access (`tool_context.state['my_key']`) or the `.get()` method (`tool_context.state.get('my_key', default_value)`).

* **Writing State**: Assign values directly (`tool_context.state['new_key'] = 'new_value'`). These changes are recorded in the state_delta of the resulting event.

* **State Prefixes**: Remember the standard state prefixes:

    * `app:*`: Shared across all users of the application.

    * `user:*`: Specific to the current user across all their sessions.

    * (No prefix): Specific to the current session.

    * `temp:*`: Temporary, not persisted across invocations (useful for passing data within a single run call but generally less useful inside a tool context which operates between LLM calls).

=== "Python"

    ```py
    from google.adk.tools import ToolContext, FunctionTool

    def update_user_preference(preference: str, value: str, tool_context: ToolContext):
        """Updates a user-specific preference."""
        user_prefs_key = "user:preferences"
        # Get current preferences or initialize if none exist
        preferences = tool_context.state.get(user_prefs_key, {})
        preferences[preference] = value
        # Write the updated dictionary back to the state
        tool_context.state[user_prefs_key] = preferences
        print(f"Tool: Updated user preference '{preference}' to '{value}'")
        return {"status": "success", "updated_preference": preference}

    pref_tool = FunctionTool(func=update_user_preference)

    # In an Agent:
    # my_agent = Agent(..., tools=[pref_tool])

    # When the LLM calls update_user_preference(preference='theme', value='dark', ...):
    # The tool_context.state will be updated, and the change will be part of the
    # resulting tool response event's actions.state_delta.

    ```

=== "Java"

    

### **Controlling Agent Flow**

The `tool_context.actions` attribute (`ToolContext.actions()` in Java) holds an **EventActions** object. Modifying attributes on this object allows your tool to influence what the agent or framework does after the tool finishes execution.

* **`skip_summarization: bool`**: (Default: False) If set to True, instructs the ADK to bypass the LLM call that typically summarizes the tool's output. This is useful if your tool's return value is already a user-ready message.

* **`transfer_to_agent: str`**: Set this to the name of another agent. The framework will halt the current agent's execution and **transfer control of the conversation to the specified agent**. This allows tools to dynamically hand off tasks to more specialized agents.

* **`escalate: bool`**: (Default: False) Setting this to True signals that the current agent cannot handle the request and should pass control up to its parent agent (if in a hierarchy). In a LoopAgent, setting **escalate=True** in a sub-agent's tool will terminate the loop.

#### Example

=== "Python"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    from google.adk.agents import Agent
    from google.adk.tools import FunctionTool
    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from google.adk.tools import ToolContext
    from google.genai import types

    APP_NAME="customer_support_agent"
    USER_ID="user1234"
    SESSION_ID="1234"


    def check_and_transfer(query: str, tool_context: ToolContext) -> str:
        """Checks if the query requires escalation and transfers to another agent if needed."""
        if "urgent" in query.lower():
            print("Tool: Detected urgency, transferring to the support agent.")
            tool_context.actions.transfer_to_agent = "support_agent"
            return "Transferring to the support agent..."
        else:
            return f"Processed query: '{query}'. No further action needed."

    escalation_tool = FunctionTool(func=check_and_transfer)

    main_agent = Agent(
        model='gemini-2.5-flash',
        name='main_agent',
        instruction="""You are the first point of contact for customer support of an analytics tool. Answer general queries. If the user indicates urgency, use the 'check_and_transfer' tool.""",
        tools=[check_and_transfer]
    )

    support_agent = Agent(
        model='gemini-2.5-flash',
        name='support_agent',
        instruction="""You are the dedicated support agent. Mentioned you are a support handler and please help the user with their urgent issue."""
    )

    main_agent.sub_agents = [support_agent]

    # Session and Runner
    async def setup_session_and_runner():
        session_service = InMemorySessionService()
        session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
        runner = Runner(agent=main_agent, app_name=APP_NAME, session_service=session_service)
        return session, runner

    # Agent Interaction
    async def call_agent_async(query):
        content = types.Content(role='user', parts=[types.Part(text=query)])
        session, runner = await setup_session_and_runner()
        events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

        async for event in events:
            if event.is_final_response():
                final_response = event.content.parts[0].text
                print("Agent Response: ", final_response)

    # Note: In Colab, you can directly use 'await' at the top level.
    # If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
    await call_agent_async("this is urgent, i cant login")
    ```

=== "Java"

    

##### Explanation

* We define two agents: `main_agent` and `support_agent`. The `main_agent` is designed to be the initial point of contact.
* The `check_and_transfer` tool, when called by `main_agent`, examines the user's query.
* If the query contains the word "urgent", the tool accesses the `tool_context`, specifically **`tool_context.actions`**, and sets the transfer\_to\_agent attribute to `support_agent`.
* This action signals to the framework to **transfer the control of the conversation to the agent named `support_agent`**.
* When the `main_agent` processes the urgent query, the `check_and_transfer` tool triggers the transfer. The subsequent response would ideally come from the `support_agent`.
* For a normal query without urgency, the tool simply processes it without triggering a transfer.

This example illustrates how a tool, through EventActions in its ToolContext, can dynamically influence the flow of the conversation by transferring control to another specialized agent.

### **Authentication**

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

ToolContext provides mechanisms for tools interacting with authenticated APIs. If your tool needs to handle authentication, you might use the following:

* **`auth_response`**: Contains credentials (e.g., a token) if authentication was already handled by the framework before your tool was called (common with RestApiTool and OpenAPI security schemes).

* **`request_credential(auth_config: dict)`**: Call this method if your tool determines authentication is needed but credentials aren't available. This signals the framework to start an authentication flow based on the provided auth_config.

* **`get_auth_response()`**: Call this in a subsequent invocation (after request_credential was successfully handled) to retrieve the credentials the user provided.

For detailed explanations of authentication flows, configuration, and examples, please refer to the dedicated Tool Authentication documentation page.

### **Context-Aware Data Access Methods**

These methods provide convenient ways for your tool to interact with persistent data associated with the session or user, managed by configured services.

* **`list_artifacts()`** (or **`listArtifacts()`** in Java): Returns a list of filenames (or keys) for all artifacts currently stored for the session via the artifact_service. Artifacts are typically files (images, documents, etc.) uploaded by the user or generated by tools/agents.

* **`load_artifact(filename: str)`**: Retrieves a specific artifact by its filename from the **artifact_service**. You can optionally specify a version; if omitted, the latest version is returned. Returns a `google.genai.types.Part` object containing the artifact data and mime type, or None if not found.

* **`save_artifact(filename: str, artifact: types.Part)`**: Saves a new version of an artifact to the artifact_service. Returns the new version number (starting from 0).

* **`search_memory(query: str)`** ![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

       Queries the user's long-term memory using the configured `memory_service`. This is useful for retrieving relevant information from past interactions or stored knowledge. The structure of the **SearchMemoryResponse** depends on the specific memory service implementation but typically contains relevant text snippets or conversation excerpts.

#### Example

=== "Python"

    ```py
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    from google.adk.tools import ToolContext, FunctionTool
    from google.genai import types


    def process_document(
        document_name: str, analysis_query: str, tool_context: ToolContext
    ) -> dict:
        """Analyzes a document using context from memory."""

        # 1. Load the artifact
        print(f"Tool: Attempting to load artifact: {document_name}")
        document_part = tool_context.load_artifact(document_name)

        if not document_part:
            return {"status": "error", "message": f"Document '{document_name}' not found."}

        document_text = document_part.text  # Assuming it's text for simplicity
        print(f"Tool: Loaded document '{document_name}' ({len(document_text)} chars).")

        # 2. Search memory for related context
        print(f"Tool: Searching memory for context related to: '{analysis_query}'")
        memory_response = tool_context.search_memory(
            f"Context for analyzing document about {analysis_query}"
        )
        memory_context = "\n".join(
            [
                m.events[0].content.parts[0].text
                for m in memory_response.memories
                if m.events and m.events[0].content
            ]
        )  # Simplified extraction
        print(f"Tool: Found memory context: {memory_context[:100]}...")

        # 3. Perform analysis (placeholder)
        analysis_result = f"Analysis of '{document_name}' regarding '{analysis_query}' using memory context: [Placeholder Analysis Result]"
        print("Tool: Performed analysis.")

        # 4. Save the analysis result as a new artifact
        analysis_part = types.Part.from_text(text=analysis_result)
        new_artifact_name = f"analysis_{document_name}"
        version = await tool_context.save_artifact(new_artifact_name, analysis_part)
        print(f"Tool: Saved analysis result as '{new_artifact_name}' version {version}.")

        return {
            "status": "success",
            "analysis_artifact": new_artifact_name,
            "version": version,
        }


    doc_analysis_tool = FunctionTool(func=process_document)

    # In an Agent:
    # Assume artifact 'report.txt' was previously saved.
    # Assume memory service is configured and has relevant past data.
    # my_agent = Agent(..., tools=[doc_analysis_tool], artifact_service=..., memory_service=...)

    ```

=== "Java"

    

By leveraging the **ToolContext**, developers can create more sophisticated and context-aware custom tools that seamlessly integrate with ADK's architecture and enhance the overall capabilities of their agents.

## Defining Effective Tool Functions

When using a method or function as an ADK Tool, how you define it significantly impacts the agent's ability to use it correctly. The agent's Large Language Model (LLM) relies heavily on the function's **name**, **parameters (arguments)**, **type hints**, and **docstring** / **source code comments** to understand its purpose and generate the correct call.

Here are key guidelines for defining effective tool functions:

* **Function Name:**
    * Use descriptive, verb-noun based names that clearly indicate the action (e.g., `get_weather`, `searchDocuments`, `schedule_meeting`).
    * Avoid generic names like `run`, `process`, `handle_data`, or overly ambiguous names like `doStuff`. Even with a good description, a name like `do_stuff` might confuse the model about when to use the tool versus, for example, `cancelFlight`.
    * The LLM uses the function name as a primary identifier during tool selection.

* **Parameters (Arguments):**
    * Your function can have any number of parameters.
    * Use clear and descriptive names (e.g., `city` instead of `c`, `search_query` instead of `q`).
    * **Provide type hints in Python**  for all parameters (e.g., `city: str`, `user_id: int`, `items: list[str]`). This is essential for ADK to generate the correct schema for the LLM.
    * Ensure all parameter types are **JSON serializable**. All java primitives as well as standard Python types like `str`, `int`, `float`, `bool`, `list`, `dict`, and their combinations are generally safe. Avoid complex custom class instances as direct parameters unless they have a clear JSON representation.
    * **Do not set default values** for parameters. E.g., `def my_func(param1: str = "default")`. Default values are not reliably supported or used by the underlying models during function call generation. All necessary information should be derived by the LLM from the context or explicitly requested if missing.
    * **`self` / `cls` Handled Automatically:** Implicit parameters like `self` (for instance methods) or `cls` (for class methods) are automatically handled by ADK and excluded from the schema shown to the LLM. You only need to define type hints and descriptions for the logical parameters your tool requires the LLM to provide.

* **Return Type:**
    * The function's return value **must be a dictionary (`dict`)** in Python or a **Map** in Java.
    * If your function returns a non-dictionary type (e.g., a string, number, list), the ADK framework will automatically wrap it into a dictionary/Map like `{'result': your_original_return_value}` before passing the result back to the model.
    * Design the dictionary/Map keys and values to be **descriptive and easily understood *by the LLM***. Remember, the model reads this output to decide its next step.
    * Include meaningful keys. For example, instead of returning just an error code like `500`, return `{'status': 'error', 'error_message': 'Database connection failed'}`.
    * It's a **highly recommended practice** to include a `status` key (e.g., `'success'`, `'error'`, `'pending'`, `'ambiguous'`) to clearly indicate the outcome of the tool execution for the model.

* **Docstring / Source Code Comments:**
    * **This is critical.** The docstring is the primary source of descriptive information for the LLM.
    * **Clearly state what the tool *does*.** Be specific about its purpose and limitations.
    * **Explain *when* the tool should be used.** Provide context or example scenarios to guide the LLM's decision-making.
    * **Describe *each parameter* clearly.** Explain what information the LLM needs to provide for that argument.
    * Describe the **structure and meaning of the expected `dict` return value**, especially the different `status` values and associated data keys.
    * **Do not describe the injected ToolContext parameter**. Avoid mentioning the optional `tool_context: ToolContext` parameter within the docstring description since it is not a parameter the LLM needs to know about. ToolContext is injected by ADK, *after* the LLM decides to call it. 

    **Example of a good definition:**

=== "Python"
    
    ```python
    def lookup_order_status(order_id: str) -> dict:
      """Fetches the current status of a customer's order using its ID.

      Use this tool ONLY when a user explicitly asks for the status of
      a specific order and provides the order ID. Do not use it for
      general inquiries.

      Args:
          order_id: The unique identifier of the order to look up.

      Returns:
          A dictionary containing the order status.
          Possible statuses: 'shipped', 'processing', 'pending', 'error'.
          Example success: {'status': 'shipped', 'tracking_number': '1Z9...'}
          Example error: {'status': 'error', 'error_message': 'Order ID not found.'}
      """
      # ... function implementation to fetch status ...
      if status := fetch_status_from_backend(order_id):
           return {"status": status.state, "tracking_number": status.tracking} # Example structure
      else:
           return {"status": "error", "error_message": f"Order ID {order_id} not found."}

    ```

=== "Java"

    

* **Simplicity and Focus:**
    * **Keep Tools Focused:** Each tool should ideally perform one well-defined task.
    * **Fewer Parameters are Better:** Models generally handle tools with fewer, clearly defined parameters more reliably than those with many optional or complex ones.
    * **Use Simple Data Types:** Prefer basic types (`str`, `int`, `bool`, `float`, `List[str]`, in **Python**, or `int`, `byte`, `short`, `long`, `float`, `double`, `boolean` and `char` in **Java**) over complex custom classes or deeply nested structures as parameters when possible.
    * **Decompose Complex Tasks:** Break down functions that perform multiple distinct logical steps into smaller, more focused tools. For instance, instead of a single `update_user_profile(profile: ProfileObject)` tool, consider separate tools like `update_user_name(name: str)`, `update_user_address(address: str)`, `update_user_preferences(preferences: list[str])`, etc. This makes it easier for the LLM to select and use the correct capability.

By adhering to these guidelines, you provide the LLM with the clarity and structure it needs to effectively utilize your custom function tools, leading to more capable and reliable agent behavior.

## Toolsets: Grouping and Dynamically Providing Tools ![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/coming soon."}

Beyond individual tools, ADK introduces the concept of a **Toolset** via the `BaseToolset` interface (defined in `google.adk.tools.base_toolset`). A toolset allows you to manage and provide a collection of `BaseTool` instances, often dynamically, to an agent.

This approach is beneficial for:

*   **Organizing Related Tools:** Grouping tools that serve a common purpose (e.g., all tools for mathematical operations, or all tools interacting with a specific API).
*   **Dynamic Tool Availability:** Enabling an agent to have different tools available based on the current context (e.g., user permissions, session state, or other runtime conditions). The `get_tools` method of a toolset can decide which tools to expose.
*   **Integrating External Tool Providers:** Toolsets can act as adapters for tools coming from external systems, like an OpenAPI specification or an MCP server, converting them into ADK-compatible `BaseTool` objects.

### The `BaseToolset` Interface

Any class acting as a toolset in ADK should implement the `BaseToolset` abstract base class. This interface primarily defines two methods:

*   **`async def get_tools(...) -> list[BaseTool]:`**
    This is the core method of a toolset. When an ADK agent needs to know its available tools, it will call `get_tools()` on each `BaseToolset` instance provided in its `tools` list.
    *   It receives an optional `readonly_context` (an instance of `ReadonlyContext`). This context provides read-only access to information like the current session state (`readonly_context.state`), agent name, and invocation ID. The toolset can use this context to dynamically decide which tools to return.
    *   It **must** return a `list` of `BaseTool` instances (e.g., `FunctionTool`, `RestApiTool`).

*   **`async def close(self) -> None:`**
    This asynchronous method is called by the ADK framework when the toolset is no longer needed, for example, when an agent server is shutting down or the `Runner` is being closed. Implement this method to perform any necessary cleanup, such as closing network connections, releasing file handles, or cleaning up other resources managed by the toolset.

### Using Toolsets with Agents

You can include instances of your `BaseToolset` implementations directly in an `LlmAgent`'s `tools` list, alongside individual `BaseTool` instances.

When the agent initializes or needs to determine its available capabilities, the ADK framework will iterate through the `tools` list:

*   If an item is a `BaseTool` instance, it's used directly.
*   If an item is a `BaseToolset` instance, its `get_tools()` method is called (with the current `ReadonlyContext`), and the returned list of `BaseTool`s is added to the agent's available tools.

### Example: A Simple Math Toolset

Let's create a basic example of a toolset that provides simple arithmetic operations.

```py
# 1. Define the individual tool functions
def add_numbers(a: int, b: int, tool_context: ToolContext) -> Dict[str, Any]:
    """Adds two integer numbers.
    Args:
        a: The first number.
        b: The second number.
    Returns:
        A dictionary with the sum, e.g., {'status': 'success', 'result': 5}
    """
    print(f"Tool: add_numbers called with a={a}, b={b}")
    result = a + b
    # Example: Storing something in tool_context state
    tool_context.state["last_math_operation"] = "addition"
    return {"status": "success", "result": result}
def subtract_numbers(a: int, b: int) -> Dict[str, Any]:
    """Subtracts the second number from the first.
    Args:
        a: The first number.
        b: The second number.
    Returns:
        A dictionary with the difference, e.g., {'status': 'success', 'result': 1}
    """
    print(f"Tool: subtract_numbers called with a={a}, b={b}")
    return {"status": "success", "result": a - b}
# 2. Create the Toolset by implementing BaseToolset
class SimpleMathToolset(BaseToolset):
    def __init__(self, prefix: str = "math_"):
        self.prefix = prefix
        # Create FunctionTool instances once
        self._add_tool = FunctionTool(
            func=add_numbers,
            name=f"{self.prefix}add_numbers",  # Toolset can customize names
        )
        self._subtract_tool = FunctionTool(
            func=subtract_numbers, name=f"{self.prefix}subtract_numbers"
        )
        print(f"SimpleMathToolset initialized with prefix '{self.prefix}'")
    async def get_tools(
        self, readonly_context: Optional[ReadonlyContext] = None
    ) -> List[BaseTool]:
        print(f"SimpleMathToolset.get_tools() called.")
        # Example of dynamic behavior:
        # Could use readonly_context.state to decide which tools to return
        # For instance, if readonly_context.state.get("enable_advanced_math"):
        #    return [self._add_tool, self._subtract_tool, self._multiply_tool]
        # For this simple example, always return both tools
        tools_to_return = [self._add_tool, self._subtract_tool]
        print(f"SimpleMathToolset providing tools: {[t.name for t in tools_to_return]}")
        return tools_to_return
    async def close(self) -> None:
        # No resources to clean up in this simple example
        print(f"SimpleMathToolset.close() called for prefix '{self.prefix}'.")
        await asyncio.sleep(0)  # Placeholder for async cleanup if needed
# 3. Define an individual tool (not part of the toolset)
def greet_user(name: str = "User") -> Dict[str, str]:
    """Greets the user."""
    print(f"Tool: greet_user called with name={name}")
    return {"greeting": f"Hello, {name}!"}
greet_tool = FunctionTool(func=greet_user)
# 4. Instantiate the toolset
math_toolset_instance = SimpleMathToolset(prefix="calculator_")
# 5. Define an agent that uses both the individual tool and the toolset
calculator_agent = LlmAgent(
    name="CalculatorAgent",
    model="gemini-2.5-flash",  # Replace with your desired model
    instruction="You are a helpful calculator and greeter. "
    "Use 'greet_user' for greetings. "
    "Use 'calculator_add_numbers' to add and 'calculator_subtract_numbers' to subtract. "
    "Announce the state of 'last_math_operation' if it's set.",
    tools=[greet_tool, math_toolset_instance],  # Individual tool  # Toolset instance
)
```

In this example:

*   `SimpleMathToolset` implements `BaseToolset` and its `get_tools()` method returns `FunctionTool` instances for `add_numbers` and `subtract_numbers`. It also customizes their names using a prefix.
*   The `calculator_agent` is configured with both an individual `greet_tool` and an instance of `SimpleMathToolset`.
*   When `calculator_agent` is run, ADK will call `math_toolset_instance.get_tools()`. The agent's LLM will then have access to `greet_user`, `calculator_add_numbers`, and `calculator_subtract_numbers` to handle user requests.
*   The `add_numbers` tool demonstrates writing to `tool_context.state`, and the agent's instruction mentions reading this state.
*   The `close()` method is called to ensure any resources held by the toolset are released.

Toolsets offer a powerful way to organize, manage, and dynamically provide collections of tools to your ADK agents, leading to more modular, maintainable, and adaptable agentic applications.


# Model Context Protocol Tools

 This guide walks you through two ways of integrating Model Context Protocol (MCP) with ADK.

## What is Model Context Protocol (MCP)?

The Model Context Protocol (MCP) is an open standard designed to standardize how Large Language Models (LLMs) like Gemini and Claude communicate with external applications, data sources, and tools. Think of it as a universal connection mechanism that simplifies how LLMs obtain context, execute actions, and interact with various systems.

MCP follows a client-server architecture, defining how **data** (resources), **interactive templates** (prompts), and **actionable functions** (tools) are exposed by an **MCP server** and consumed by an **MCP client** (which could be an LLM host application or an AI agent).

This guide covers two primary integration patterns:

1. **Using Existing MCP Servers within ADK:** An ADK agent acts as an MCP client, leveraging tools provided by external MCP servers.
2. **Exposing ADK Tools via an MCP Server:** Building an MCP server that wraps ADK tools, making them accessible to any MCP client.

## Prerequisites

Before you begin, ensure you have the following set up:

* **Set up ADK:** Follow the standard ADK [setup instructions](../get-started/quickstart.md/#venv-install) in the quickstart.
* **Install/update Python/Java:** MCP requires Python version of 3.10 or higher for Python or Java 17+.
* **Setup Node.js and npx:** **(Python only)** Many community MCP servers are distributed as Node.js packages and run using `npx`. Install Node.js (which includes npx) if you haven't already. For details, see [https://nodejs.org/en](https://nodejs.org/en).
* **Verify Installations:** **(Python only)** Confirm `adk` and `npx` are in your PATH within the activated virtual environment:

```shell
# Both commands should print the path to the executables.
which adk
which npx
```

## 1. Using MCP servers with ADK agents (ADK as an MCP client) in `adk web`

This section demonstrates how to integrate tools from external MCP (Model Context Protocol) servers into your ADK agents. This is the **most common** integration pattern when your ADK agent needs to use capabilities provided by an existing service that exposes an MCP interface. You will see how the `MCPToolset` class can be directly added to your agent's `tools` list, enabling seamless connection to an MCP server, discovery of its tools, and making them available for your agent to use. These examples primarily focus on interactions within the `adk web` development environment.

### `MCPToolset` class

The `MCPToolset` class is ADK's primary mechanism for integrating tools from an MCP server. When you include an `MCPToolset` instance in your agent's `tools` list, it automatically handles the interaction with the specified MCP server. Here's how it works:

1.  **Connection Management:** On initialization, `MCPToolset` establishes and manages the connection to the MCP server. This can be a local server process (using `StdioServerParameters` for communication over standard input/output) or a remote server (using `SseServerParams` for Server-Sent Events). The toolset also handles the graceful shutdown of this connection when the agent or application terminates.
2.  **Tool Discovery & Adaptation:** Once connected, `MCPToolset` queries the MCP server for its available tools (via the `list_tools` MCP method). It then converts the schemas of these discovered MCP tools into ADK-compatible `BaseTool` instances.
3.  **Exposure to Agent:** These adapted tools are then made available to your `LlmAgent` as if they were native ADK tools.
4.  **Proxying Tool Calls:** When your `LlmAgent` decides to use one of these tools, `MCPToolset` transparently proxies the call (using the `call_tool` MCP method) to the MCP server, sends the necessary arguments, and returns the server's response back to the agent.
5.  **Filtering (Optional):** You can use the `tool_filter` parameter when creating an `MCPToolset` to select a specific subset of tools from the MCP server, rather than exposing all of them to your agent.

The following examples demonstrate how to use `MCPToolset` within the `adk web` development environment. For scenarios where you need more fine-grained control over the MCP connection lifecycle or are not using `adk web`, refer to the "Using MCP Tools in your own Agent out of `adk web`" section later in this page.

### Example 1: File System MCP Server

This example demonstrates connecting to a local MCP server that provides file system operations.

#### Step 1: Define your Agent with `MCPToolset`

Create an `agent.py` file (e.g., in `./adk_agent_samples/mcp_agent/agent.py`). The `MCPToolset` is instantiated directly within the `tools` list of your `LlmAgent`.

*   **Important:** Replace `"/path/to/your/folder"` in the `args` list with the **absolute path** to an actual folder on your local system that the MCP server can access.
*   **Important:** Place the `.env` file in the parent directory of the `./adk_agent_samples` directory.

```python
# ./adk_agent_samples/mcp_agent/agent.py
import os # Required for path operations
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioServerParameters

# It's good practice to define paths dynamically if possible,
# or ensure the user understands the need for an ABSOLUTE path.
# For this example, we'll construct a path relative to this file,
# assuming '/path/to/your/folder' is in the same directory as agent.py.
# REPLACE THIS with an actual absolute path if needed for your setup.
TARGET_FOLDER_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "/path/to/your/folder")
# Ensure TARGET_FOLDER_PATH is an absolute path for the MCP server.
# If you created ./adk_agent_samples/mcp_agent/your_folder,

root_agent = LlmAgent(
    model='gemini-2.5-flash',
    name='filesystem_assistant_agent',
    instruction='Help the user manage their files. You can list files, read files, etc.',
    tools=[
        MCPToolset(
            connection_params=StdioServerParameters(
                command='npx',
                args=[
                    "-y",  # Argument for npx to auto-confirm install
                    "@modelcontextprotocol/server-filesystem",
                    # IMPORTANT: This MUST be an ABSOLUTE path to a folder the
                    # npx process can access.
                    # Replace with a valid absolute path on your system.
                    # For example: "/Users/youruser/accessible_mcp_files"
                    # or use a dynamically constructed absolute path:
                    os.path.abspath(TARGET_FOLDER_PATH),
                ],
            ),
            # Optional: Filter which tools from the MCP server are exposed
            # tool_filter=['list_directory', 'read_file']
        )
    ],
)
```


#### Step 2: Create an `__init__.py` file

Ensure you have an `__init__.py` in the same directory as `agent.py` to make it a discoverable Python package for ADK.

```python
# ./adk_agent_samples/mcp_agent/__init__.py
from . import agent
```

#### Step 3: Run `adk web` and Interact

Navigate to the parent directory of `mcp_agent` (e.g., `adk_agent_samples`) in your terminal and run:

```shell
cd ./adk_agent_samples # Or your equivalent parent directory
adk web
```

!!!info "Note for Windows users"

    When hitting the `_make_subprocess_transport NotImplementedError`, consider using `adk web --no-reload` instead.


Once the ADK Web UI loads in your browser:

1.  Select the `filesystem_assistant_agent` from the agent dropdown.
2.  Try prompts like:
    *   "List files in the current directory."
    *   "Can you read the file named sample.txt?" (assuming you created it in `TARGET_FOLDER_PATH`).
    *   "What is the content of `another_file.md`?"

You should see the agent interacting with the MCP file system server, and the server's responses (file listings, file content) relayed through the agent. The `adk web` console (terminal where you ran the command) might also show logs from the `npx` process if it outputs to stderr.

<img src="../../assets/adk-tool-mcp-filesystem-adk-web-demo.png" alt="MCP with ADK Web - FileSystem Example">


### Example 2: Google Maps MCP Server

This example demonstrates connecting to the Google Maps MCP server.

#### Step 1: Get API Key and Enable APIs

1.  **Google Maps API Key:** Follow the directions at [Use API keys](https://developers.google.com/maps/documentation/javascript/get-api-key#create-api-keys) to obtain a Google Maps API Key.
2.  **Enable APIs:** In your Google Cloud project, ensure the following APIs are enabled:
    *   Directions API
    *   Routes API
    For instructions, see the [Getting started with Google Maps Platform](https://developers.google.com/maps/get-started#enable-api-sdk) documentation.

#### Step 2: Define your Agent with `MCPToolset` for Google Maps

Modify your `agent.py` file (e.g., in `./adk_agent_samples/mcp_agent/agent.py`). Replace `YOUR_GOOGLE_MAPS_API_KEY` with the actual API key you obtained.

```python
# ./adk_agent_samples/mcp_agent/agent.py
import os
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioServerParameters

# Retrieve the API key from an environment variable or directly insert it.
# Using an environment variable is generally safer.
# Ensure this environment variable is set in the terminal where you run 'adk web'.
# Example: export GOOGLE_MAPS_API_KEY="YOUR_ACTUAL_KEY"
google_maps_api_key = os.environ.get("GOOGLE_MAPS_API_KEY")

if not google_maps_api_key:
    # Fallback or direct assignment for testing - NOT RECOMMENDED FOR PRODUCTION
    google_maps_api_key = "YOUR_GOOGLE_MAPS_API_KEY_HERE" # Replace if not using env var
    if google_maps_api_key == "YOUR_GOOGLE_MAPS_API_KEY_HERE":
        print("WARNING: GOOGLE_MAPS_API_KEY is not set. Please set it as an environment variable or in the script.")
        # You might want to raise an error or exit if the key is crucial and not found.

root_agent = LlmAgent(
    model='gemini-2.5-flash',
    name='maps_assistant_agent',
    instruction='Help the user with mapping, directions, and finding places using Google Maps tools.',
    tools=[
        MCPToolset(
            connection_params=StdioServerParameters(
                command='npx',
                args=[
                    "-y",
                    "@modelcontextprotocol/server-google-maps",
                ],
                # Pass the API key as an environment variable to the npx process
                # This is how the MCP server for Google Maps expects the key.
                env={
                    "GOOGLE_MAPS_API_KEY": google_maps_api_key
                }
            ),
            # You can filter for specific Maps tools if needed:
            # tool_filter=['get_directions', 'find_place_by_id']
        )
    ],
)
```

#### Step 3: Ensure `__init__.py` Exists

If you created this in Example 1, you can skip this. Otherwise, ensure you have an `__init__.py` in the `./adk_agent_samples/mcp_agent/` directory:

```python
# ./adk_agent_samples/mcp_agent/__init__.py
from . import agent
```

#### Step 4: Run `adk web` and Interact

1.  **Set Environment Variable (Recommended):**
    Before running `adk web`, it's best to set your Google Maps API key as an environment variable in your terminal:
    ```shell
    export GOOGLE_MAPS_API_KEY="YOUR_ACTUAL_GOOGLE_MAPS_API_KEY"
    ```
    Replace `YOUR_ACTUAL_GOOGLE_MAPS_API_KEY` with your key.

2.  **Run `adk web`**:
    Navigate to the parent directory of `mcp_agent` (e.g., `adk_agent_samples`) and run:
    ```shell
    cd ./adk_agent_samples # Or your equivalent parent directory
    adk web
    ```

3.  **Interact in the UI**:
    *   Select the `maps_assistant_agent`.
    *   Try prompts like:
        *   "Get directions from GooglePlex to SFO."
        *   "Find coffee shops near Golden Gate Park."
        *   "What's the route from Paris, France to Berlin, Germany?"

You should see the agent use the Google Maps MCP tools to provide directions or location-based information.

<img src="../../assets/adk-tool-mcp-maps-adk-web-demo.png" alt="MCP with ADK Web - Google Maps Example">


## 2. Building an MCP server with ADK tools (MCP server exposing ADK)

This pattern allows you to wrap existing ADK tools and make them available to any standard MCP client application. The example in this section exposes the ADK `load_web_page` tool through a custom-built MCP server.

### Summary of steps

You will create a standard Python MCP server application using the `mcp` library. Within this server, you will:

1.  Instantiate the ADK tool(s) you want to expose (e.g., `FunctionTool(load_web_page)`).
2.  Implement the MCP server's `@app.list_tools()` handler to advertise the ADK tool(s). This involves converting the ADK tool definition to the MCP schema using the `adk_to_mcp_tool_type` utility from `google.adk.tools.mcp_tool.conversion_utils`.
3.  Implement the MCP server's `@app.call_tool()` handler. This handler will:
    *   Receive tool call requests from MCP clients.
    *   Identify if the request targets one of your wrapped ADK tools.
    *   Execute the ADK tool's `.run_async()` method.
    *   Format the ADK tool's result into an MCP-compliant response (e.g., `mcp.types.TextContent`).

### Prerequisites

Install the MCP server library in the same Python environment as your ADK installation:

```shell
pip install mcp
```

### Step 1: Create the MCP Server Script

Create a new Python file for your MCP server, for example, `my_adk_mcp_server.py`.

### Step 2: Implement the Server Logic

Add the following code to `my_adk_mcp_server.py`. This script sets up an MCP server that exposes the ADK `load_web_page` tool.

```python
# my_adk_mcp_server.py
import asyncio
import json
import os
from dotenv import load_dotenv

# MCP Server Imports
from mcp import types as mcp_types # Use alias to avoid conflict
from mcp.server.lowlevel import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio # For running as a stdio server

# ADK Tool Imports
from google.adk.tools.function_tool import FunctionTool
from google.adk.tools.load_web_page import load_web_page # Example ADK tool
# ADK <-> MCP Conversion Utility
from google.adk.tools.mcp_tool.conversion_utils import adk_to_mcp_tool_type

# --- Load Environment Variables (If ADK tools need them, e.g., API keys) ---
load_dotenv() # Create a .env file in the same directory if needed

# --- Prepare the ADK Tool ---
# Instantiate the ADK tool you want to expose.
# This tool will be wrapped and called by the MCP server.
print("Initializing ADK load_web_page tool...")
adk_tool_to_expose = FunctionTool(load_web_page)
print(f"ADK tool '{adk_tool_to_expose.name}' initialized and ready to be exposed via MCP.")
# --- End ADK Tool Prep ---

# --- MCP Server Setup ---
print("Creating MCP Server instance...")
# Create a named MCP Server instance using the mcp.server library
app = Server("adk-tool-exposing-mcp-server")

# Implement the MCP server's handler to list available tools
@app.list_tools()
async def list_mcp_tools() -> list[mcp_types.Tool]:
    """MCP handler to list tools this server exposes."""
    print("MCP Server: Received list_tools request.")
    # Convert the ADK tool's definition to the MCP Tool schema format
    mcp_tool_schema = adk_to_mcp_tool_type(adk_tool_to_expose)
    print(f"MCP Server: Advertising tool: {mcp_tool_schema.name}")
    return [mcp_tool_schema]

# Implement the MCP server's handler to execute a tool call
@app.call_tool()
async def call_mcp_tool(
    name: str, arguments: dict
) -> list[mcp_types.Content]: # MCP uses mcp_types.Content
    """MCP handler to execute a tool call requested by an MCP client."""
    print(f"MCP Server: Received call_tool request for '{name}' with args: {arguments}")

    # Check if the requested tool name matches our wrapped ADK tool
    if name == adk_tool_to_expose.name:
        try:
            # Execute the ADK tool's run_async method.
            # Note: tool_context is None here because this MCP server is
            # running the ADK tool outside of a full ADK Runner invocation.
            # If the ADK tool requires ToolContext features (like state or auth),
            # this direct invocation might need more sophisticated handling.
            adk_tool_response = await adk_tool_to_expose.run_async(
                args=arguments,
                tool_context=None,
            )
            print(f"MCP Server: ADK tool '{name}' executed. Response: {adk_tool_response}")

            # Format the ADK tool's response (often a dict) into an MCP-compliant format.
            # Here, we serialize the response dictionary as a JSON string within TextContent.
            # Adjust formatting based on the ADK tool's output and client needs.
            response_text = json.dumps(adk_tool_response, indent=2)
            # MCP expects a list of mcp_types.Content parts
            return [mcp_types.TextContent(type="text", text=response_text)]

        except Exception as e:
            print(f"MCP Server: Error executing ADK tool '{name}': {e}")
            # Return an error message in MCP format
            error_text = json.dumps({"error": f"Failed to execute tool '{name}': {str(e)}"})
            return [mcp_types.TextContent(type="text", text=error_text)]
    else:
        # Handle calls to unknown tools
        print(f"MCP Server: Tool '{name}' not found/exposed by this server.")
        error_text = json.dumps({"error": f"Tool '{name}' not implemented by this server."})
        return [mcp_types.TextContent(type="text", text=error_text)]

# --- MCP Server Runner ---
async def run_mcp_stdio_server():
    """Runs the MCP server, listening for connections over standard input/output."""
    # Use the stdio_server context manager from the mcp.server.stdio library
    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        print("MCP Stdio Server: Starting handshake with client...")
        await app.run(
            read_stream,
            write_stream,
            InitializationOptions(
                server_name=app.name, # Use the server name defined above
                server_version="0.1.0",
                capabilities=app.get_capabilities(
                    # Define server capabilities - consult MCP docs for options
                    notification_options=NotificationOptions(),
                    experimental_capabilities={},
                ),
            ),
        )
        print("MCP Stdio Server: Run loop finished or client disconnected.")

if __name__ == "__main__":
    print("Launching MCP Server to expose ADK tools via stdio...")
    try:
        asyncio.run(run_mcp_stdio_server())
    except KeyboardInterrupt:
        print("\nMCP Server (stdio) stopped by user.")
    except Exception as e:
        print(f"MCP Server (stdio) encountered an error: {e}")
    finally:
        print("MCP Server (stdio) process exiting.")
# --- End MCP Server ---
```

### Step 3: Test your Custom MCP Server with an ADK Agent

Now, create an ADK agent that will act as a client to the MCP server you just built. This ADK agent will use `MCPToolset` to connect to your `my_adk_mcp_server.py` script.

Create an `agent.py` (e.g., in `./adk_agent_samples/mcp_client_agent/agent.py`):

```python
# ./adk_agent_samples/mcp_client_agent/agent.py
import os
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioServerParameters

# IMPORTANT: Replace this with the ABSOLUTE path to your my_adk_mcp_server.py script
PATH_TO_YOUR_MCP_SERVER_SCRIPT = "/path/to/your/my_adk_mcp_server.py" # <<< REPLACE

if PATH_TO_YOUR_MCP_SERVER_SCRIPT == "/path/to/your/my_adk_mcp_server.py":
    print("WARNING: PATH_TO_YOUR_MCP_SERVER_SCRIPT is not set. Please update it in agent.py.")
    # Optionally, raise an error if the path is critical

root_agent = LlmAgent(
    model='gemini-2.5-flash',
    name='web_reader_mcp_client_agent',
    instruction="Use the 'load_web_page' tool to fetch content from a URL provided by the user.",
    tools=[
        MCPToolset(
            connection_params=StdioServerParameters(
                command='python3', # Command to run your MCP server script
                args=[PATH_TO_YOUR_MCP_SERVER_SCRIPT], # Argument is the path to the script
            )
            # tool_filter=['load_web_page'] # Optional: ensure only specific tools are loaded
        )
    ],
)
```

And an `__init__.py` in the same directory:
```python
# ./adk_agent_samples/mcp_client_agent/__init__.py
from . import agent
```

**To run the test:**

1.  **Start your custom MCP server (optional, for separate observation):**
    You can run your `my_adk_mcp_server.py` directly in one terminal to see its logs:
    ```shell
    python3 /path/to/your/my_adk_mcp_server.py
    ```
    It will print "Launching MCP Server..." and wait. The ADK agent (run via `adk web`) will then connect to this process if the `command` in `StdioServerParameters` is set up to execute it.
    *(Alternatively, `MCPToolset` will start this server script as a subprocess automatically when the agent initializes).*

2.  **Run `adk web` for the client agent:**
    Navigate to the parent directory of `mcp_client_agent` (e.g., `adk_agent_samples`) and run:
    ```shell
    cd ./adk_agent_samples # Or your equivalent parent directory
    adk web
    ```

3.  **Interact in the ADK Web UI:**
    *   Select the `web_reader_mcp_client_agent`.
    *   Try a prompt like: "Load the content from https://example.com"

The ADK agent (`web_reader_mcp_client_agent`) will use `MCPToolset` to start and connect to your `my_adk_mcp_server.py`. Your MCP server will receive the `call_tool` request, execute the ADK `load_web_page` tool, and return the result. The ADK agent will then relay this information. You should see logs from both the ADK Web UI (and its terminal) and potentially from your `my_adk_mcp_server.py` terminal if you ran it separately.

This example demonstrates how ADK tools can be encapsulated within an MCP server, making them accessible to a broader range of MCP-compliant clients, not just ADK agents.

Refer to the [documentation](https://modelcontextprotocol.io/quickstart/server#core-mcp-concepts), to try it out with Claude Desktop.

## Using MCP Tools in your own Agent out of `adk web`

This section is relevant to you if:

* You are developing your own Agent using ADK
* And, you are **NOT** using `adk web`,
* And, you are exposing the agent via your own UI


Using MCP Tools requires a different setup than using regular tools, due to the fact that specs for MCP Tools are fetched asynchronously
from the MCP Server running remotely, or in another process.

The following example is modified from the "Example 1: File System MCP Server" example above. The main differences are:

1. Your tool and agent are created asynchronously
2. You need to properly manage the exit stack, so that your agents and tools are destructed properly when the connection to MCP Server is closed.

```python
# agent.py (modify get_tools_async and other parts as needed)
# ./adk_agent_samples/mcp_agent/agent.py
import os
import asyncio
from dotenv import load_dotenv
from google.genai import types
from google.adk.agents.llm_agent import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts.in_memory_artifact_service import InMemoryArtifactService # Optional
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams, StdioServerParameters

# Load environment variables from .env file in the parent directory
# Place this near the top, before using env vars like API keys
load_dotenv('../.env')

# Ensure TARGET_FOLDER_PATH is an absolute path for the MCP server.
TARGET_FOLDER_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "/path/to/your/folder")

# --- Step 1: Agent Definition ---
async def get_agent_async():
  """Creates an ADK Agent equipped with tools from the MCP Server."""
  toolset = MCPToolset(
      # Use StdioServerParameters for local process communication
      connection_params=StdioServerParameters(
          command='npx', # Command to run the server
          args=["-y",    # Arguments for the command
                "@modelcontextprotocol/server-filesystem",
                TARGET_FOLDER_PATH],
      ),
      tool_filter=['read_file', 'list_directory'] # Optional: filter specific tools
      # For remote servers, you would use SseServerParams instead:
      # connection_params=SseServerParams(url="http://remote-server:port/path", headers={...})
  )

  # Use in an agent
  root_agent = LlmAgent(
      model='gemini-2.5-flash', # Adjust model name if needed based on availability
      name='enterprise_assistant',
      instruction='Help user accessing their file systems',
      tools=[toolset], # Provide the MCP tools to the ADK agent
  )
  return root_agent, toolset

# --- Step 2: Main Execution Logic ---
async def async_main():
  session_service = InMemorySessionService()
  # Artifact service might not be needed for this example
  artifacts_service = InMemoryArtifactService()

  session = await session_service.create_session(
      state={}, app_name='mcp_filesystem_app', user_id='user_fs'
  )

  # TODO: Change the query to be relevant to YOUR specified folder.
  # e.g., "list files in the 'documents' subfolder" or "read the file 'notes.txt'"
  query = "list files in the tests folder"
  print(f"User Query: '{query}'")
  content = types.Content(role='user', parts=[types.Part(text=query)])

  root_agent, toolset = await get_agent_async()

  runner = Runner(
      app_name='mcp_filesystem_app',
      agent=root_agent,
      artifact_service=artifacts_service, # Optional
      session_service=session_service,
  )

  print("Running agent...")
  events_async = runner.run_async(
      session_id=session.id, user_id=session.user_id, new_message=content
  )

  async for event in events_async:
    print(f"Event received: {event}")

  # Cleanup is handled automatically by the agent framework
  # But you can also manually close if needed:
  print("Closing MCP server connection...")
  await toolset.close()
  print("Cleanup complete.")

if __name__ == '__main__':
  try:
    asyncio.run(async_main())
  except Exception as e:
    print(f"An error occurred: {e}")
```


## Key considerations

When working with MCP and ADK, keep these points in mind:

* **Protocol vs. Library:** MCP is a protocol specification, defining communication rules. ADK is a Python library/framework for building agents. MCPToolset bridges these by implementing the client side of the MCP protocol within the ADK framework. Conversely, building an MCP server in Python requires using the model-context-protocol library.

* **ADK Tools vs. MCP Tools:**

    * ADK Tools (BaseTool, FunctionTool, AgentTool, etc.) are Python objects designed for direct use within the ADK's LlmAgent and Runner.
    * MCP Tools are capabilities exposed by an MCP Server according to the protocol's schema. MCPToolset makes these look like ADK tools to an LlmAgent.
    * Langchain/CrewAI Tools are specific implementations within those libraries, often simple functions or classes, lacking the server/protocol structure of MCP. ADK offers wrappers (LangchainTool, CrewaiTool) for some interoperability.

* **Asynchronous nature:** Both ADK and the MCP Python library are heavily based on the asyncio Python library. Tool implementations and server handlers should generally be async functions.

* **Stateful sessions (MCP):** MCP establishes stateful, persistent connections between a client and server instance. This differs from typical stateless REST APIs.

    * **Deployment:** This statefulness can pose challenges for scaling and deployment, especially for remote servers handling many users. The original MCP design often assumed client and server were co-located. Managing these persistent connections requires careful infrastructure considerations (e.g., load balancing, session affinity).
    * **ADK MCPToolset:** Manages this connection lifecycle. The exit\_stack pattern shown in the examples is crucial for ensuring the connection (and potentially the server process) is properly terminated when the ADK agent finishes.

## Further Resources

* [Model Context Protocol Documentation](https://modelcontextprotocol.io/ )
* [MCP Specification](https://modelcontextprotocol.io/specification/)
* [MCP Python SDK & Examples](https://github.com/modelcontextprotocol/)


# OpenAPI Integration

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

## Integrating REST APIs with OpenAPI

ADK simplifies interacting with external REST APIs by automatically generating callable tools directly from an [OpenAPI Specification (v3.x)](https://swagger.io/specification/). This eliminates the need to manually define individual function tools for each API endpoint.

!!! tip "Core Benefit"
    Use `OpenAPIToolset` to instantly create agent tools (`RestApiTool`) from your existing API documentation (OpenAPI spec), enabling agents to seamlessly call your web services.

## Key Components

* **`OpenAPIToolset`**: This is the primary class you'll use. You initialize it with your OpenAPI specification, and it handles the parsing and generation of tools.
* **`RestApiTool`**: This class represents a single, callable API operation (like `GET /pets/{petId}` or `POST /pets`). `OpenAPIToolset` creates one `RestApiTool` instance for each operation defined in your spec.

## How it Works

The process involves these main steps when you use `OpenAPIToolset`:

1. **Initialization & Parsing**:
    * You provide the OpenAPI specification to `OpenAPIToolset` either as a Python dictionary, a JSON string, or a YAML string.
    * The toolset internally parses the spec, resolving any internal references (`$ref`) to understand the complete API structure.

2. **Operation Discovery**:
    * It identifies all valid API operations (e.g., `GET`, `POST`, `PUT`, `DELETE`) defined within the `paths` object of your specification.

3. **Tool Generation**:
    * For each discovered operation, `OpenAPIToolset` automatically creates a corresponding `RestApiTool` instance.
    * **Tool Name**: Derived from the `operationId` in the spec (converted to `snake_case`, max 60 chars). If `operationId` is missing, a name is generated from the method and path.
    * **Tool Description**: Uses the `summary` or `description` from the operation for the LLM.
    * **API Details**: Stores the required HTTP method, path, server base URL, parameters (path, query, header, cookie), and request body schema internally.

4. **`RestApiTool` Functionality**: Each generated `RestApiTool`:
    * **Schema Generation**: Dynamically creates a `FunctionDeclaration` based on the operation's parameters and request body. This schema tells the LLM how to call the tool (what arguments are expected).
    * **Execution**: When called by the LLM, it constructs the correct HTTP request (URL, headers, query params, body) using the arguments provided by the LLM and the details from the OpenAPI spec. It handles authentication (if configured) and executes the API call using the `requests` library.
    * **Response Handling**: Returns the API response (typically JSON) back to the agent flow.

5. **Authentication**: You can configure global authentication (like API keys or OAuth - see [Authentication](../tools/authentication.md) for details) when initializing `OpenAPIToolset`. This authentication configuration is automatically applied to all generated `RestApiTool` instances.

## Usage Workflow

Follow these steps to integrate an OpenAPI spec into your agent:

1. **Obtain Spec**: Get your OpenAPI specification document (e.g., load from a `.json` or `.yaml` file, fetch from a URL).
2. **Instantiate Toolset**: Create an `OpenAPIToolset` instance, passing the spec content and type (`spec_str`/`spec_dict`, `spec_str_type`). Provide authentication details (`auth_scheme`, `auth_credential`) if required by the API.

    ```python
    from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

    # Example with a JSON string
    openapi_spec_json = '...' # Your OpenAPI JSON string
    toolset = OpenAPIToolset(spec_str=openapi_spec_json, spec_str_type="json")

    # Example with a dictionary
    # openapi_spec_dict = {...} # Your OpenAPI spec as a dict
    # toolset = OpenAPIToolset(spec_dict=openapi_spec_dict)
    ```

3. **Add to Agent**: Include the retrieved tools in your `LlmAgent`'s `tools` list.

    ```python
    from google.adk.agents import LlmAgent

    my_agent = LlmAgent(
        name="api_interacting_agent",
        model="gemini-2.5-flash", # Or your preferred model
        tools=[toolset], # Pass the toolset
        # ... other agent config ...
    )
    ```

4. **Instruct Agent**: Update your agent's instructions to inform it about the new API capabilities and the names of the tools it can use (e.g., `list_pets`, `create_pet`). The tool descriptions generated from the spec will also help the LLM.
5. **Run Agent**: Execute your agent using the `Runner`. When the LLM determines it needs to call one of the APIs, it will generate a function call targeting the appropriate `RestApiTool`, which will then handle the HTTP request automatically.

## Example

This example demonstrates generating tools from a simple Pet Store OpenAPI spec (using `httpbin.org` for mock responses) and interacting with them via an agent.

???+ "Code: Pet Store API"

    ```python title="openapi_example.py"
    # Copyright 2026 Google LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    import asyncio
    import uuid # For unique session IDs
    from dotenv import load_dotenv

    from google.adk.agents import LlmAgent
    from google.adk.runners import Runner
    from google.adk.sessions import InMemorySessionService
    from google.genai import types

    # --- OpenAPI Tool Imports ---
    from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import OpenAPIToolset

    # --- Load Environment Variables (If ADK tools need them, e.g., API keys) ---
    load_dotenv() # Create a .env file in the same directory if needed

    # --- Constants ---
    APP_NAME_OPENAPI = "openapi_petstore_app"
    USER_ID_OPENAPI = "user_openapi_1"
    SESSION_ID_OPENAPI = f"session_openapi_{uuid.uuid4()}" # Unique session ID
    AGENT_NAME_OPENAPI = "petstore_manager_agent"
    GEMINI_MODEL = "gemini-2.5-flash"

    # --- Sample OpenAPI Specification (JSON String) ---
    # A basic Pet Store API example using httpbin.org as a mock server
    openapi_spec_string = """
    {
      "openapi": "3.0.0",
      "info": {
        "title": "Simple Pet Store API (Mock)",
        "version": "1.0.1",
        "description": "An API to manage pets in a store, using httpbin for responses."
      },
      "servers": [
        {
          "url": "https://httpbin.org",
          "description": "Mock server (httpbin.org)"
        }
      ],
      "paths": {
        "/get": {
          "get": {
            "summary": "List all pets (Simulated)",
            "operationId": "listPets",
            "description": "Simulates returning a list of pets. Uses httpbin's /get endpoint which echoes query parameters.",
            "parameters": [
              {
                "name": "limit",
                "in": "query",
                "description": "Maximum number of pets to return",
                "required": false,
                "schema": { "type": "integer", "format": "int32" }
              },
              {
                 "name": "status",
                 "in": "query",
                 "description": "Filter pets by status",
                 "required": false,
                 "schema": { "type": "string", "enum": ["available", "pending", "sold"] }
              }
            ],
            "responses": {
              "200": {
                "description": "A list of pets (echoed query params).",
                "content": { "application/json": { "schema": { "type": "object" } } }
              }
            }
          }
        },
        "/post": {
          "post": {
            "summary": "Create a pet (Simulated)",
            "operationId": "createPet",
            "description": "Simulates adding a new pet. Uses httpbin's /post endpoint which echoes the request body.",
            "requestBody": {
              "description": "Pet object to add",
              "required": true,
              "content": {
                "application/json": {
                  "schema": {
                    "type": "object",
                    "required": ["name"],
                    "properties": {
                      "name": {"type": "string", "description": "Name of the pet"},
                      "tag": {"type": "string", "description": "Optional tag for the pet"}
                    }
                  }
                }
              }
            },
            "responses": {
              "201": {
                "description": "Pet created successfully (echoed request body).",
                "content": { "application/json": { "schema": { "type": "object" } } }
              }
            }
          }
        },
        "/get?petId={petId}": {
          "get": {
            "summary": "Info for a specific pet (Simulated)",
            "operationId": "showPetById",
            "description": "Simulates returning info for a pet ID. Uses httpbin's /get endpoint.",
            "parameters": [
              {
                "name": "petId",
                "in": "path",
                "description": "This is actually passed as a query param to httpbin /get",
                "required": true,
                "schema": { "type": "integer", "format": "int64" }
              }
            ],
            "responses": {
              "200": {
                "description": "Information about the pet (echoed query params)",
                "content": { "application/json": { "schema": { "type": "object" } } }
              },
              "404": { "description": "Pet not found (simulated)" }
            }
          }
        }
      }
    }
    """

    # --- Create OpenAPIToolset ---
    petstore_toolset = OpenAPIToolset(
        spec_str=openapi_spec_string,
        spec_str_type='json',
        # No authentication needed for httpbin.org
    )

    # --- Agent Definition ---
    root_agent = LlmAgent(
        name=AGENT_NAME_OPENAPI,
        model=GEMINI_MODEL,
        tools=[petstore_toolset], # Pass the list of RestApiTool objects
        instruction="""You are a Pet Store assistant managing pets via an API.
        Use the available tools to fulfill user requests.
        When creating a pet, confirm the details echoed back by the API.
        When listing pets, mention any filters used (like limit or status).
        When showing a pet by ID, state the ID you requested.
        """,
        description="Manages a Pet Store using tools generated from an OpenAPI spec."
    )

    # --- Session and Runner Setup ---
    async def setup_session_and_runner():
        session_service_openapi = InMemorySessionService()
        runner_openapi = Runner(
            agent=root_agent,
            app_name=APP_NAME_OPENAPI,
            session_service=session_service_openapi,
        )
        await session_service_openapi.create_session(
            app_name=APP_NAME_OPENAPI,
            user_id=USER_ID_OPENAPI,
            session_id=SESSION_ID_OPENAPI,
        )
        return runner_openapi

    # --- Agent Interaction Function ---
    async def call_openapi_agent_async(query, runner_openapi):
        print("\n--- Running OpenAPI Pet Store Agent ---")
        print(f"Query: {query}")

        content = types.Content(role='user', parts=[types.Part(text=query)])
        final_response_text = "Agent did not provide a final text response."
        try:
            async for event in runner_openapi.run_async(
                user_id=USER_ID_OPENAPI, session_id=SESSION_ID_OPENAPI, new_message=content
                ):
                # Optional: Detailed event logging for debugging
                # print(f"  DEBUG Event: Author={event.author}, Type={'Final' if event.is_final_response() else 'Intermediate'}, Content={str(event.content)[:100]}...")
                if event.get_function_calls():
                    call = event.get_function_calls()[0]
                    print(f"  Agent Action: Called function '{call.name}' with args {call.args}")
                elif event.get_function_responses():
                    response = event.get_function_responses()[0]
                    print(f"  Agent Action: Received response for '{response.name}'")
                    # print(f"  Tool Response Snippet: {str(response.response)[:200]}...") # Uncomment for response details
                elif event.is_final_response() and event.content and event.content.parts:
                    # Capture the last final text response
                    final_response_text = event.content.parts[0].text.strip()

            print(f"Agent Final Response: {final_response_text}")

        except Exception as e:
            print(f"An error occurred during agent run: {e}")
            import traceback
            traceback.print_exc() # Print full traceback for errors
        print("-" * 30)

    # --- Run Examples ---
    async def run_openapi_example():
        runner_openapi = await setup_session_and_runner()

        # Trigger listPets
        await call_openapi_agent_async("Show me the pets available.", runner_openapi)
        # Trigger createPet
        await call_openapi_agent_async("Please add a new dog named 'Dukey'.", runner_openapi)
        # Trigger showPetById
        await call_openapi_agent_async("Get info for pet with ID 123.", runner_openapi)

    # --- Execute ---
    if __name__ == "__main__":
        print("Executing OpenAPI example...")
        # Use asyncio.run() for top-level execution
        try:
            asyncio.run(run_openapi_example())
        except RuntimeError as e:
            if "cannot be called from a running event loop" in str(e):
                print("Info: Cannot run asyncio.run from a running event loop (e.g., Jupyter/Colab).")
                # If in Jupyter/Colab, you might need to run like this:
                # await run_openapi_example()
            else:
                raise e
        print("OpenAPI example finished.")
    
    ```


