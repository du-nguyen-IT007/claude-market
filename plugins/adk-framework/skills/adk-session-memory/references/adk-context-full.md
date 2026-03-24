# Context

## What are Context

In the Agent Development Kit (ADK), "context" refers to the crucial bundle of information available to your agent and its tools during specific operations. Think of it as the necessary background knowledge and resources needed to handle a current task or conversation turn effectively.

Agents often need more than just the latest user message to perform well. Context is essential because it enables:

1. **Maintaining State:** Remembering details across multiple steps in a conversation (e.g., user preferences, previous calculations, items in a shopping cart). This is primarily managed through **session state**.
2. **Passing Data:** Sharing information discovered or generated in one step (like an LLM call or a tool execution) with subsequent steps. Session state is key here too.
3. **Accessing Services:** Interacting with framework capabilities like:
    * **Artifact Storage:** Saving or loading files or data blobs (like PDFs, images, configuration files) associated with the session.
    * **Memory:** Searching for relevant information from past interactions or external knowledge sources connected to the user.
    * **Authentication:** Requesting and retrieving credentials needed by tools to access external APIs securely.
4. **Identity and Tracking:** Knowing which agent is currently running (`agent.name`) and uniquely identifying the current request-response cycle (`invocation_id`) for logging and debugging.
5. **Tool-Specific Actions:** Enabling specialized operations within tools, such as requesting authentication or searching memory, which require access to the current interaction's details.


The central piece holding all this information together for a single, complete user-request-to-final-response cycle (an **invocation**) is the `InvocationContext`. However, you typically won't create or manage this object directly. The ADK framework creates it when an invocation starts (e.g., via `runner.run_async`) and passes the relevant contextual information implicitly to your agent code, callbacks, and tools.

=== "Python"

    ```python
    # Conceptual Pseudocode: How the framework provides context (Internal Logic)
    
    # runner = Runner(agent=my_root_agent, session_service=..., artifact_service=...)
    # user_message = types.Content(...)
    # session = session_service.get_session(...) # Or create new
    
    # --- Inside runner.run_async(...) ---
    # 1. Framework creates the main context for this specific run
    # invocation_context = InvocationContext(
    #     invocation_id="unique-id-for-this-run",
    #     session=session,
    #     user_content=user_message,
    #     agent=my_root_agent, # The starting agent
    #     session_service=session_service,
    #     artifact_service=artifact_service,
    #     memory_service=memory_service,
    #     # ... other necessary fields ...
    # )
    #
    # 2. Framework calls the agent's run method, passing the context implicitly
    #    (The agent's method signature will receive it, e.g., runAsyncImpl(InvocationContext invocationContext))
    # await my_root_agent.run_async(invocation_context)
    #   --- End Internal Logic ---
    #
    # As a developer, you work with the context objects provided in method arguments.
    ```

=== "Java"

    

## The Different types of Context

While `InvocationContext` acts as the comprehensive internal container, ADK provides specialized context objects tailored to specific situations. This ensures you have the right tools and permissions for the task at hand without needing to handle the full complexity of the internal context everywhere. Here are the different "flavors" you'll encounter:

1.  **`InvocationContext`**
    *   **Where Used:** Received as the `ctx` argument directly within an agent's core implementation methods (`_run_async_impl`, `_run_live_impl`).
    *   **Purpose:** Provides access to the *entire* state of the current invocation. This is the most comprehensive context object.
    *   **Key Contents:** Direct access to `session` (including `state` and `events`), the current `agent` instance, `invocation_id`, initial `user_content`, references to configured services (`artifact_service`, `memory_service`, `session_service`), and fields related to live/streaming modes.
    *   **Use Case:** Primarily used when the agent's core logic needs direct access to the overall session or services, though often state and artifact interactions are delegated to callbacks/tools which use their own contexts. Also used to control the invocation itself (e.g., setting `ctx.end_invocation = True`).

    === "Python"
    
        ```python
        # Pseudocode: Agent implementation receiving InvocationContext
        from google.adk.agents import BaseAgent
        from google.adk.agents.invocation_context import InvocationContext
        from google.adk.events import Event
        from typing import AsyncGenerator
    
        class MyAgent(BaseAgent):
            async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
                # Direct access example
                agent_name = ctx.agent.name
                session_id = ctx.session.id
                print(f"Agent {agent_name} running in session {session_id} for invocation {ctx.invocation_id}")
                # ... agent logic using ctx ...
                yield # ... event ...
        ```
    
    === "Java"
    
        

2.  **`ReadonlyContext`**
    *   **Where Used:** Provided in scenarios where only read access to basic information is needed and mutation is disallowed (e.g., `InstructionProvider` functions). It's also the base class for other contexts.
    *   **Purpose:** Offers a safe, read-only view of fundamental contextual details.
    *   **Key Contents:** `invocation_id`, `agent_name`, and a read-only *view* of the current `state`.

    === "Python"
    
        ```python
        # Pseudocode: Instruction provider receiving ReadonlyContext
        from google.adk.agents import ReadonlyContext
    
        def my_instruction_provider(context: ReadonlyContext) -> str:
            # Read-only access example
            user_tier = context.state().get("user_tier", "standard") # Can read state
            # context.state['new_key'] = 'value' # This would typically cause an error or be ineffective
            return f"Process the request for a {user_tier} user."
        ```
    
    === "Java"
    
        
    
3.  **`CallbackContext`**
    *   **Where Used:** Passed as `callback_context` to agent lifecycle callbacks (`before_agent_callback`, `after_agent_callback`) and model interaction callbacks (`before_model_callback`, `after_model_callback`).
    *   **Purpose:** Facilitates inspecting and modifying state, interacting with artifacts, and accessing invocation details *specifically within callbacks*.
    *   **Key Capabilities (Adds to `ReadonlyContext`):**
        *   **Mutable `state` Property:** Allows reading *and writing* to session state. Changes made here (`callback_context.state['key'] = value`) are tracked and associated with the event generated by the framework after the callback.
        *   **Artifact Methods:** `load_artifact(filename)` and `save_artifact(filename, part)` methods for interacting with the configured `artifact_service`.
        *   Direct `user_content` access.

    === "Python"
    
        ```python
        # Pseudocode: Callback receiving CallbackContext
        from google.adk.agents.callback_context import CallbackContext
        from google.adk.models import LlmRequest
        from google.genai import types
        from typing import Optional
    
        def my_before_model_cb(callback_context: CallbackContext, request: LlmRequest) -> Optional[types.Content]:
            # Read/Write state example
            call_count = callback_context.state.get("model_calls", 0)
            callback_context.state["model_calls"] = call_count + 1 # Modify state
    
            # Optionally load an artifact
            # config_part = callback_context.load_artifact("model_config.json")
            print(f"Preparing model call #{call_count + 1} for invocation {callback_context.invocation_id}")
            return None # Allow model call to proceed
        ```
    
    === "Java"
    
        

4.  **`ToolContext`**
    *   **Where Used:** Passed as `tool_context` to the functions backing `FunctionTool`s and to tool execution callbacks (`before_tool_callback`, `after_tool_callback`).
    *   **Purpose:** Provides everything `CallbackContext` does, plus specialized methods essential for tool execution, like handling authentication, searching memory, and listing artifacts.
    *   **Key Capabilities (Adds to `CallbackContext`):**
        *   **Authentication Methods:** `request_credential(auth_config)` to trigger an auth flow, and `get_auth_response(auth_config)` to retrieve credentials provided by the user/system.
        *   **Artifact Listing:** `list_artifacts()` to discover available artifacts in the session.
        *   **Memory Search:** `search_memory(query)` to query the configured `memory_service`.
        *   **`function_call_id` Property:** Identifies the specific function call from the LLM that triggered this tool execution, crucial for linking authentication requests or responses back correctly.
        *   **`actions` Property:** Direct access to the `EventActions` object for this step, allowing the tool to signal state changes, auth requests, etc.

    === "Python"
    
        ```python
        # Pseudocode: Tool function receiving ToolContext
        from google.adk.tools import ToolContext
        from typing import Dict, Any
    
        # Assume this function is wrapped by a FunctionTool
        def search_external_api(query: str, tool_context: ToolContext) -> Dict[str, Any]:
            api_key = tool_context.state.get("api_key")
            if not api_key:
                # Define required auth config
                # auth_config = AuthConfig(...)
                # tool_context.request_credential(auth_config) # Request credentials
                # Use the 'actions' property to signal the auth request has been made
                # tool_context.actions.requested_auth_configs[tool_context.function_call_id] = auth_config
                return {"status": "Auth Required"}
    
            # Use the API key...
            print(f"Tool executing for query '{query}' using API key. Invocation: {tool_context.invocation_id}")
    
            # Optionally search memory or list artifacts
            # relevant_docs = tool_context.search_memory(f"info related to {query}")
            # available_files = tool_context.list_artifacts()
    
            return {"result": f"Data for {query} fetched."}
        ```
    
    === "Java"
    
        

Understanding these different context objects and when to use them is key to effectively managing state, accessing services, and controlling the flow of your ADK application. The next section will detail common tasks you can perform using these contexts.


## Common Tasks Using Context

Now that you understand the different context objects, let's focus on how to use them for common tasks when building your agents and tools.

### Accessing Information

You'll frequently need to read information stored within the context.

*   **Reading Session State:** Access data saved in previous steps or user/app-level settings. Use dictionary-like access on the `state` property.

    === "Python"
    
        ```python
        # Pseudocode: In a Tool function
        from google.adk.tools import ToolContext
    
        def my_tool(tool_context: ToolContext, **kwargs):
            user_pref = tool_context.state.get("user_display_preference", "default_mode")
            api_endpoint = tool_context.state.get("app:api_endpoint") # Read app-level state
    
            if user_pref == "dark_mode":
                # ... apply dark mode logic ...
                pass
            print(f"Using API endpoint: {api_endpoint}")
            # ... rest of tool logic ...
    
        # Pseudocode: In a Callback function
        from google.adk.agents.callback_context import CallbackContext
    
        def my_callback(callback_context: CallbackContext, **kwargs):
            last_tool_result = callback_context.state.get("temp:last_api_result") # Read temporary state
            if last_tool_result:
                print(f"Found temporary result from last tool: {last_tool_result}")
            # ... callback logic ...
        ```
    
    === "Java"
    
        

*   **Getting Current Identifiers:** Useful for logging or custom logic based on the current operation.

    === "Python"
    
        ```python
        # Pseudocode: In any context (ToolContext shown)
        from google.adk.tools import ToolContext
    
        def log_tool_usage(tool_context: ToolContext, **kwargs):
            agent_name = tool_context.agent_nameSystem.out.println("Found temporary result from last tool: " + lastToolResult);
            inv_id = tool_context.invocation_id
            func_call_id = getattr(tool_context, 'function_call_id', 'N/A') # Specific to ToolContext
    
            print(f"Log: Invocation={inv_id}, Agent={agent_name}, FunctionCallID={func_call_id} - Tool Executed.")
        ```
    
    === "Java"
    
        

*   **Accessing the Initial User Input:** Refer back to the message that started the current invocation.

    === "Python"
    
        ```python
        # Pseudocode: In a Callback
        from google.adk.agents.callback_context import CallbackContext
    
        def check_initial_intent(callback_context: CallbackContext, **kwargs):
            initial_text = "N/A"
            if callback_context.user_content and callback_context.user_content.parts:
                initial_text = callback_context.user_content.parts[0].text or "Non-text input"
    
            print(f"This invocation started with user input: '{initial_text}'")
    
        # Pseudocode: In an Agent's _run_async_impl
        # async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        #     if ctx.user_content and ctx.user_content.parts:
        #         initial_text = ctx.user_content.parts[0].text
        #         print(f"Agent logic remembering initial query: {initial_text}")
        #     ...
        ```
    
    === "Java"
    
        
    
### Managing Session State

State is crucial for memory and data flow. When you modify state using `CallbackContext` or `ToolContext`, the changes are automatically tracked and persisted by the framework.

*   **How it Works:** Writing to `callback_context.state['my_key'] = my_value` or `tool_context.state['my_key'] = my_value` adds this change to the `EventActions.state_delta` associated with the current step's event. The `SessionService` then applies these deltas when persisting the event.
*   **Passing Data Between Tools:**

    === "Python"
    
        ```python
        # Pseudocode: Tool 1 - Fetches user ID
        from google.adk.tools import ToolContext
        import uuid
    
        def get_user_profile(tool_context: ToolContext) -> dict:
            user_id = str(uuid.uuid4()) # Simulate fetching ID
            # Save the ID to state for the next tool
            tool_context.state["temp:current_user_id"] = user_id
            return {"profile_status": "ID generated"}
    
        # Pseudocode: Tool 2 - Uses user ID from state
        def get_user_orders(tool_context: ToolContext) -> dict:
            user_id = tool_context.state.get("temp:current_user_id")
            if not user_id:
                return {"error": "User ID not found in state"}
    
            print(f"Fetching orders for user ID: {user_id}")
            # ... logic to fetch orders using user_id ...
            return {"orders": ["order123", "order456"]}
        ```
    
    === "Java"
    
        

*   **Updating User Preferences:**

    === "Python"
    
        ```python
        # Pseudocode: Tool or Callback identifies a preference
        from google.adk.tools import ToolContext # Or CallbackContext
    
        def set_user_preference(tool_context: ToolContext, preference: str, value: str) -> dict:
            # Use 'user:' prefix for user-level state (if using a persistent SessionService)
            state_key = f"user:{preference}"
            tool_context.state[state_key] = value
            print(f"Set user preference '{preference}' to '{value}'")
            return {"status": "Preference updated"}
        ```
    
    === "Java"
    
        

*   **State Prefixes:** While basic state is session-specific, prefixes like `app:` and `user:` can be used with persistent `SessionService` implementations (like `DatabaseSessionService` or `VertexAiSessionService`) to indicate broader scope (app-wide or user-wide across sessions). `temp:` can denote data only relevant within the current invocation.

### Working with Artifacts

Use artifacts to handle files or large data blobs associated with the session. Common use case: processing uploaded documents.

*   **Document Summarizer Example Flow:**

    1.  **Ingest Reference (e.g., in a Setup Tool or Callback):** Save the *path or URI* of the document, not the entire content, as an artifact.

        === "Python"
    
               ```python
               # Pseudocode: In a callback or initial tool
               from google.adk.agents import CallbackContext # Or ToolContext
               from google.genai import types
                
               def save_document_reference(context: CallbackContext, file_path: str) -> None:
                   # Assume file_path is something like "gs://my-bucket/docs/report.pdf" or "/local/path/to/report.pdf"
                   try:
                       # Create a Part containing the path/URI text
                       artifact_part = types.Part(text=file_path)
                       version = context.save_artifact("document_to_summarize.txt", artifact_part)
                       print(f"Saved document reference '{file_path}' as artifact version {version}")
                       # Store the filename in state if needed by other tools
                       context.state["temp:doc_artifact_name"] = "document_to_summarize.txt"
                   except ValueError as e:
                       print(f"Error saving artifact: {e}") # E.g., Artifact service not configured
                   except Exception as e:
                       print(f"Unexpected error saving artifact reference: {e}")
                
               # Example usage:
               # save_document_reference(callback_context, "gs://my-bucket/docs/report.pdf")
               ```
    
        === "Java"
    
               

    2.  **Summarizer Tool:** Load the artifact to get the path/URI, read the actual document content using appropriate libraries, summarize, and return the result.

        === "Python"

            ```python
            # Pseudocode: In the Summarizer tool function
            from google.adk.tools import ToolContext
            from google.genai import types
            # Assume libraries like google.cloud.storage or built-in open are available
            # Assume a 'summarize_text' function exists
            # from my_summarizer_lib import summarize_text

            def summarize_document_tool(tool_context: ToolContext) -> dict:
                artifact_name = tool_context.state.get("temp:doc_artifact_name")
                if not artifact_name:
                    return {"error": "Document artifact name not found in state."}

                try:
                    # 1. Load the artifact part containing the path/URI
                    artifact_part = tool_context.load_artifact(artifact_name)
                    if not artifact_part or not artifact_part.text:
                        return {"error": f"Could not load artifact or artifact has no text path: {artifact_name}"}

                    file_path = artifact_part.text
                    print(f"Loaded document reference: {file_path}")

                    # 2. Read the actual document content (outside ADK context)
                    document_content = ""
                    if file_path.startswith("gs://"):
                        # Example: Use GCS client library to download/read
                        # from google.cloud import storage
                        # client = storage.Client()
                        # blob = storage.Blob.from_string(file_path, client=client)
                        # document_content = blob.download_as_text() # Or bytes depending on format
                        pass # Replace with actual GCS reading logic
                    elif file_path.startswith("/"):
                         # Example: Use local file system
                         with open(file_path, 'r', encoding='utf-8') as f:
                             document_content = f.read()
                    else:
                        return {"error": f"Unsupported file path scheme: {file_path}"}

                    # 3. Summarize the content
                    if not document_content:
                         return {"error": "Failed to read document content."}

                    # summary = summarize_text(document_content) # Call your summarization logic
                    summary = f"Summary of content from {file_path}" # Placeholder

                    return {"summary": summary}

                except ValueError as e:
                     return {"error": f"Artifact service error: {e}"}
                except FileNotFoundError:
                     return {"error": f"Local file not found: {file_path}"}
                # except Exception as e: # Catch specific exceptions for GCS etc.
                #      return {"error": f"Error reading document {file_path}: {e}"}
            ```

        === "Java"

            
    
*   **Listing Artifacts:** Discover what files are available.
    
    === "Python"
        
        ```python
        # Pseudocode: In a tool function
        from google.adk.tools import ToolContext
        
        def check_available_docs(tool_context: ToolContext) -> dict:
            try:
                artifact_keys = tool_context.list_artifacts()
                print(f"Available artifacts: {artifact_keys}")
                return {"available_docs": artifact_keys}
            except ValueError as e:
                return {"error": f"Artifact service error: {e}"}
        ```
        
    === "Java"
        
        

### Handling Tool Authentication 

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

Securely manage API keys or other credentials needed by tools.

```python
# Pseudocode: Tool requiring auth
from google.adk.tools import ToolContext
from google.adk.auth import AuthConfig # Assume appropriate AuthConfig is defined

# Define your required auth configuration (e.g., OAuth, API Key)
MY_API_AUTH_CONFIG = AuthConfig(...)
AUTH_STATE_KEY = "user:my_api_credential" # Key to store retrieved credential

def call_secure_api(tool_context: ToolContext, request_data: str) -> dict:
    # 1. Check if credential already exists in state
    credential = tool_context.state.get(AUTH_STATE_KEY)

    if not credential:
        # 2. If not, request it
        print("Credential not found, requesting...")
        try:
            tool_context.request_credential(MY_API_AUTH_CONFIG)
            # The framework handles yielding the event. The tool execution stops here for this turn.
            return {"status": "Authentication required. Please provide credentials."}
        except ValueError as e:
            return {"error": f"Auth error: {e}"} # e.g., function_call_id missing
        except Exception as e:
            return {"error": f"Failed to request credential: {e}"}

    # 3. If credential exists (might be from a previous turn after request)
    #    or if this is a subsequent call after auth flow completed externally
    try:
        # Optionally, re-validate/retrieve if needed, or use directly
        # This might retrieve the credential if the external flow just completed
        auth_credential_obj = tool_context.get_auth_response(MY_API_AUTH_CONFIG)
        api_key = auth_credential_obj.api_key # Or access_token, etc.

        # Store it back in state for future calls within the session
        tool_context.state[AUTH_STATE_KEY] = auth_credential_obj.model_dump() # Persist retrieved credential

        print(f"Using retrieved credential to call API with data: {request_data}")
        # ... Make the actual API call using api_key ...
        api_result = f"API result for {request_data}"

        return {"result": api_result}
    except Exception as e:
        # Handle errors retrieving/using the credential
        print(f"Error using credential: {e}")
        # Maybe clear the state key if credential is invalid?
        # tool_context.state[AUTH_STATE_KEY] = None
        return {"error": "Failed to use credential"}

```
*Remember: `request_credential` pauses the tool and signals the need for authentication. The user/system provides credentials, and on a subsequent call, `get_auth_response` (or checking state again) allows the tool to proceed.* The `tool_context.function_call_id` is used implicitly by the framework to link the request and response.

### Leveraging Memory 

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

Access relevant information from the past or external sources.

```python
# Pseudocode: Tool using memory search
from google.adk.tools import ToolContext

def find_related_info(tool_context: ToolContext, topic: str) -> dict:
    try:
        search_results = tool_context.search_memory(f"Information about {topic}")
        if search_results.results:
            print(f"Found {len(search_results.results)} memory results for '{topic}'")
            # Process search_results.results (which are SearchMemoryResponseEntry)
            top_result_text = search_results.results[0].text
            return {"memory_snippet": top_result_text}
        else:
            return {"message": "No relevant memories found."}
    except ValueError as e:
        return {"error": f"Memory service error: {e}"} # e.g., Service not configured
    except Exception as e:
        return {"error": f"Unexpected error searching memory: {e}"}
```

### Advanced: Direct `InvocationContext` Usage 

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

While most interactions happen via `CallbackContext` or `ToolContext`, sometimes the agent's core logic (`_run_async_impl`/`_run_live_impl`) needs direct access.

```python
# Pseudocode: Inside agent's _run_async_impl
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event
from typing import AsyncGenerator

class MyControllingAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        # Example: Check if a specific service is available
        if not ctx.memory_service:
            print("Memory service is not available for this invocation.")
            # Potentially change agent behavior

        # Example: Early termination based on some condition
        if ctx.session.state.get("critical_error_flag"):
            print("Critical error detected, ending invocation.")
            ctx.end_invocation = True # Signal framework to stop processing
            yield Event(author=self.name, invocation_id=ctx.invocation_id, content="Stopping due to critical error.")
            return # Stop this agent's execution

        # ... Normal agent processing ...
        yield # ... event ...
```

Setting `ctx.end_invocation = True` is a way to gracefully stop the entire request-response cycle from within the agent or its callbacks/tools (via their respective context objects which also have access to modify the underlying `InvocationContext`'s flag).

## Key Takeaways & Best Practices

*   **Use the Right Context:** Always use the most specific context object provided (`ToolContext` in tools/tool-callbacks, `CallbackContext` in agent/model-callbacks, `ReadonlyContext` where applicable). Use the full `InvocationContext` (`ctx`) directly in `_run_async_impl` / `_run_live_impl` only when necessary.
*   **State for Data Flow:** `context.state` is the primary way to share data, remember preferences, and manage conversational memory *within* an invocation. Use prefixes (`app:`, `user:`, `temp:`) thoughtfully when using persistent storage.
*   **Artifacts for Files:** Use `context.save_artifact` and `context.load_artifact` for managing file references (like paths or URIs) or larger data blobs. Store references, load content on demand.
*   **Tracked Changes:** Modifications to state or artifacts made via context methods are automatically linked to the current step's `EventActions` and handled by the `SessionService`.
*   **Start Simple:** Focus on `state` and basic artifact usage first. Explore authentication, memory, and advanced `InvocationContext` fields (like those for live streaming) as your needs become more complex.

By understanding and effectively using these context objects, you can build more sophisticated, stateful, and capable agents with ADK.


Thank you for your interest in contributing to the Agent Development Kit (ADK)! We welcome contributions to both the core framework (Python and Java) and its documentation.

This guide provides information on how to get involved.

## 1. [`google/adk-python`](https://github.com/google/adk-python)

Contains the core Python library source code.

## 2. [`google/adk-java`](https://github.com/google/adk-java)

Contains the core Java library source code.

## 3. [`google/adk-docs`](https://github.com/google/adk-docs)

Contains the source for the documentation site you are currently reading.

## 4. [`google/adk-web`](https://github.com/google/adk-web)

Contains the source for the `adk web` dev UI.

## Before you begin

### ✏️ Sign our Contributor License Agreement

Contributions to this project must be accompanied by a
[Contributor License Agreement](https://cla.developers.google.com/about) (CLA).
You (or your employer) retain the copyright to your contribution; this simply
gives us permission to use and redistribute your contributions as part of the
project.

If you or your current employer have already signed the Google CLA (even if it
was for a different project), you probably don't need to do it again.

Visit <https://cla.developers.google.com/> to see your current agreements or to
sign a new one.

### 📜 Review our community guidelines

This project follows
[Google's Open Source Community Guidelines](https://opensource.google/conduct/).

## 💬 Join the Discussion!

Have questions, want to share ideas, or discuss how you're using the ADK? Head over to our **[Python](https://github.com/google/adk-python/discussions)** or **[Java](https://github.com/google/adk-java/discussions)** Discussions!

This is the primary place for:

* Asking questions and getting help from the community and maintainers.
* Sharing your projects or use cases (`Show and Tell`).
* Discussing potential features or improvements before creating a formal issue.
* General conversation about the ADK.

## How to Contribute

There are several ways you can contribute to the ADK:

### 1. Reporting Issues (Bugs & Errors)

If you find a bug in the framework or an error in the documentation:

* **Framework Bugs:** Open an issue in [`google/adk-python`](https://github.com/google/adk-python/issues/new) or in [`google/adk-java`](https://github.com/google/adk-java/issues/new)
* **Documentation Errors:** [Open an issue in `google/adk-docs` (use bug template)](https://github.com/google/adk-docs/issues/new?template=bug_report.md)

### 2. Suggesting Enhancements

Have an idea for a new feature or an improvement to an existing one?

* **Framework Enhancements:** Open an issue in [`google/adk-python`](https://github.com/google/adk-python/issues/new) or in [`google/adk-java`](https://github.com/google/adk-java/issues/new)
* **Documentation Enhancements:** [Open an issue in `google/adk-docs`](https://github.com/google/adk-docs/issues/new)

### 3. Improving Documentation

Found a typo, unclear explanation, or missing information? Submit your changes directly:

* **How:** Submit a Pull Request (PR) with your suggested improvements.
* **Where:** [Create a Pull Request in `google/adk-docs`](https://github.com/google/adk-docs/pulls)

### 4. Writing Code

Help fix bugs, implement new features or contribute code samples for the documentation:

**How:** Submit a Pull Request (PR) with your code changes.

* **Python Framework:** [Create a Pull Request in `google/adk-python`](https://github.com/google/adk-python/pulls)
* **Java Framework:** [Create a Pull Request in `google/adk-java`](https://github.com/google/adk-java/pulls)
* **Documentation:** [Create a Pull Request in `google/adk-docs`](https://github.com/google/adk-docs/pulls)

### Code Reviews

* All contributions, including those from project members, undergo a review process.

* We use GitHub Pull Requests (PRs) for code submission and review. Please ensure your PR clearly describes the changes you are making.

## License

By contributing, you agree that your contributions will be licensed under the project's [Apache 2.0 License](https://github.com/google/adk-docs/blob/main/LICENSE).

## Questions?

If you get stuck or have questions, feel free to open an issue on the relevant repository's issue tracker.


# Artifacts

In ADK, **Artifacts** represent a crucial mechanism for managing named, versioned binary data associated either with a specific user interaction session or persistently with a user across multiple sessions. They allow your agents and tools to handle data beyond simple text strings, enabling richer interactions involving files, images, audio, and other binary formats.

!!! Note
    The specific parameters or method names for the primitives may vary slightly by SDK language (e.g., `save_artifact` in Python, `saveArtifact` in Java). Refer to the language-specific API documentation for details.

## What are Artifacts?

*   **Definition:** An Artifact is essentially a piece of binary data (like the content of a file) identified by a unique `filename` string within a specific scope (session or user). Each time you save an artifact with the same filename, a new version is created.

*   **Representation:** Artifacts are consistently represented using the standard `google.genai.types.Part` object. The core data is typically stored within an inline data structure of the `Part` (accessed via `inline_data`), which itself contains:
    *   `data`: The raw binary content as bytes.
    *   `mime_type`: A string indicating the type of the data (e.g., `"image/png"`, `"application/pdf"`). This is essential for correctly interpreting the data later.


=== "Python"

    ```py
    # Example of how an artifact might be represented as a types.Part
    import google.genai.types as types

    # Assume 'image_bytes' contains the binary data of a PNG image
    image_bytes = b'\x89PNG\r\n\x1a\n...' # Placeholder for actual image bytes

    image_artifact = types.Part(
        inline_data=types.Blob(
            mime_type="image/png",
            data=image_bytes
        )
    )

    # You can also use the convenience constructor:
    # image_artifact_alt = types.Part.from_bytes(data=image_bytes, mime_type="image/png")

    print(f"Artifact MIME Type: {image_artifact.inline_data.mime_type}")
    print(f"Artifact Data (first 10 bytes): {image_artifact.inline_data.data[:10]}...")
    ```

=== "Java"

    

*   **Persistence & Management:** Artifacts are not stored directly within the agent or session state. Their storage and retrieval are managed by a dedicated **Artifact Service** (an implementation of `BaseArtifactService`, defined in `google.adk.artifacts`. ADK provides various implementations, such as:
    *   An in-memory service for testing or temporary storage (e.g., `InMemoryArtifactService` in Python, defined in `google.adk.artifacts.in_memory_artifact_service.py`).
    *   A service for persistent storage using Google Cloud Storage (GCS) (e.g., `GcsArtifactService` in Python, defined in `google.adk.artifacts.gcs_artifact_service.py`).
    The chosen service implementation handles versioning automatically when you save data.

## Why Use Artifacts?

While session `state` is suitable for storing small pieces of configuration or conversational context (like strings, numbers, booleans, or small dictionaries/lists), Artifacts are designed for scenarios involving binary or large data:

1. **Handling Non-Textual Data:** Easily store and retrieve images, audio clips, video snippets, PDFs, spreadsheets, or any other file format relevant to your agent's function.  
2. **Persisting Large Data:** Session state is generally not optimized for storing large amounts of data. Artifacts provide a dedicated mechanism for persisting larger blobs without cluttering the session state.  
3. **User File Management:** Provide capabilities for users to upload files (which can be saved as artifacts) and retrieve or download files generated by the agent (loaded from artifacts).  
4. **Sharing Outputs:** Enable tools or agents to generate binary outputs (like a PDF report or a generated image) that can be saved via `save_artifact` and later accessed by other parts of the application or even in subsequent sessions (if using user namespacing).  
5. **Caching Binary Data:** Store the results of computationally expensive operations that produce binary data (e.g., rendering a complex chart image) as artifacts to avoid regenerating them on subsequent requests.

In essence, whenever your agent needs to work with file-like binary data that needs to be persisted, versioned, or shared, Artifacts managed by an `ArtifactService` are the appropriate mechanism within ADK.


## Common Use Cases

Artifacts provide a flexible way to handle binary data within your ADK applications.

Here are some typical scenarios where they prove valuable:

* **Generated Reports/Files:**
    * A tool or agent generates a report (e.g., a PDF analysis, a CSV data export, an image chart).

* **Handling User Uploads:**

    * A user uploads a file (e.g., an image for analysis, a document for summarization) through a front-end interface.

* **Storing Intermediate Binary Results:**

    * An agent performs a complex multi-step process where one step generates intermediate binary data (e.g., audio synthesis, simulation results).

* **Persistent User Data:**

    * Storing user-specific configuration or data that isn't a simple key-value state.

* **Caching Generated Binary Content:**

    * An agent frequently generates the same binary output based on certain inputs (e.g., a company logo image, a standard audio greeting).



## Core Concepts

Understanding artifacts involves grasping a few key components: the service that manages them, the data structure used to hold them, and how they are identified and versioned.

### Artifact Service (`BaseArtifactService`)

* **Role:** The central component responsible for the actual storage and retrieval logic for artifacts. It defines *how* and *where* artifacts are persisted.  

* **Interface:** Defined by the abstract base class `BaseArtifactService`. Any concrete implementation must provide methods for:  

    * `Save Artifact`: Stores the artifact data and returns its assigned version number.  
    * `Load Artifact`: Retrieves a specific version (or the latest) of an artifact.  
    * `List Artifact keys`: Lists the unique filenames of artifacts within a given scope.  
    * `Delete Artifact`: Removes an artifact (and potentially all its versions, depending on implementation).  
    * `List versions`: Lists all available version numbers for a specific artifact filename.

* **Configuration:** You provide an instance of an artifact service (e.g., `InMemoryArtifactService`, `GcsArtifactService`) when initializing the `Runner`. The `Runner` then makes this service available to agents and tools via the `InvocationContext`.

=== "Python"

    ```py
    from google.adk.runners import Runner
    from google.adk.artifacts import InMemoryArtifactService # Or GcsArtifactService
    from google.adk.agents import LlmAgent # Any agent
    from google.adk.sessions import InMemorySessionService

    # Example: Configuring the Runner with an Artifact Service
    my_agent = LlmAgent(name="artifact_user_agent", model="gemini-2.5-flash")
    artifact_service = InMemoryArtifactService() # Choose an implementation
    session_service = InMemorySessionService()

    runner = Runner(
        agent=my_agent,
        app_name="my_artifact_app",
        session_service=session_service,
        artifact_service=artifact_service # Provide the service instance here
    )
    # Now, contexts within runs managed by this runner can use artifact methods
    ```

=== "Java"
    
    

### Artifact Data

* **Standard Representation:** Artifact content is universally represented using the `google.genai.types.Part` object, the same structure used for parts of LLM messages.  

* **Key Attribute (`inline_data`):** For artifacts, the most relevant attribute is `inline_data`, which is a `google.genai.types.Blob` object containing:  

    * `data` (`bytes`): The raw binary content of the artifact.  
    * `mime_type` (`str`): A standard MIME type string (e.g., `'application/pdf'`, `'image/png'`, `'audio/mpeg'`) describing the nature of the binary data. **This is crucial for correct interpretation when loading the artifact.**

=== "Python"

    ```python
    import google.genai.types as types

    # Example: Creating an artifact Part from raw bytes
    pdf_bytes = b'%PDF-1.4...' # Your raw PDF data
    pdf_mime_type = "application/pdf"

    # Using the constructor
    pdf_artifact_py = types.Part(
        inline_data=types.Blob(data=pdf_bytes, mime_type=pdf_mime_type)
    )

    # Using the convenience class method (equivalent)
    pdf_artifact_alt_py = types.Part.from_bytes(data=pdf_bytes, mime_type=pdf_mime_type)

    print(f"Created Python artifact with MIME type: {pdf_artifact_py.inline_data.mime_type}")
    ```
    
=== "Java"

    

### Filename

* **Identifier:** A simple string used to name and retrieve an artifact within its specific namespace.  
* **Uniqueness:** Filenames must be unique within their scope (either the session or the user namespace).  
* **Best Practice:** Use descriptive names, potentially including file extensions (e.g., `"monthly_report.pdf"`, `"user_avatar.jpg"`), although the extension itself doesn't dictate behavior – the `mime_type` does.

### Versioning

* **Automatic Versioning:** The artifact service automatically handles versioning. When you call `save_artifact`, the service determines the next available version number (typically starting from 0 and incrementing) for that specific filename and scope.  
* **Returned by `save_artifact`:** The `save_artifact` method returns the integer version number that was assigned to the newly saved artifact.  
* **Retrieval:**  
  * `load_artifact(..., version=None)` (default): Retrieves the *latest* available version of the artifact.  
  * `load_artifact(..., version=N)`: Retrieves the specific version `N`.  
* **Listing Versions:** The `list_versions` method (on the service, not context) can be used to find all existing version numbers for an artifact.

### Namespacing (Session vs. User)

* **Concept:** Artifacts can be scoped either to a specific session or more broadly to a user across all their sessions within the application. This scoping is determined by the `filename` format and handled internally by the `ArtifactService`.  

* **Default (Session Scope):** If you use a plain filename like `"report.pdf"`, the artifact is associated with the specific `app_name`, `user_id`, *and* `session_id`. It's only accessible within that exact session context.  


* **User Scope (`"user:"` prefix):** If you prefix the filename with `"user:"`, like `"user:profile.png"`, the artifact is associated only with the `app_name` and `user_id`. It can be accessed or updated from *any* session belonging to that user within the app.  


=== "Python"

    ```python
    # Example illustrating namespace difference (conceptual)

    # Session-specific artifact filename
    session_report_filename = "summary.txt"

    # User-specific artifact filename
    user_config_filename = "user:settings.json"

    # When saving 'summary.txt' via context.save_artifact,
    # it's tied to the current app_name, user_id, and session_id.

    # When saving 'user:settings.json' via context.save_artifact,
    # the ArtifactService implementation should recognize the "user:" prefix
    # and scope it to app_name and user_id, making it accessible across sessions for that user.
    ```

=== "Java"

    

These core concepts work together to provide a flexible system for managing binary data within the ADK framework.

## Interacting with Artifacts (via Context Objects)

The primary way you interact with artifacts within your agent's logic (specifically within callbacks or tools) is through methods provided by the `CallbackContext` and `ToolContext` objects. These methods abstract away the underlying storage details managed by the `ArtifactService`.

### Prerequisite: Configuring the `ArtifactService`

Before you can use any artifact methods via the context objects, you **must** provide an instance of a [`BaseArtifactService` implementation](#available-implementations) (like [`InMemoryArtifactService`](#inmemoryartifactservice) or [`GcsArtifactService`](#gcsartifactservice)) when initializing your `Runner`.

=== "Python"

    In Python, you provide this instance when initializing your `Runner`.

    ```python
    from google.adk.runners import Runner
    from google.adk.artifacts import InMemoryArtifactService # Or GcsArtifactService
    from google.adk.agents import LlmAgent
    from google.adk.sessions import InMemorySessionService

    # Your agent definition
    agent = LlmAgent(name="my_agent", model="gemini-2.5-flash")

    # Instantiate the desired artifact service
    artifact_service = InMemoryArtifactService()

    # Provide it to the Runner
    runner = Runner(
        agent=agent,
        app_name="artifact_app",
        session_service=InMemorySessionService(),
        artifact_service=artifact_service # Service must be provided here
    )
    ```
    If no `artifact_service` is configured in the `InvocationContext` (which happens if it's not passed to the `Runner`), calling `save_artifact`, `load_artifact`, or `list_artifacts` on the context objects will raise a `ValueError`.

=== "Java"

    In Java, you would instantiate a `BaseArtifactService` implementation and then ensure it's accessible to the parts of your application that manage artifacts. This is often done through dependency injection or by explicitly passing the service instance.

    
    In Java, if an `ArtifactService` instance is not available (e.g., `null`) when artifact operations are attempted, it would typically result in a `NullPointerException` or a custom error, depending on how your application is structured. Robust applications often use dependency injection frameworks to manage service lifecycles and ensure availability.


### Accessing Methods

The artifact interaction methods are available directly on instances of `CallbackContext` (passed to agent and model callbacks) and `ToolContext` (passed to tool callbacks). Remember that `ToolContext` inherits from `CallbackContext`.

*   **Code Example:**

    === "Python"

        ```python
        import google.genai.types as types
        from google.adk.agents.callback_context import CallbackContext # Or ToolContext

        async def save_generated_report_py(context: CallbackContext, report_bytes: bytes):
            """Saves generated PDF report bytes as an artifact."""
            report_artifact = types.Part.from_data(
                data=report_bytes,
                mime_type="application/pdf"
            )
            filename = "generated_report.pdf"

            try:
                version = await context.save_artifact(filename=filename, artifact=report_artifact)
                print(f"Successfully saved Python artifact '{filename}' as version {version}.")
                # The event generated after this callback will contain:
                # event.actions.artifact_delta == {"generated_report.pdf": version}
            except ValueError as e:
                print(f"Error saving Python artifact: {e}. Is ArtifactService configured in Runner?")
            except Exception as e:
                # Handle potential storage errors (e.g., GCS permissions)
                print(f"An unexpected error occurred during Python artifact save: {e}")

        # --- Example Usage Concept (Python) ---
        # async def main_py():
        #   callback_context: CallbackContext = ... # obtain context
        #   report_data = b'...' # Assume this holds the PDF bytes
        #   await save_generated_report_py(callback_context, report_data)
        ```

    === "Java"
    
        

#### Loading Artifacts

*   **Code Example:**

    === "Python"

        ```python
        import google.genai.types as types
        from google.adk.agents.callback_context import CallbackContext # Or ToolContext

        async def process_latest_report_py(context: CallbackContext):
            """Loads the latest report artifact and processes its data."""
            filename = "generated_report.pdf"
            try:
                # Load the latest version
                report_artifact = await context.load_artifact(filename=filename)

                if report_artifact and report_artifact.inline_data:
                    print(f"Successfully loaded latest Python artifact '{filename}'.")
                    print(f"MIME Type: {report_artifact.inline_data.mime_type}")
                    # Process the report_artifact.inline_data.data (bytes)
                    pdf_bytes = report_artifact.inline_data.data
                    print(f"Report size: {len(pdf_bytes)} bytes.")
                    # ... further processing ...
                else:
                    print(f"Python artifact '{filename}' not found.")

                # Example: Load a specific version (if version 0 exists)
                # specific_version_artifact = await context.load_artifact(filename=filename, version=0)
                # if specific_version_artifact:
                #     print(f"Loaded version 0 of '{filename}'.")

            except ValueError as e:
                print(f"Error loading Python artifact: {e}. Is ArtifactService configured?")
            except Exception as e:
                # Handle potential storage errors
                print(f"An unexpected error occurred during Python artifact load: {e}")

        # --- Example Usage Concept (Python) ---
        # async def main_py():
        #   callback_context: CallbackContext = ... # obtain context
        #   await process_latest_report_py(callback_context)
        ```

    === "Java"

        

#### Listing Artifact Filenames

*   **Code Example:**

    === "Python"

        ```python
        from google.adk.tools.tool_context import ToolContext

        def list_user_files_py(tool_context: ToolContext) -> str:
            """Tool to list available artifacts for the user."""
            try:
                available_files = await tool_context.list_artifacts()
                if not available_files:
                    return "You have no saved artifacts."
                else:
                    # Format the list for the user/LLM
                    file_list_str = "\n".join([f"- {fname}" for fname in available_files])
                    return f"Here are your available Python artifacts:\n{file_list_str}"
            except ValueError as e:
                print(f"Error listing Python artifacts: {e}. Is ArtifactService configured?")
                return "Error: Could not list Python artifacts."
            except Exception as e:
                print(f"An unexpected error occurred during Python artifact list: {e}")
                return "Error: An unexpected error occurred while listing Python artifacts."

        # This function would typically be wrapped in a FunctionTool
        # from google.adk.tools import FunctionTool
        # list_files_tool = FunctionTool(func=list_user_files_py)
        ```

    === "Java"

        

These methods for saving, loading, and listing provide a convenient and consistent way to manage binary data persistence within ADK, whether using Python's context objects or directly interacting with the `BaseArtifactService` in Java, regardless of the chosen backend storage implementation.

## Available Implementations

ADK provides concrete implementations of the `BaseArtifactService` interface, offering different storage backends suitable for various development stages and deployment needs. These implementations handle the details of storing, versioning, and retrieving artifact data based on the `app_name`, `user_id`, `session_id`, and `filename` (including the `user:` namespace prefix).

### InMemoryArtifactService

*   **Storage Mechanism:**
    *   Python: Uses a Python dictionary (`self.artifacts`) held in the application's memory. The dictionary keys represent the artifact path, and the values are lists of `types.Part`, where each list element is a version.
    *   Java: Uses nested `HashMap` instances (`private final Map<String, Map<String, Map<String, Map<String, List<Part>>>>> artifacts;`) held in memory. The keys at each level are `appName`, `userId`, `sessionId`, and `filename` respectively. The innermost `List<Part>` stores the versions of the artifact, where the list index corresponds to the version number.
*   **Key Features:**
    *   **Simplicity:** Requires no external setup or dependencies beyond the core ADK library.
    *   **Speed:** Operations are typically very fast as they involve in-memory map/dictionary lookups and list manipulations.
    *   **Ephemeral:** All stored artifacts are **lost** when the application process terminates. Data does not persist between application restarts.
*   **Use Cases:**
    *   Ideal for local development and testing where persistence is not required.
    *   Suitable for short-lived demonstrations or scenarios where artifact data is purely temporary within a single run of the application.
*   **Instantiation:**

    === "Python"

        ```python
        from google.adk.artifacts import InMemoryArtifactService

        # Simply instantiate the class
        in_memory_service_py = InMemoryArtifactService()

        # Then pass it to the Runner
        # runner = Runner(..., artifact_service=in_memory_service_py)
        ```

    === "Java"

        

### GcsArtifactService


*   **Storage Mechanism:** Leverages Google Cloud Storage (GCS) for persistent artifact storage. Each version of an artifact is stored as a separate object (blob) within a specified GCS bucket.
*   **Object Naming Convention:** It constructs GCS object names (blob names) using a hierarchical path structure.
*   **Key Features:**
    *   **Persistence:** Artifacts stored in GCS persist across application restarts and deployments.
    *   **Scalability:** Leverages the scalability and durability of Google Cloud Storage.
    *   **Versioning:** Explicitly stores each version as a distinct GCS object. The `saveArtifact` method in `GcsArtifactService`.
    *   **Permissions Required:** The application environment needs appropriate credentials (e.g., Application Default Credentials) and IAM permissions to read from and write to the specified GCS bucket.
*   **Use Cases:**
    *   Production environments requiring persistent artifact storage.
    *   Scenarios where artifacts need to be shared across different application instances or services (by accessing the same GCS bucket).
    *   Applications needing long-term storage and retrieval of user or session data.
*   **Instantiation:**

    === "Python"

        ```python
        from google.adk.artifacts import GcsArtifactService

        # Specify the GCS bucket name
        gcs_bucket_name_py = "your-gcs-bucket-for-adk-artifacts" # Replace with your bucket name

        try:
            gcs_service_py = GcsArtifactService(bucket_name=gcs_bucket_name_py)
            print(f"Python GcsArtifactService initialized for bucket: {gcs_bucket_name_py}")
            # Ensure your environment has credentials to access this bucket.
            # e.g., via Application Default Credentials (ADC)

            # Then pass it to the Runner
            # runner = Runner(..., artifact_service=gcs_service_py)

        except Exception as e:
            # Catch potential errors during GCS client initialization (e.g., auth issues)
            print(f"Error initializing Python GcsArtifactService: {e}")
            # Handle the error appropriately - maybe fall back to InMemory or raise
        ```

    === "Java"

        

Choosing the appropriate `ArtifactService` implementation depends on your application's requirements for data persistence, scalability, and operational environment.

## Best Practices

To use artifacts effectively and maintainably:

* **Choose the Right Service:** Use `InMemoryArtifactService` for rapid prototyping, testing, and scenarios where persistence isn't needed. Use `GcsArtifactService` (or implement your own `BaseArtifactService` for other backends) for production environments requiring data persistence and scalability.  
* **Meaningful Filenames:** Use clear, descriptive filenames. Including relevant extensions (`.pdf`, `.png`, `.wav`) helps humans understand the content, even though the `mime_type` dictates programmatic handling. Establish conventions for temporary vs. persistent artifact names.  
* **Specify Correct MIME Types:** Always provide an accurate `mime_type` when creating the `types.Part` for `save_artifact`. This is critical for applications or tools that later `load_artifact` to interpret the `bytes` data correctly. Use standard IANA MIME types where possible.  
* **Understand Versioning:** Remember that `load_artifact()` without a specific `version` argument retrieves the *latest* version. If your logic depends on a specific historical version of an artifact, be sure to provide the integer version number when loading.  
* **Use Namespacing (`user:`) Deliberately:** Only use the `"user:"` prefix for filenames when the data truly belongs to the user and should be accessible across all their sessions. For data specific to a single conversation or session, use regular filenames without the prefix.  
* **Error Handling:**  
    * Always check if an `artifact_service` is actually configured before calling context methods (`save_artifact`, `load_artifact`, `list_artifacts`) – they will raise a `ValueError` if the service is `None`. 
    * Check the return value of `load_artifact`, as it will be `None` if the artifact or version doesn't exist. Don't assume it always returns a `Part`.  
    * Be prepared to handle exceptions from the underlying storage service, especially with `GcsArtifactService` (e.g., `google.api_core.exceptions.Forbidden` for permission issues, `NotFound` if the bucket doesn't exist, network errors).  
* **Size Considerations:** Artifacts are suitable for typical file sizes, but be mindful of potential costs and performance impacts with extremely large files, especially with cloud storage. `InMemoryArtifactService` can consume significant memory if storing many large artifacts. Evaluate if very large data might be better handled through direct GCS links or other specialized storage solutions rather than passing entire byte arrays in-memory.  
* **Cleanup Strategy:** For persistent storage like `GcsArtifactService`, artifacts remain until explicitly deleted. If artifacts represent temporary data or have a limited lifespan, implement a strategy for cleanup. This might involve:  
    * Using GCS lifecycle policies on the bucket.  
    * Building specific tools or administrative functions that utilize the `artifact_service.delete_artifact` method (note: delete is *not* exposed via context objects for safety).  
    * Carefully managing filenames to allow pattern-based deletion if needed.

# Memory: Long-Term Knowledge with `MemoryService`

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

We've seen how `Session` tracks the history (`events`) and temporary data (`state`) for a *single, ongoing conversation*. But what if an agent needs to recall information from *past* conversations or access external knowledge bases? This is where the concept of **Long-Term Knowledge** and the **`MemoryService`** come into play.

Think of it this way:

* **`Session` / `State`:** Like your short-term memory during one specific chat.  
* **Long-Term Knowledge (`MemoryService`)**: Like a searchable archive or knowledge library the agent can consult, potentially containing information from many past chats or other sources.

## The `MemoryService` Role

The `BaseMemoryService` defines the interface for managing this searchable, long-term knowledge store. Its primary responsibilities are:

1. **Ingesting Information (`add_session_to_memory`):** Taking the contents of a (usually completed) `Session` and adding relevant information to the long-term knowledge store.  
2. **Searching Information (`search_memory`):** Allowing an agent (typically via a `Tool`) to query the knowledge store and retrieve relevant snippets or context based on a search query.

## `MemoryService` Implementations

ADK provides different ways to implement this long-term knowledge store:

1. **`InMemoryMemoryService`**  

    * **How it works:** Stores session information in the application's memory and performs basic keyword matching for searches.  
    * **Persistence:** None. **All stored knowledge is lost if the application restarts.**  
    * **Requires:** Nothing extra.  
    * **Best for:** Prototyping, simple testing, scenarios where only basic keyword recall is needed and persistence isn't required.

    ```py
    from google.adk.memory import InMemoryMemoryService
    memory_service = InMemoryMemoryService()
    ```

2. **`VertexAiRagMemoryService`**  

    * **How it works:** Leverages Google Cloud's Vertex AI RAG (Retrieval-Augmented Generation) service. It ingests session data into a specified RAG Corpus and uses powerful semantic search capabilities for retrieval.  
    * **Persistence:** Yes. The knowledge is stored persistently within the configured Vertex AI RAG Corpus.  
    * **Requires:** A Google Cloud project, appropriate permissions, necessary SDKs (`pip install google-adk[vertexai]`), and a pre-configured Vertex AI RAG Corpus resource name/ID.  
    * **Best for:** Production applications needing scalable, persistent, and semantically relevant knowledge retrieval, especially when deployed on Google Cloud.

    ```py
    # Requires: pip install google-adk[vertexai]
    # Plus GCP setup, RAG Corpus, and authentication
    from google.adk.memory import VertexAiRagMemoryService

    # The RAG Corpus name or ID
    RAG_CORPUS_RESOURCE_NAME = "projects/your-gcp-project-id/locations/us-central1/ragCorpora/your-corpus-id"
    # Optional configuration for retrieval
    SIMILARITY_TOP_K = 5
    VECTOR_DISTANCE_THRESHOLD = 0.7

    memory_service = VertexAiRagMemoryService(
        rag_corpus=RAG_CORPUS_RESOURCE_NAME,
        similarity_top_k=SIMILARITY_TOP_K,
        vector_distance_threshold=VECTOR_DISTANCE_THRESHOLD
    )
    ```

## How Memory Works in Practice

The typical workflow involves these steps:

1. **Session Interaction:** A user interacts with an agent via a `Session`, managed by a `SessionService`. Events are added, and state might be updated.  
2. **Ingestion into Memory:** At some point (often when a session is considered complete or has yielded significant information), your application calls `memory_service.add_session_to_memory(session)`. This extracts relevant information from the session's events and adds it to the long-term knowledge store (in-memory dictionary or RAG Corpus).  
3. **Later Query:** In a *different* (or the same) session, the user might ask a question requiring past context (e.g., "What did we discuss about project X last week?").  
4. **Agent Uses Memory Tool:** An agent equipped with a memory-retrieval tool (like the built-in `load_memory` tool) recognizes the need for past context. It calls the tool, providing a search query (e.g., "discussion project X last week").  
5. **Search Execution:** The tool internally calls `memory_service.search_memory(app_name, user_id, query)`.  
6. **Results Returned:** The `MemoryService` searches its store (using keyword matching or semantic search) and returns relevant snippets as a `SearchMemoryResponse` containing a list of `MemoryResult` objects (each potentially holding events from a relevant past session).  
7. **Agent Uses Results:** The tool returns these results to the agent, usually as part of the context or function response. The agent can then use this retrieved information to formulate its final answer to the user.

## Example: Adding and Searching Memory

This example demonstrates the basic flow using the `InMemory` services for simplicity.

???+ "Full Code"

    ```py
    import asyncio
    from google.adk.agents import LlmAgent
    from google.adk.sessions import InMemorySessionService, Session
    from google.adk.memory import InMemoryMemoryService # Import MemoryService
    from google.adk.runners import Runner
    from google.adk.tools import load_memory # Tool to query memory
    from google.genai.types import Content, Part

    # --- Constants ---
    APP_NAME = "memory_example_app"
    USER_ID = "mem_user"
    MODEL = "gemini-2.5-flash" # Use a valid model

    # --- Agent Definitions ---
    # Agent 1: Simple agent to capture information
    info_capture_agent = LlmAgent(
        model=MODEL,
        name="InfoCaptureAgent",
        instruction="Acknowledge the user's statement.",
        # output_key="captured_info" # Could optionally save to state too
    )

    # Agent 2: Agent that can use memory
    memory_recall_agent = LlmAgent(
        model=MODEL,
        name="MemoryRecallAgent",
        instruction="Answer the user's question. Use the 'load_memory' tool "
                    "if the answer might be in past conversations.",
        tools=[load_memory] # Give the agent the tool
    )

    # --- Services and Runner ---
    session_service = InMemorySessionService()
    memory_service = InMemoryMemoryService() # Use in-memory for demo

    runner = Runner(
        # Start with the info capture agent
        agent=info_capture_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service # Provide the memory service to the Runner
    )

    # --- Scenario ---

    # Turn 1: Capture some information in a session
    print("--- Turn 1: Capturing Information ---")
    session1_id = "session_info"
    session1 = await runner.session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=session1_id)
    user_input1 = Content(parts=[Part(text="My favorite project is Project Alpha.")], role="user")

    # Run the agent
    final_response_text = "(No final response)"
    async for event in runner.run_async(user_id=USER_ID, session_id=session1_id, new_message=user_input1):
        if event.is_final_response() and event.content and event.content.parts:
            final_response_text = event.content.parts[0].text
    print(f"Agent 1 Response: {final_response_text}")

    # Get the completed session
    completed_session1 = await runner.session_service.get_session(app_name=APP_NAME, user_id=USER_ID, session_id=session1_id)

    # Add this session's content to the Memory Service
    print("\n--- Adding Session 1 to Memory ---")
    memory_service = await memory_service.add_session_to_memory(completed_session1)
    print("Session added to memory.")

    # Turn 2: In a *new* (or same) session, ask a question requiring memory
    print("\n--- Turn 2: Recalling Information ---")
    session2_id = "session_recall" # Can be same or different session ID
    session2 = await runner.session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=session2_id)

    # Switch runner to the recall agent
    runner.agent = memory_recall_agent
    user_input2 = Content(parts=[Part(text="What is my favorite project?")], role="user")

    # Run the recall agent
    print("Running MemoryRecallAgent...")
    final_response_text_2 = "(No final response)"
    async for event in runner.run_async(user_id=USER_ID, session_id=session2_id, new_message=user_input2):
        print(f"  Event: {event.author} - Type: {'Text' if event.content and event.content.parts and event.content.parts[0].text else ''}"
            f"{'FuncCall' if event.get_function_calls() else ''}"
            f"{'FuncResp' if event.get_function_responses() else ''}")
        if event.is_final_response() and event.content and event.content.parts:
            final_response_text_2 = event.content.parts[0].text
            print(f"Agent 2 Final Response: {final_response_text_2}")
            break # Stop after final response

    # Expected Event Sequence for Turn 2:
    # 1. User sends "What is my favorite project?"
    # 2. Agent (LLM) decides to call `load_memory` tool with a query like "favorite project".
    # 3. Runner executes the `load_memory` tool, which calls `memory_service.search_memory`.
    # 4. `InMemoryMemoryService` finds the relevant text ("My favorite project is Project Alpha.") from session1.
    # 5. Tool returns this text in a FunctionResponse event.
    # 6. Agent (LLM) receives the function response, processes the retrieved text.
    # 7. Agent generates the final answer (e.g., "Your favorite project is Project Alpha.").
    ```


# Session: Tracking Individual Conversations

Following our Introduction, let's dive into the `Session`. Think back to the
idea of a "conversation thread." Just like you wouldn't start every text message
from scratch, agents need context regarding the ongoing interaction.
**`Session`** is the ADK object designed specifically to track and manage these
individual conversation threads.

## The `Session` Object

When a user starts interacting with your agent, the `SessionService` creates a
`Session` object (`google.adk.sessions.Session`). This object acts as the
container holding everything related to that *one specific chat thread*. Here
are its key properties:

*   **Identification (`id`, `appName`, `userId`):** Unique labels for the
    conversation.
    * `id`: A unique identifier for *this specific* conversation thread, essential for retrieving it later. A SessionService object can handle multiple `Session`(s). This field identifies which particular session object are we referring to. For example, "test_id_modification".
    * `app_name`: Identifies which agent application this conversation belongs to. For example, "id_modifier_workflow". 
    *   `userId`: Links the conversation to a particular user.
*   **History (`events`):** A chronological sequence of all interactions
    (`Event` objects – user messages, agent responses, tool actions) that have
    occurred within this specific thread.
*   **Session State (`state`):** A place to store temporary data relevant *only*
    to this specific, ongoing conversation. This acts as a scratchpad for the
    agent during the interaction. We will cover how to use and manage `state` in
    detail in the next section.
*   **Activity Tracking (`lastUpdateTime`):** A timestamp indicating the last
    time an event occurred in this conversation thread.

### Example: Examining Session Properties


=== "Python"

       ```py
        from google.adk.sessions import InMemorySessionService, Session
    
        # Create a simple session to examine its properties
        temp_service = InMemorySessionService()
        example_session = await temp_service.create_session(
            app_name="my_app",
            user_id="example_user",
            state={"initial_key": "initial_value"} # State can be initialized
        )

        print(f"--- Examining Session Properties ---")
        print(f"ID (`id`):                {example_session.id}")
        print(f"Application Name (`app_name`): {example_session.app_name}")
        print(f"User ID (`user_id`):         {example_session.user_id}")
        print(f"State (`state`):           {example_session.state}") # Note: Only shows initial state here
        print(f"Events (`events`):         {example_session.events}") # Initially empty
        print(f"Last Update (`last_update_time`): {example_session.last_update_time:.2f}")
        print(f"---------------------------------")

        # Clean up (optional for this example)
        temp_service = await temp_service.delete_session(app_name=example_session.app_name,
                                    user_id=example_session.user_id, session_id=example_session.id)
        print("The final status of temp_service - ", temp_service)
       ```

=== "Java"

       

*(**Note:** The state shown above is only the initial state. State updates
happen via events, as discussed in the State section.)*

## Managing Sessions with a `SessionService`

As seen above, you don't typically create or manage `Session` objects directly.
Instead, you use a **`SessionService`**. This service acts as the central
manager responsible for the entire lifecycle of your conversation sessions.

Its core responsibilities include:

*   **Starting New Conversations:** Creating fresh `Session` objects when a user
    begins an interaction.
*   **Resuming Existing Conversations:** Retrieving a specific `Session` (using
    its ID) so the agent can continue where it left off.
*   **Saving Progress:** Appending new interactions (`Event` objects) to a
    session's history. This is also the mechanism through which session `state`
    gets updated (more in the `State` section).
*   **Listing Conversations:** Finding the active session threads for a
    particular user and application.
*   **Cleaning Up:** Deleting `Session` objects and their associated data when
    conversations are finished or no longer needed.

## `SessionService` Implementations

ADK provides different `SessionService` implementations, allowing you to choose
the storage backend that best suits your needs:

1.  **`InMemorySessionService`**

    *   **How it works:** Stores all session data directly in the application's
        memory.
    *   **Persistence:** None. **All conversation data is lost if the
        application restarts.**
    *   **Requires:** Nothing extra.
    *   **Best for:** Quick development, local testing, examples, and scenarios
        where long-term persistence isn't required.

    === "Python"
    
           ```py
            from google.adk.sessions import InMemorySessionService
            session_service = InMemorySessionService()
           ```
    === "Java"
    
           

2.  **`VertexAiSessionService`**

    *   **How it works:** Uses Google Cloud's Vertex AI infrastructure via API
        calls for session management.
    *   **Persistence:** Yes. Data is managed reliably and scalably via
        [Vertex AI Agent Engine](https://google.github.io/adk-docs/deploy/agent-engine/).
    *   **Requires:**
        *   A Google Cloud project (`pip install vertexai`)
        *   A Google Cloud storage bucket that can be configured by this
            [step](https://cloud.google.com/vertex-ai/docs/pipelines/configure-project#storage).
        *   A Reasoning Engine resource name/ID that can setup following this
            [tutorial](https://google.github.io/adk-docs/deploy/agent-engine/).
    *   **Best for:** Scalable production applications deployed on Google Cloud,
        especially when integrating with other Vertex AI features.

    === "Python"
    
           ```py
           # Requires: pip install google-adk[vertexai]
           # Plus GCP setup and authentication
           from google.adk.sessions import VertexAiSessionService

           PROJECT_ID = "your-gcp-project-id"
           LOCATION = "us-central1"
           # The app_name used with this service should be the Reasoning Engine ID or name
           REASONING_ENGINE_APP_NAME = "projects/your-gcp-project-id/locations/us-central1/reasoningEngines/your-engine-id"

           session_service = VertexAiSessionService(project=PROJECT_ID, location=LOCATION)
           # Use REASONING_ENGINE_APP_NAME when calling service methods, e.g.:
           # session_service = await session_service.create_session(app_name=REASONING_ENGINE_APP_NAME, ...)
           ```
       
    === "Java"
    
           

3.  **`DatabaseSessionService`**

    ![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

    *   **How it works:** Connects to a relational database (e.g., PostgreSQL,
        MySQL, SQLite) to store session data persistently in tables.
    *   **Persistence:** Yes. Data survives application restarts.
    *   **Requires:** A configured database.
    *   **Best for:** Applications needing reliable, persistent storage that you
        manage yourself.

    ```py
    from google.adk.sessions import DatabaseSessionService
    # Example using a local SQLite file:
    db_url = "sqlite:///./my_agent_data.db"
    session_service = DatabaseSessionService(db_url=db_url)
    ```

Choosing the right `SessionService` is key to defining how your agent's
conversation history and temporary data are stored and persist.

## The Session Lifecycle

<img src="../../assets/session_lifecycle.png" alt="Session lifecycle">

Here’s a simplified flow of how `Session` and `SessionService` work together
during a conversation turn:

1.  **Start or Resume:** Your application's `Runner` uses the `SessionService`
    to either `create_session` (for a new chat) or `get_session` (to retrieve an
    existing one).
2.  **Context Provided:** The `Runner` gets the appropriate `Session` object
    from the appropriate service method, providing the agent with access to the
    corresponding Session's `state` and `events`.
3.  **Agent Processing:** The user prompts the agent with a query. The agent
    analyzes the query and potentially the session `state` and `events` history
    to determine the response.
4.  **Response & State Update:** The agent generates a response (and potentially
    flags data to be updated in the `state`). The `Runner` packages this as an
    `Event`.
5.  **Save Interaction:** The `Runner` calls
    `sessionService.append_event(session, event)` with the `session` and the new
    `event` as the arguments. The service adds the `Event` to the history and
    updates the session's `state` in storage based on information within the
    event. The session's `last_update_time` also get updated.
6.  **Ready for Next:** The agent's response goes to the user. The updated
    `Session` is now stored by the `SessionService`, ready for the next turn
    (which restarts the cycle at step 1, usually with the continuation of the
    conversation in the current session).
7.  **End Conversation:** When the conversation is over, your application calls
    `sessionService.delete_session(...)` to clean up the stored session data if
    it is no longer required.

This cycle highlights how the `SessionService` ensures conversational continuity
by managing the history and state associated with each `Session` object.


# State: The Session's Scratchpad

Within each `Session` (our conversation thread), the **`state`** attribute acts like the agent's dedicated scratchpad for that specific interaction. While `session.events` holds the full history, `session.state` is where the agent stores and updates dynamic details needed *during* the conversation.

## What is `session.state`?

Conceptually, `session.state` is a collection (dictionary or Map) holding key-value pairs. It's designed for information the agent needs to recall or track to make the current conversation effective:

* **Personalize Interaction:** Remember user preferences mentioned earlier (e.g., `'user_preference_theme': 'dark'`).
* **Track Task Progress:** Keep tabs on steps in a multi-turn process (e.g., `'booking_step': 'confirm_payment'`).
* **Accumulate Information:** Build lists or summaries (e.g., `'shopping_cart_items': ['book', 'pen']`).
* **Make Informed Decisions:** Store flags or values influencing the next response (e.g., `'user_is_authenticated': True`).

### Key Characteristics of `State`

1. **Structure: Serializable Key-Value Pairs**

    * Data is stored as `key: value`.
    * **Keys:** Always strings (`str`). Use clear names (e.g., `'departure_city'`, `'user:language_preference'`).
    * **Values:** Must be **serializable**. This means they can be easily saved and loaded by the `SessionService`. Stick to basic types in the specific languages (Python/ Java) like strings, numbers, booleans, and simple lists or dictionaries containing *only* these basic types. (See API documentation for precise details).
    * **⚠️ Avoid Complex Objects:** **Do not store non-serializable objects** (custom class instances, functions, connections, etc.) directly in the state. Store simple identifiers if needed, and retrieve the complex object elsewhere.

2. **Mutability: It Changes**

    * The contents of the `state` are expected to change as the conversation evolves.

3. **Persistence: Depends on `SessionService`**

    * Whether state survives application restarts depends on your chosen service:
      * `InMemorySessionService`: **Not Persistent.** State is lost on restart.
      * `DatabaseSessionService` / `VertexAiSessionService`: **Persistent.** State is saved reliably.

!!! Note
    The specific parameters or method names for the primitives may vary slightly by SDK language (e.g., `session.state['current_intent'] = 'book_flight'` in Python, `session.state().put("current_intent", "book_flight)` in Java). Refer to the language-specific API documentation for details.

### Organizing State with Prefixes: Scope Matters

Prefixes on state keys define their scope and persistence behavior, especially with persistent services:

* **No Prefix (Session State):**

    * **Scope:** Specific to the *current* session (`id`).
    * **Persistence:** Only persists if the `SessionService` is persistent (`Database`, `VertexAI`).
    * **Use Cases:** Tracking progress within the current task (e.g., `'current_booking_step'`), temporary flags for this interaction (e.g., `'needs_clarification'`).
    * **Example:** `session.state['current_intent'] = 'book_flight'`

* **`user:` Prefix (User State):**

    * **Scope:** Tied to the `user_id`, shared across *all* sessions for that user (within the same `app_name`).
    * **Persistence:** Persistent with `Database` or `VertexAI`. (Stored by `InMemory` but lost on restart).
    * **Use Cases:** User preferences (e.g., `'user:theme'`), profile details (e.g., `'user:name'`).
    * **Example:** `session.state['user:preferred_language'] = 'fr'`

* **`app:` Prefix (App State):**

    * **Scope:** Tied to the `app_name`, shared across *all* users and sessions for that application.
    * **Persistence:** Persistent with `Database` or `VertexAI`. (Stored by `InMemory` but lost on restart).
    * **Use Cases:** Global settings (e.g., `'app:api_endpoint'`), shared templates.
    * **Example:** `session.state['app:global_discount_code'] = 'SAVE10'`

* **`temp:` Prefix (Temporary Session State):**

    * **Scope:** Specific to the *current* session processing turn.
    * **Persistence:** **Never Persistent.** Guaranteed to be discarded, even with persistent services.
    * **Use Cases:** Intermediate results needed only immediately, data you explicitly don't want stored.
    * **Example:** `session.state['temp:raw_api_response'] = {...}`

**How the Agent Sees It:** Your agent code interacts with the *combined* state through the single `session.state` collection (dict/ Map). The `SessionService` handles fetching/merging state from the correct underlying storage based on prefixes.

### How State is Updated: Recommended Methods

State should **always** be updated as part of adding an `Event` to the session history using `session_service.append_event()`. This ensures changes are tracked, persistence works correctly, and updates are thread-safe.

**1\. The Easy Way: `output_key` (for Agent Text Responses)**

This is the simplest method for saving an agent's final text response directly into the state. When defining your `LlmAgent`, specify the `output_key`:

=== "Python"

    ```py
    from google.adk.agents import LlmAgent
    from google.adk.sessions import InMemorySessionService, Session
    from google.adk.runners import Runner
    from google.genai.types import Content, Part

    # Define agent with output_key
    greeting_agent = LlmAgent(
        name="Greeter",
        model="gemini-2.5-flash", # Use a valid model
        instruction="Generate a short, friendly greeting.",
        output_key="last_greeting" # Save response to state['last_greeting']
    )

    # --- Setup Runner and Session ---
    app_name, user_id, session_id = "state_app", "user1", "session1"
    session_service = InMemorySessionService()
    runner = Runner(
        agent=greeting_agent,
        app_name=app_name,
        session_service=session_service
    )
    session = await session_service.create_session(app_name=app_name,
                                        user_id=user_id,
                                        session_id=session_id)
    print(f"Initial state: {session.state}")

    # --- Run the Agent ---
    # Runner handles calling append_event, which uses the output_key
    # to automatically create the state_delta.
    user_message = Content(parts=[Part(text="Hello")])
    for event in runner.run(user_id=user_id,
                            session_id=session_id,
                            new_message=user_message):
        if event.is_final_response():
          print(f"Agent responded.") # Response text is also in event.content

    # --- Check Updated State ---
    updated_session = await session_service.get_session(app_name=APP_NAME, user_id=USER_ID, session_id=session_id)
    print(f"State after agent run: {updated_session.state}")
    # Expected output might include: {'last_greeting': 'Hello there! How can I help you today?'}
    ```

=== "Java"

    

Behind the scenes, the `Runner` uses the `output_key` to create the necessary `EventActions` with a `state_delta` and calls `append_event`.

**2\. The Standard Way: `EventActions.state_delta` (for Complex Updates)**

For more complex scenarios (updating multiple keys, non-string values, specific scopes like `user:` or `app:`, or updates not tied directly to the agent's final text), you manually construct the `state_delta` within `EventActions`.

=== "Python"

    ```py
    from google.adk.sessions import InMemorySessionService, Session
    from google.adk.events import Event, EventActions
    from google.genai.types import Part, Content
    import time

    # --- Setup ---
    session_service = InMemorySessionService()
    app_name, user_id, session_id = "state_app_manual", "user2", "session2"
    session = await session_service.create_session(
        app_name=app_name,
        user_id=user_id,
        session_id=session_id,
        state={"user:login_count": 0, "task_status": "idle"}
    )
    print(f"Initial state: {session.state}")

    # --- Define State Changes ---
    current_time = time.time()
    state_changes = {
        "task_status": "active",              # Update session state
        "user:login_count": session.state.get("user:login_count", 0) + 1, # Update user state
        "user:last_login_ts": current_time,   # Add user state
        "temp:validation_needed": True        # Add temporary state (will be discarded)
    }

    # --- Create Event with Actions ---
    actions_with_update = EventActions(state_delta=state_changes)
    # This event might represent an internal system action, not just an agent response
    system_event = Event(
        invocation_id="inv_login_update",
        author="system", # Or 'agent', 'tool' etc.
        actions=actions_with_update,
        timestamp=current_time
        # content might be None or represent the action taken
    )

    # --- Append the Event (This updates the state) ---
    await session_service.append_event(session, system_event)
    print("`append_event` called with explicit state delta.")

    # --- Check Updated State ---
    updated_session = await session_service.get_session(app_name=app_name,
                                                user_id=user_id,
                                                session_id=session_id)
    print(f"State after event: {updated_session.state}")
    # Expected: {'user:login_count': 1, 'task_status': 'active', 'user:last_login_ts': <timestamp>}
    # Note: 'temp:validation_needed' is NOT present.
    ```

=== "Java"

    

**3. Via `CallbackContext` or `ToolContext` (Recommended for Callbacks and Tools)**

Modifying state within agent callbacks (e.g., `on_before_agent_call`, `on_after_agent_call`) or tool functions is best done using the `state` attribute of the `CallbackContext` or `ToolContext` provided to your function.

*   `callback_context.state['my_key'] = my_value`
*   `tool_context.state['my_key'] = my_value`

These context objects are specifically designed to manage state changes within their respective execution scopes. When you modify `context.state`, the ADK framework ensures that these changes are automatically captured and correctly routed into the `EventActions.state_delta` for the event being generated by the callback or tool. This delta is then processed by the `SessionService` when the event is appended, ensuring proper persistence and tracking.

This method abstracts away the manual creation of `EventActions` and `state_delta` for most common state update scenarios within callbacks and tools, making your code cleaner and less error-prone.

For more comprehensive details on context objects, refer to the [Context documentation](../context/index.md).

=== "Python"

    ```python
    # In an agent callback or tool function
    from google.adk.agents import CallbackContext # or ToolContext

    def my_callback_or_tool_function(context: CallbackContext, # Or ToolContext
                                     # ... other parameters ...
                                    ):
        # Update existing state
        count = context.state.get("user_action_count", 0)
        context.state["user_action_count"] = count + 1

        # Add new state
        context.state["temp:last_operation_status"] = "success"

        # State changes are automatically part of the event's state_delta
        # ... rest of callback/tool logic ...
    ```

=== "Java"

    

**What `append_event` Does:**

* Adds the `Event` to `session.events`.
* Reads the `state_delta` from the event's `actions`.
* Applies these changes to the state managed by the `SessionService`, correctly handling prefixes and persistence based on the service type.
* Updates the session's `last_update_time`.
* Ensures thread-safety for concurrent updates.

### ⚠️ A Warning About Direct State Modification

Avoid directly modifying the `session.state` collection (dictionary/Map) on a `Session` object that was obtained directly from the `SessionService` (e.g., via `session_service.get_session()` or `session_service.create_session()`) *outside* of the managed lifecycle of an agent invocation (i.e., not through a `CallbackContext` or `ToolContext`). For example, code like `retrieved_session = await session_service.get_session(...); retrieved_session.state['key'] = value` is problematic.

State modifications *within* callbacks or tools using `CallbackContext.state` or `ToolContext.state` are the correct way to ensure changes are tracked, as these context objects handle the necessary integration with the event system.

**Why direct modification (outside of contexts) is strongly discouraged:**

1. **Bypasses Event History:** The change isn't recorded as an `Event`, losing auditability.
2. **Breaks Persistence:** Changes made this way **will likely NOT be saved** by `DatabaseSessionService` or `VertexAiSessionService`. They rely on `append_event` to trigger saving.
3. **Not Thread-Safe:** Can lead to race conditions and lost updates.
4. **Ignores Timestamps/Logic:** Doesn't update `last_update_time` or trigger related event logic.

**Recommendation:** Stick to updating state via `output_key`, `EventActions.state_delta` (when manually creating events), or by modifying the `state` property of `CallbackContext` or `ToolContext` objects when within their respective scopes. These methods ensure reliable, trackable, and persistent state management. Use direct access to `session.state` (from a `SessionService`-retrieved session) only for *reading* state.

### Best Practices for State Design Recap

* **Minimalism:** Store only essential, dynamic data.
* **Serialization:** Use basic, serializable types.
* **Descriptive Keys & Prefixes:** Use clear names and appropriate prefixes (`user:`, `app:`, `temp:`, or none).
* **Shallow Structures:** Avoid deep nesting where possible.
* **Standard Update Flow:** Rely on `append_event`.


# Configurating streaming behaviour

There are some configurations you can set for live(streaming) agents. 

It's set by [RunConfig](https://github.com/google/adk-python/blob/main/src/google/adk/agents/run_config.py). You should use RunConfig with your [Runner.run_live(...)](https://github.com/google/adk-python/blob/main/src/google/adk/runners.py). 

For example, if you want to set voice config, you can leverage speech_config. 

```python
voice_config = genai_types.VoiceConfig(
    prebuilt_voice_config=genai_types.PrebuiltVoiceConfigDict(
        voice_name='Aoede'
    )
)
speech_config = genai_types.SpeechConfig(voice_config=voice_config)
run_config = RunConfig(speech_config=speech_config)

runner.run_live(
    ...,
    run_config=run_config,
)
```




# Custom Audio Streaming app (WebSocket) {#custom-streaming-websocket}

This article overviews the server and client code for a custom asynchronous web app built with ADK Streaming and [FastAPI](https://fastapi.tiangolo.com/), enabling real-time, bidirectional audio and text communication with WebSockets.

**Note:** This guide assumes you have experience of JavaScript and Python `asyncio` programming.

## Supported models for voice/video streaming {#supported-models}

In order to use voice/video streaming in ADK, you will need to use Gemini models that support the Live API. You can find the **model ID(s)** that supports the Gemini Live API in the documentation:

- [Google AI Studio: Gemini Live API](https://ai.google.dev/gemini-api/docs/models#live-api)
- [Vertex AI: Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api)

There is also a [SSE](custom-streaming.md) version of the sample is available.

## 1. Install ADK {#1.-setup-installation}

Create & Activate Virtual Environment (Recommended):

```bash
# Create
python -m venv .venv
# Activate (each new terminal)
# macOS/Linux: source .venv/bin/activate
# Windows CMD: .venv\Scripts\activate.bat
# Windows PowerShell: .venv\Scripts\Activate.ps1
```

Install ADK:

```bash
pip install --upgrade google-adk==1.2.1
```

Set `SSL_CERT_FILE` variable with the following command.

```shell
export SSL_CERT_FILE=$(python -m certifi)
```

Download the sample code:

```bash
git clone --no-checkout https://github.com/google/adk-docs.git
cd adk-docs
git sparse-checkout init --cone
git sparse-checkout set examples/python/snippets/streaming/adk-streaming-ws
git checkout main
cd examples/python/snippets/streaming/adk-streaming-ws/app
```

This sample code has the following files and folders:

```console
adk-streaming-ws/
└── app/ # the web app folder
    ├── .env # Gemini API key / Google Cloud Project ID
    ├── main.py # FastAPI web app
    ├── static/ # Static content folder
    |   ├── js # JavaScript files folder (includes app.js)
    |   └── index.html # The web client page
    └── google_search_agent/ # Agent folder
        ├── __init__.py # Python package
        └── agent.py # Agent definition
```

## 2\. Set up the platform {#2.-set-up-the-platform}

To run the sample app, choose a platform from either Google AI Studio or Google Cloud Vertex AI:

=== "Gemini - Google AI Studio"
    1. Get an API key from [Google AI Studio](https://aistudio.google.com/apikey).
    2. Open the **`.env`** file located inside (`app/`) and copy-paste the following code.

        ```env title=".env"
        GOOGLE_GENAI_USE_VERTEXAI=FALSE
        GOOGLE_API_KEY=PASTE_YOUR_ACTUAL_API_KEY_HERE
        ```

    3. Replace `PASTE_YOUR_ACTUAL_API_KEY_HERE` with your actual `API KEY`.

=== "Gemini - Google Cloud Vertex AI"
    1. You need an existing
       [Google Cloud](https://cloud.google.com/?e=48754805&hl=en) account and a
       project.
        * Set up a
          [Google Cloud project](https://cloud.google.com/vertex-ai/generative-ai/docs/start/quickstarts/quickstart-multimodal#setup-gcp)
        * Set up the
          [gcloud CLI](https://cloud.google.com/vertex-ai/generative-ai/docs/start/quickstarts/quickstart-multimodal#setup-local)
        * Authenticate to Google Cloud, from the terminal by running
          `gcloud auth login`.
        * [Enable the Vertex AI API](https://console.cloud.google.com/flows/enableapi?apiid=aiplatform.googleapis.com).
    2. Open the **`.env`** file located inside (`app/`). Copy-paste
       the following code and update the project ID and location.

        ```env title=".env"
        GOOGLE_GENAI_USE_VERTEXAI=TRUE
        GOOGLE_CLOUD_PROJECT=PASTE_YOUR_ACTUAL_PROJECT_ID
        GOOGLE_CLOUD_LOCATION=us-central1
        ```


### agent.py

The agent definition code `agent.py` in the `google_search_agent` folder is where the agent's logic is written:


```py
from google.adk.agents import Agent
from google.adk.tools import google_search  # Import the tool

root_agent = Agent(
   name="google_search_agent",
   model="gemini-2.5-flash-exp", # if this model does not work, try below
   #model="gemini-2.5-flash-live-001",
   description="Agent to answer questions using Google Search.",
   instruction="Answer the question using the Google Search tool.",
   tools=[google_search],
)
```

**Note:**  To enable both text and audio/video input, the model must support the generateContent (for text) and bidiGenerateContent methods. Verify these capabilities by referring to the [List Models Documentation](https://ai.google.dev/api/models#method:-models.list). This quickstart utilizes the gemini-2.5-flash-exp model for demonstration purposes.

Notice how easily you integrated [grounding with Google Search](https://ai.google.dev/gemini-api/docs/grounding?lang=python#configure-search) capabilities.  The `Agent` class and the `google_search` tool handle the complex interactions with the LLM and grounding with the search API, allowing you to focus on the agent's *purpose* and *behavior*.

![intro_components.png](../assets/quickstart-streaming-tool.png)

## 3\. Interact with Your Streaming app {#3.-interact-with-your-streaming-app}

1\. **Navigate to the Correct Directory:**

   To run your agent effectively, make sure you are in the **app folder (`adk-streaming-ws/app`)**

2\. **Start the Fast API**: Run the following command to start CLI interface with

```console
uvicorn main:app --reload
```

3\. **Access the app with the text mode:** Once the app starts, the terminal will display a local URL (e.g., [http://localhost:8000](http://localhost:8000)). Click this link to open the UI in your browser.

Now you should see the UI like this:

![ADK Streaming app](../assets/adk-streaming-text.png)

Try asking a question `What time is it now?`. The agent will use Google Search to respond to your queries. You would notice that the UI shows the agent's response as streaming text. You can also send messages to the agent at any time, even while the agent is still responding. This demonstrates the bidirectional communication capability of ADK Streaming.

4\. **Access the app with the audio mode:** Now click the `Start Audio` button. The app reconnects with the server in an audio mode, and the UI will show the following dialog for the first time:

![ADK Streaming app](../assets/adk-streaming-audio-dialog.png)

Click `Allow while visiting the site`, then you will see the microphone icon will be shown at the top of the browser:

![ADK Streaming app](../assets/adk-streaming-mic.png)

Now you can talk to the agent with voice. Ask questions like `What time is it now?` with voice and you will hear the agent responding in voice too. As Streaming for ADK supports [multiple languages](https://ai.google.dev/gemini-api/docs/live#supported-languages), it can also respond to question in the supported languages.

5\. **Check console logs**

If you are using the Chrome browser, use the right click and select `Inspect` to open the DevTools. On the `Console`, you can see the incoming and outgoing audio data such as `[CLIENT TO AGENT]` and `[AGENT TO CLIENT]`, representing the audio data streaming in and out between the browser and the server.

At the same time, in the app server console, you should see something like this:

```
INFO:     ('127.0.0.1', 50068) - "WebSocket /ws/70070018?is_audio=true" [accepted]
Client #70070018 connected, audio mode: true
INFO:     connection open
INFO:     127.0.0.1:50061 - "GET /static/js/pcm-player-processor.js HTTP/1.1" 200 OK
INFO:     127.0.0.1:50060 - "GET /static/js/pcm-recorder-processor.js HTTP/1.1" 200 OK
[AGENT TO CLIENT]: audio/pcm: 9600 bytes.
INFO:     127.0.0.1:50082 - "GET /favicon.ico HTTP/1.1" 404 Not Found
[AGENT TO CLIENT]: audio/pcm: 11520 bytes.
[AGENT TO CLIENT]: audio/pcm: 11520 bytes.
```

These console logs are important in case you develop your own streaming application. In many cases, the communication failure between the browser and server becomes a major cause for the streaming application bugs.

6\. **Troubleshooting tips**

- **When `ws://` doesn't work:** If you see any errors on the Chrome DevTools with regard to `ws://` connection, try replacing `ws://` with `wss://` on `app/static/js/app.js` at line 28. This may happen when you are running the sample on a cloud environment and using a proxy connection to connect from your browser.
- **When `gemini-2.5-flash-exp` model doesn't work:** If you see any errors on the app server console with regard to `gemini-2.5-flash-exp` model availability, try replacing it with `gemini-2.5-flash-live-001` on `app/google_search_agent/agent.py` at line 6.

## 4. Server code overview {#4.-server-side-code-overview}

This server app enables real-time, streaming interaction with ADK agent via WebSockets. Clients send text/audio to the ADK agent and receive streamed text/audio responses.

Core functions:
1.  Initialize/manage ADK agent sessions.
2.  Handle client WebSocket connections.
3.  Relay client messages to the ADK agent.
4.  Stream ADK agent responses (text/audio) to clients.

### ADK Streaming Setup

```py
import os
import json
import asyncio
import base64

from pathlib import Path
from dotenv import load_dotenv

from google.genai.types import (
    Part,
    Content,
    Blob,
)

from google.adk.runners import Runner
from google.adk.agents import LiveRequestQueue
from google.adk.agents.run_config import RunConfig
from google.adk.sessions.in_memory_session_service import InMemorySessionService

from fastapi import FastAPI, WebSocket
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

from google_search_agent.agent import root_agent
```

*   **Imports:** Includes standard Python libraries, `dotenv` for environment variables, Google ADK, and FastAPI.
*   **`load_dotenv()`:** Loads environment variables.
*   **`APP_NAME`**: Application identifier for ADK.
*   **`session_service = InMemorySessionService()`**: Initializes an in-memory ADK session service, suitable for single-instance or development use. Production might use a persistent store.

### `start_agent_session(session_id, is_audio=False)`

```py
async def start_agent_session(user_id, is_audio=False):
    """Starts an agent session"""

    # Create a Runner
    runner = InMemoryRunner(
        app_name=APP_NAME,
        agent=root_agent,
    )

    # Create a Session
    session = await runner.session_service.create_session(
        app_name=APP_NAME,
        user_id=user_id,  # Replace with actual user ID
    )

    # Set response modality
    modality = "AUDIO" if is_audio else "TEXT"
    run_config = RunConfig(response_modalities=[modality])

    # Create a LiveRequestQueue for this session
    live_request_queue = LiveRequestQueue()

    # Start agent session
    live_events = runner.run_live(
        session=session,
        live_request_queue=live_request_queue,
        run_config=run_config,
    )
    return live_events, live_request_queue
```

This function initializes an ADK agent live session.

| Parameter    | Type    | Description                                             |
|--------------|---------|---------------------------------------------------------|
| `user_id` | `str`   | Unique client identifier.                       |
| `is_audio`   | `bool`  | `True` for audio responses, `False` for text (default). |

**Key Steps:**
1\.  **Create Runner:** Instantiates the ADK runner for the `root_agent`.
2\.  **Create Session:** Establishes an ADK session.
3\.  **Set Response Modality:** Configures agent response as "AUDIO" or "TEXT".
4\.  **Create LiveRequestQueue:** Creates a queue for client inputs to the agent.
