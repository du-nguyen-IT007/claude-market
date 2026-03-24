
### 403 Permission Denied for `Gemini 2.0 Flash`

This usually means that the Kubernetes service account does not have the necessary permission to access the Vertex AI API. Ensure that you have created the service account and bound it to the `Vertex AI User` role as described in the [Configure Kubernetes Service Account for Vertex AI](#configure-kubernetes-service-account-for-vertex-ai) section. If you are using AI Studio, ensure that you have set the `GOOGLE_API_KEY` environment variable in the deployment manifest and it is valid.

### Attempt to write a readonly database

You might see there is no session id created in the UI and the agent does not respond to any messages. This is usually caused by the SQLite database being read-only. This can happen if you run the agent locally and then create the container image which copies the SQLite database into the container. The database is then read-only in the container.

```bash
sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) attempt to write a readonly database
[SQL: UPDATE app_states SET state=?, update_time=CURRENT_TIMESTAMP WHERE app_states.app_name = ?]
```

To fix this issue, you can either:

Delete the SQLite database file from your local machine before building the container image. This will create a new SQLite database when the container is started.

```bash
rm -f sessions.db
```

or (recommended) you can add a `.dockerignore` file to your project directory to exclude the SQLite database from being copied into the container image.

```txt title=".dockerignore"
sessions.db
```

Build the container image abd deploy the application again.

## Cleanup

To delete the GKE cluster and all associated resources, run:

```bash
gcloud container clusters delete adk-cluster \
    --location=$GOOGLE_CLOUD_LOCATION \
    --project=$GOOGLE_CLOUD_PROJECT
```

To delete the Artifact Registry repository, run:

```bash
gcloud artifacts repositories delete adk-repo \
    --location=$GOOGLE_CLOUD_LOCATION \
    --project=$GOOGLE_CLOUD_PROJECT
```

You can also delete the project if you no longer need it. This will delete all resources associated with the project, including the GKE cluster, Artifact Registry repository, and any other resources you created.

```bash
gcloud projects delete $GOOGLE_CLOUD_PROJECT
```


# Deploying Your Agent

Once you've built and tested your agent using ADK,
the next step is to deploy it so it can be accessed, queried, and used in
production or integrated with other applications. Deployment moves your agent
from your local development machine to a scalable and reliable environment.

<img src="../assets/deploy-agent.png" alt="Deploying your agent">

## Deployment Options

Your ADK agent can be deployed to a range of different environments based
on your needs for production readiness or custom flexibility:

### Agent Engine in Vertex AI

[Agent Engine](agent-engine.md) is a fully managed auto-scaling service on Google Cloud
specifically designed for deploying, managing, and scaling AI agents built with
frameworks such as ADK.

Learn more about [deploying your agent to Vertex AI Agent Engine](agent-engine.md).

### Cloud Run

[Cloud Run](https://cloud.google.com/run) is a managed auto-scaling compute platform on
Google Cloud that enables you to run your agent as a container-based
application.

Learn more about [deploying your agent to Cloud Run](cloud-run.md).

### Google Kubernetes Engine (GKE)

[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) is a managed
Kubernetes service of Google Cloud that allows you to run your agent in a containerized
environment. GKE is a good option if you need more control over the deployment as well as
for running Open Models.

Learn more about [deploying your agent to GKE](gke.md).


# Why Evaluate Agents

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="This feature is currently available for Python. Java support is planned/ coming soon."}

In traditional software development, unit tests and integration tests provide confidence that code functions as expected and remains stable through changes. These tests provide a clear "pass/fail" signal, guiding further development. However, LLM agents introduce a level of variability that makes traditional testing approaches insufficient.

Due to the probabilistic nature of models, deterministic "pass/fail" assertions are often unsuitable for evaluating agent performance. Instead, we need qualitative evaluations of both the final output and the agent's trajectory \- the sequence of steps taken to reach the solution. This involves assessing the quality of the agent's decisions, its reasoning process, and the final result.

This may seem like a lot of extra work to set up, but the investment of automating evaluations pays off quickly. If you intend to progress beyond prototype, this is a highly recommended best practice.

![intro_components.png](../assets/evaluate_agent.png)

## Preparing for Agent Evaluations

Before automating agent evaluations, define clear objectives and success criteria:

* **Define Success:** What constitutes a successful outcome for your agent?  
* **Identify Critical Tasks:** What are the essential tasks your agent must accomplish?  
* **Choose Relevant Metrics:** What metrics will you track to measure performance?

These considerations will guide the creation of evaluation scenarios and enable effective monitoring of agent behavior in real-world deployments.

## What to Evaluate?

To bridge the gap between a proof-of-concept and a production-ready AI agent, a robust and automated evaluation framework is essential. Unlike evaluating generative models, where the focus is primarily on the final output, agent evaluation requires a deeper understanding of the decision-making process. Agent evaluation can be broken down into two components:

1. **Evaluating Trajectory and Tool Use:** Analyzing the steps an agent takes to reach a solution, including its choice of tools, strategies, and the efficiency of its approach.  
2. **Evaluating the Final Response:** Assessing the quality, relevance, and correctness of the agent's final output.

The trajectory is just a list of steps the agent took before it returned to the user. We can compare that against the list of steps we expect the agent to have taken.

### Evaluating trajectory and tool use

Before responding to a user, an agent typically performs a series of actions, which we refer to as a 'trajectory.' It might compare the user input with session history to disambiguate a term, or lookup a policy document, search a knowledge base or invoke an API to save a ticket. We call this a ‘trajectory’ of actions. Evaluating an agent's performance requires comparing its actual trajectory to an expected, or ideal, one. This comparison can reveal errors and inefficiencies in the agent's process. The expected trajectory represents the ground truth \-- the list of steps we anticipate the agent should take.

For example:

```python
# Trajectory evaluation will compare
expected_steps = ["determine_intent", "use_tool", "review_results", "report_generation"]
actual_steps = ["determine_intent", "use_tool", "review_results", "report_generation"]
```

Several ground-truth-based trajectory evaluations exist:

1. **Exact match:** Requires a perfect match to the ideal trajectory.  
2. **In-order match:** Requires the correct actions in the correct order, allows for extra actions.  
3. **Any-order match:** Requires the correct actions in any order, allows for extra actions.  
4. **Precision:** Measures the relevance/correctness of predicted actions.  
5. **Recall:** Measures how many essential actions are captured in the prediction.  
6. **Single-tool use:** Checks for the inclusion of a specific action.

Choosing the right evaluation metric depends on the specific requirements and goals of your agent. For instance, in high-stakes scenarios, an exact match might be crucial, while in more flexible situations, an in-order or any-order match might suffice.

## How Evaluation works with the ADK

The ADK offers two methods for evaluating agent performance against predefined datasets and evaluation criteria. While conceptually similar, they differ in the amount of data they can process, which typically dictates the appropriate use case for each.

### First approach: Using a test file

This approach involves creating individual test files, each representing a single, simple agent-model interaction (a session). It's most effective during active agent development, serving as a form of unit testing. These tests are designed for rapid execution and should focus on simple session complexity. Each test file contains a single session, which may consist of multiple turns. A turn represents a single interaction between the user and the agent. Each turn includes

-   `User Content`: The user issued query.
-   `Expected Intermediate Tool Use Trajectory`: The tool calls we expect the
    agent to make in order to respond correctly to the user query.
-   `Expected Intermediate Agent Responses`: These are the natural language
    responses that the agent (or sub-agents) generates as it moves towards
    generating a final answer. These natural language responses are usually an
    artifact of a multi-agent system, where your root agent depends on sub-agents to achieve a goal. These intermediate responses, may or may not be of
    interest to the end user, but for a developer/owner of the system, are of
    critical importance, as they give you the confidence that the agent went
    through the right path to generate final response.
-   `Final Response`: The expected final response from the agent.

You can give the file any name for example `evaluation.test.json`.The framework only checks for the `.test.json` suffix, and the preceding part of the filename is not constrained. Here is a test file with a few examples:

NOTE: The test files are now backed by a formal Pydantic data model. The two key
schema files are
[Eval Set](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_set.py) and
[Eval Case](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_case.py)

*(Note: Comments are included for explanatory purposes and should be removed for the JSON to be valid.)*

```json
# Do note that some fields are removed for sake of making this doc readable.
{
  "eval_set_id": "home_automation_agent_light_on_off_set",
  "name": "",
  "description": "This is an eval set that is used for unit testing `x` behavior of the Agent",
  "eval_cases": [
    {
      "eval_id": "eval_case_id",
      "conversation": [
        {
          "invocation_id": "b7982664-0ab6-47cc-ab13-326656afdf75", # Unique identifier for the invocation.
          "user_content": { # Content provided by the user in this invocation. This is the query.
            "parts": [
              {
                "text": "Turn off device_2 in the Bedroom."
              }
            ],
            "role": "user"
          },
          "final_response": { # Final response from the agent that acts as a reference of benchmark.
            "parts": [
              {
                "text": "I have set the device_2 status to off."
              }
            ],
            "role": "model"
          },
          "intermediate_data": {
            "tool_uses": [ # Tool use trajectory in chronological order.
              {
                "args": {
                  "location": "Bedroom",
                  "device_id": "device_2",
                  "status": "OFF"
                },
                "name": "set_device_info"
              }
            ],
            "intermediate_responses": [] # Any intermediate sub-agent responses.
          },
        }
      ],
      "session_input": { # Initial session input.
        "app_name": "home_automation_agent",
        "user_id": "test_user",
        "state": {}
      },
    }
  ],
}
```

Test files can be organized into folders. Optionally, a folder can also include a `test_config.json` file that specifies the evaluation criteria.

#### How to migrate test files not backed by the Pydantic schema?

NOTE: If your test files don't adhere to [EvalSet](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_set.py) schema file, then this section is relevant to you.

Please use `AgentEvaluator.migrate_eval_data_to_new_schema` to migrate your
existing `*.test.json` files to the Pydantic backed schema.

The utility takes your current test data file and an optional initial session
file, and generates a single output json file with data serialized in the new
format. Given that the new schema is more cohesive, both the old test data file
and initial session file can be ignored (or removed.)

### Second approach: Using An Evalset File

The evalset approach utilizes a dedicated dataset called an "evalset" for evaluating agent-model interactions. Similar to a test file, the evalset contains example interactions. However, an evalset can contain multiple, potentially lengthy sessions, making it ideal for simulating complex, multi-turn conversations. Due to its ability to represent complex sessions, the evalset is well-suited for integration tests. These tests are typically run less frequently than unit tests due to their more extensive nature.

An evalset file contains multiple "evals," each representing a distinct session. Each eval consists of one or more "turns," which include the user query, expected tool use, expected intermediate agent responses, and a reference response. These fields have the same meaning as they do in the test file approach. Each eval is identified by a unique name. Furthermore, each eval includes an associated initial session state.

Creating evalsets manually can be complex, therefore UI tools are provided to help capture relevant sessions and easily convert them into evals within your evalset. Learn more about using the web UI for evaluation below. Here is an example evalset containing two sessions.

NOTE: The eval set files are now backed by a formal Pydantic data model. The two key
schema files are
[Eval Set](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_set.py) and
[Eval Case](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_case.py)

*(Note: Comments are included for explanatory purposes and should be removed for the JSON to be valid.)*

```json
# Do note that some fields are removed for sake of making this doc readable.
{
  "eval_set_id": "eval_set_example_with_multiple_sessions",
  "name": "Eval set with multiple sessions",
  "description": "This eval set is an example that shows that an eval set can have more than one session.",
  "eval_cases": [
    {
      "eval_id": "session_01",
      "conversation": [
        {
          "invocation_id": "e-0067f6c4-ac27-4f24-81d7-3ab994c28768",
          "user_content": {
            "parts": [
              {
                "text": "What can you do?"
              }
            ],
            "role": "user"
          },
          "final_response": {
            "parts": [
              {

                "text": "I can roll dice of different sizes and check if numbers are prime."
              }
            ],
            "role": null
          },
          "intermediate_data": {
            "tool_uses": [],
            "intermediate_responses": []
          },
        },
      ],
      "session_input": {
        "app_name": "hello_world",
        "user_id": "user",
        "state": {}
      },
    },
    {
      "eval_id": "session_02",
      "conversation": [
        {
          "invocation_id": "e-92d34c6d-0a1b-452a-ba90-33af2838647a",
          "user_content": {
            "parts": [
              {
                "text": "Roll a 19 sided dice"
              }
            ],
            "role": "user"
          },
          "final_response": {
            "parts": [
              {
                "text": "I rolled a 17."
              }
            ],
            "role": null
          },
          "intermediate_data": {
            "tool_uses": [],
            "intermediate_responses": []
          },
        },
        {
          "invocation_id": "e-bf8549a1-2a61-4ecc-a4ee-4efbbf25a8ea",
          "user_content": {
            "parts": [
              {
                "text": "Roll a 10 sided dice twice and then check if 9 is a prime or not"
              }
            ],
            "role": "user"
          },
          "final_response": {
            "parts": [
              {
                "text": "I got 4 and 7 from the dice roll, and 9 is not a prime number.\n"
              }
            ],
            "role": null
          },
          "intermediate_data": {
            "tool_uses": [
              {
                "id": "adk-1a3f5a01-1782-4530-949f-07cf53fc6f05",
                "args": {
                  "sides": 10
                },
                "name": "roll_die"
              },
              {
                "id": "adk-52fc3269-caaf-41c3-833d-511e454c7058",
                "args": {
                  "sides": 10
                },
                "name": "roll_die"
              },
              {
                "id": "adk-5274768e-9ec5-4915-b6cf-f5d7f0387056",
                "args": {
                  "nums": [
                    9
                  ]
                },
                "name": "check_prime"
              }
            ],
            "intermediate_responses": [
              [
                "data_processing_agent",
                [
                  {
                    "text": "I have rolled a 10 sided die twice. The first roll is 5 and the second roll is 3.\n"
                  }
                ]
              ]
            ]
          },
        }
      ],
      "session_input": {
        "app_name": "hello_world",
        "user_id": "user",
        "state": {}
      },
    }
  ],
}
```

#### How to migrate eval set files not backed by the Pydantic schema?

NOTE: If your eval set files don't adhere to [EvalSet](https://github.com/google/adk-python/blob/main/src/google/adk/evaluation/eval_set.py) schema file, then this section is relevant to you.

Based on who is maintaining the eval set data, there are two routes:

1.  **Eval set data maintained by ADK UI** If you use ADK UI to maintain your
    Eval set data then *no action is needed* from you.

2.  **Eval set data is developed and maintained manually and used in ADK eval CLI** A
    migration tool is in the works, until then the ADK eval CLI command will
    continue to support data in the old format.

### Evaluation Criteria

The evaluation criteria define how the agent's performance is measured against the evalset. The following metrics are supported:

* `tool_trajectory_avg_score`: This metric compares the agent's actual tool usage during the evaluation against the expected tool usage defined in the `expected_tool_use` field. Each matching tool usage step receives a score of 1, while a mismatch receives a score of 0\. The final score is the average of these matches, representing the accuracy of the tool usage trajectory.  
* `response_match_score`: This metric compares the agent's final natural language response to the expected final response, stored in the `reference` field. We use the [ROUGE](https://en.wikipedia.org/wiki/ROUGE_\(metric\)) metric to calculate the similarity between the two responses.

If no evaluation criteria are provided, the following default configuration is used:

* `tool_trajectory_avg_score`: Defaults to 1.0, requiring a 100% match in the tool usage trajectory.  
* `response_match_score`: Defaults to 0.8, allowing for a small margin of error in the agent's natural language responses.

Here is an example of a `test_config.json` file specifying custom evaluation criteria:

```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}
```

## How to run Evaluation with the ADK

As a developer, you can evaluate your agents using the ADK in the following ways:

1. **Web-based UI (**`adk web`**):** Evaluate agents interactively through a web-based interface.  
2. **Programmatically (**`pytest`**)**: Integrate evaluation into your testing pipeline using `pytest` and test files.  
3. **Command Line Interface (**`adk eval`**):** Run evaluations on an existing evaluation set file directly from the command line.

### 1\. `adk web` \- Run Evaluations via the Web UI

The web UI provides an interactive way to evaluate agents, generate evaluation datasets, and inspect agent behavior in detail.

#### Step 1: Create and Save a Test Case

1. Start the web server by running: `adk web <path_to_your_agents_folder>`
2. In the web interface, select an agent and interact with it to create a session.
3. Navigate to the **Eval** tab on the right side of the interface.
4. Create a new eval set or select an existing one.
5. Click **"Add current session"** to save the conversation as a new evaluation case.

#### Step 2: View and Edit Your Test Case

Once a case is saved, you can click its ID in the list to inspect it. To make changes, click the **Edit current eval case** icon (pencil). This interactive view allows you to:

* **Modify** agent text responses to refine test scenarios.
* **Delete** individual agent messages from the conversation.
* **Delete** the entire evaluation case if it's no longer needed.

![adk-eval-case.gif](../assets/adk-eval-case.gif)

#### Step 3: Run the Evaluation with Custom Metrics

1. Select one or more test cases from your evalset.
2. Click **Run Evaluation**. An **EVALUATION METRIC** dialog will appear.
3. In the dialog, use the sliders to configure the thresholds for:
    * **Tool trajectory avg score**
    * **Response match score**
4. Click **Start** to run the evaluation using your custom criteria. The evaluation history will record the metrics used for each run.

![adk-eval-config.gif](../assets/adk-eval-config.gif)

#### Step 4: Analyze Results

After the run completes, you can analyze the results:

* **Analyze Run Failures**: Click on any **Pass** or **Fail** result. For failures, you can hover over the `Fail` label to see a side-by-side comparison of the **Actual vs. Expected Output** and the scores that caused the failure.

### Debugging with the Trace View

The ADK web UI includes a powerful **Trace** tab for debugging agent behavior. This feature is available for any agent session, not just during evaluation.

The **Trace** tab provides a detailed and interactive way to inspect your agent's execution flow. Traces are automatically grouped by user message, making it easy to follow the chain of events.

Each trace row is interactive:

* **Hovering** over a trace row highlights the corresponding message in the chat window.
* **Clicking** on a trace row opens a detailed inspection panel with four tabs:
    * **Event**: The raw event data.
    * **Request**: The request sent to the model.
    * **Response**: The response received from the model.
    * **Graph**: A visual representation of the tool calls and agent logic flow.

![adk-trace1.gif](../assets/adk-trace1.gif)
![adk-trace2.gif](../assets/adk-trace2.gif)

Blue rows in the trace view indicate that an event was generated from that interaction. Clicking on these blue rows will open the bottom event detail panel, providing deeper insights into the agent's execution flow.

### 2\.  `pytest` \- Run Tests Programmatically

You can also use **`pytest`** to run test files as part of your integration tests.

#### Example Command

```shell
pytest tests/integration/
```

#### Example Test Code

Here is an example of a `pytest` test case that runs a single test file:

```py
from google.adk.evaluation.agent_evaluator import AgentEvaluator
import pytest

@pytest.mark.asyncio
async def test_with_single_test_file():
    """Test the agent's basic ability via a session file."""
    await AgentEvaluator.evaluate(
        agent_module="home_automation_agent",
        eval_dataset_file_path_or_dir="tests/integration/fixture/home_automation_agent/simple_test.test.json",
    )
```

This approach allows you to integrate agent evaluations into your CI/CD pipelines or larger test suites. If you want to specify the initial session state for your tests, you can do that by storing the session details in a file and passing that to `AgentEvaluator.evaluate` method.

### 3\. `adk eval` \- Run Evaluations via the CLI

You can also run evaluation of an eval set file through the command line interface (CLI). This runs the same evaluation that runs on the UI, but it helps with automation, i.e. you can add this command as a part of your regular build generation and verification process.

Here is the command:

```shell
adk eval \
    <AGENT_MODULE_FILE_PATH> \
    <EVAL_SET_FILE_PATH> \
    [--config_file_path=<PATH_TO_TEST_JSON_CONFIG_FILE>] \
    [--print_detailed_results]
```

For example:

```shell
adk eval \
    samples_for_testing/hello_world \
    samples_for_testing/hello_world/hello_world_eval_set_001.evalset.json
```

Here are the details for each command line argument:

* `AGENT_MODULE_FILE_PATH`: The path to the `__init__.py` file that contains a module by the name "agent". "agent" module contains a `root_agent`.  
* `EVAL_SET_FILE_PATH`: The path to evaluations file(s). You can specify one or more eval set file paths. For each file, all evals will be run by default. If you want to run only specific evals from a eval set, first create a comma separated list of eval names and then add that as a suffix to the eval set file name, demarcated by a colon `:` .
* For example: `sample_eval_set_file.json:eval_1,eval_2,eval_3`  
  `This will only run eval_1, eval_2 and eval_3 from sample_eval_set_file.json`  
* `CONFIG_FILE_PATH`: The path to the config file.  
* `PRINT_DETAILED_RESULTS`: Prints detailed results on the console.


# Events

Events are the fundamental units of information flow within the Agent Development Kit (ADK). They represent every significant occurrence during an agent's interaction lifecycle, from initial user input to the final response and all the steps in between. Understanding events is crucial because they are the primary way components communicate, state is managed, and control flow is directed.

## What Events Are and Why They Matter

An `Event` in ADK is an immutable record representing a specific point in the agent's execution. It captures user messages, agent replies, requests to use tools (function calls), tool results, state changes, control signals, and errors.

=== "Python"
    Technically, it's an instance of the `google.adk.events.Event` class, which builds upon the basic `LlmResponse` structure by adding essential ADK-specific metadata and an `actions` payload.

    ```python
    # Conceptual Structure of an Event (Python)
    # from google.adk.events import Event, EventActions
    # from google.genai import types

    # class Event(LlmResponse): # Simplified view
    #     # --- LlmResponse fields ---
    #     content: Optional[types.Content]
    #     partial: Optional[bool]
    #     # ... other response fields ...

    #     # --- ADK specific additions ---
    #     author: str          # 'user' or agent name
    #     invocation_id: str   # ID for the whole interaction run
    #     id: str              # Unique ID for this specific event
    #     timestamp: float     # Creation time
    #     actions: EventActions # Important for side-effects & control
    #     branch: Optional[str] # Hierarchy path
    #     # ...
    ```

=== "Java"
    In Java, this is an instance of the `com.google.adk.events.Event` class. It also builds upon a basic response structure by adding essential ADK-specific metadata and an `actions` payload.

    

Events are central to ADK's operation for several key reasons:

1.  **Communication:** They serve as the standard message format between the user interface, the `Runner`, agents, the LLM, and tools. Everything flows as an `Event`.

2.  **Signaling State & Artifact Changes:** Events carry instructions for state modifications and track artifact updates. The `SessionService` uses these signals to ensure persistence. In Python changes are signaled via `event.actions.state_delta` and `event.actions.artifact_delta`.

3.  **Control Flow:** Specific fields like `event.actions.transfer_to_agent` or `event.actions.escalate` act as signals that direct the framework, determining which agent runs next or if a loop should terminate.

4.  **History & Observability:** The sequence of events recorded in `session.events` provides a complete, chronological history of an interaction, invaluable for debugging, auditing, and understanding agent behavior step-by-step.

In essence, the entire process, from a user's query to the agent's final answer, is orchestrated through the generation, interpretation, and processing of `Event` objects.


## Understanding and Using Events

As a developer, you'll primarily interact with the stream of events yielded by the `Runner`. Here's how to understand and extract information from them:

!!! Note
    The specific parameters or method names for the primitives may vary slightly by SDK language (e.g., `event.content()` in Python, `event.content().get().parts()` in Java). Refer to the language-specific API documentation for details.

### Identifying Event Origin and Type

Quickly determine what an event represents by checking:

*   **Who sent it? (`event.author`)**
    *   `'user'`: Indicates input directly from the end-user.
    *   `'AgentName'`: Indicates output or action from a specific agent (e.g., `'WeatherAgent'`, `'SummarizerAgent'`).
*   **What's the main payload? (`event.content` and `event.content.parts`)**
    *   **Text:** Indicates a conversational message. For Python, check if `event.content.parts[0].text` exists. For Java, check if `event.content()` is present, its `parts()` are present and not empty, and the first part's `text()` is present.
    *   **Tool Call Request:** Check `event.get_function_calls()`. If not empty, the LLM is asking to execute one or more tools. Each item in the list has `.name` and `.args`.
    *   **Tool Result:** Check `event.get_function_responses()`. If not empty, this event carries the result(s) from tool execution(s). Each item has `.name` and `.response` (the dictionary returned by the tool). *Note:* For history structuring, the `role` inside the `content` is often `'user'`, but the event `author` is typically the agent that requested the tool call.

*   **Is it streaming output? (`event.partial`)**
    Indicates whether this is an incomplete chunk of text from the LLM.
    *   `True`: More text will follow.
    *   `False` or `None`/`Optional.empty()`: This part of the content is complete (though the overall turn might not be finished if `turn_complete` is also false).

=== "Python"
    ```python
    # Pseudocode: Basic event identification (Python)
    # async for event in runner.run_async(...):
    #     print(f"Event from: {event.author}")
    #
    #     if event.content and event.content.parts:
    #         if event.get_function_calls():
    #             print("  Type: Tool Call Request")
    #         elif event.get_function_responses():
    #             print("  Type: Tool Result")
    #         elif event.content.parts[0].text:
    #             if event.partial:
    #                 print("  Type: Streaming Text Chunk")
    #             else:
    #                 print("  Type: Complete Text Message")
    #         else:
    #             print("  Type: Other Content (e.g., code result)")
    #     elif event.actions and (event.actions.state_delta or event.actions.artifact_delta):
    #         print("  Type: State/Artifact Update")
    #     else:
    #         print("  Type: Control Signal or Other")
    ```

=== "Java"
    

### Extracting Key Information

Once you know the event type, access the relevant data:

*   **Text Content:**
    Always check for the presence of content and parts before accessing text. In Python its `text = event.content.parts[0].text`.

*   **Function Call Details:**
    
    === "Python"
        ```python
        calls = event.get_function_calls()
        if calls:
            for call in calls:
                tool_name = call.name
                arguments = call.args # This is usually a dictionary
                print(f"  Tool: {tool_name}, Args: {arguments}")
                # Application might dispatch execution based on this
        ```
    === "Java"

        

*   **Function Response Details:**
    
    === "Python"
        ```python
        responses = event.get_function_responses()
        if responses:
            for response in responses:
                tool_name = response.name
                result_dict = response.response # The dictionary returned by the tool
                print(f"  Tool Result: {tool_name} -> {result_dict}")
        ```
    === "Java"

        

*   **Identifiers:**
    *   `event.id`: Unique ID for this specific event instance.
    *   `event.invocation_id`: ID for the entire user-request-to-final-response cycle this event belongs to. Useful for logging and tracing.

### Detecting Actions and Side Effects

The `event.actions` object signals changes that occurred or should occur. Always check if `event.actions` and it's fields/ methods exists before accessing them.

*   **State Changes:** Gives you a collection of key-value pairs that were modified in the session state during the step that produced this event.
    
    === "Python"
        `delta = event.actions.state_delta` (a dictionary of `{key: value}` pairs).
        ```python
        if event.actions and event.actions.state_delta:
            print(f"  State changes: {event.actions.state_delta}")
            # Update local UI or application state if necessary
        ```
    === "Java"
        `ConcurrentMap<String, Object> delta = event.actions().stateDelta();`

        

*   **Artifact Saves:** Gives you a collection indicating which artifacts were saved and their new version number (or relevant `Part` information).
    
    === "Python"
        `artifact_changes = event.actions.artifact_delta` (a dictionary of `{filename: version}`).
        ```python
        if event.actions and event.actions.artifact_delta:
            print(f"  Artifacts saved: {event.actions.artifact_delta}")
            # UI might refresh an artifact list
        ```
    === "Java"
        `ConcurrentMap<String, Part> artifactChanges = event.actions().artifactDelta();`
        
        

*   **Control Flow Signals:** Check boolean flags or string values:
    
    === "Python"
        *   `event.actions.transfer_to_agent` (string): Control should pass to the named agent.
        *   `event.actions.escalate` (bool): A loop should terminate.
        *   `event.actions.skip_summarization` (bool): A tool result should not be summarized by the LLM.
        ```python
        if event.actions:
            if event.actions.transfer_to_agent:
                print(f"  Signal: Transfer to {event.actions.transfer_to_agent}")
            if event.actions.escalate:
                print("  Signal: Escalate (terminate loop)")
            if event.actions.skip_summarization:
                print("  Signal: Skip summarization for tool result")
        ```
    === "Java"
        *   `event.actions().transferToAgent()` (returns `Optional<String>`): Control should pass to the named agent.
        *   `event.actions().escalate()` (returns `Optional<Boolean>`): A loop should terminate.
        *   `event.actions().skipSummarization()` (returns `Optional<Boolean>`): A tool result should not be summarized by the LLM.

        

### Determining if an Event is a "Final" Response

Use the built-in helper method `event.is_final_response()` to identify events suitable for display as the agent's complete output for a turn.

*   **Purpose:** Filters out intermediate steps (like tool calls, partial streaming text, internal state updates) from the final user-facing message(s).
*   **When `True`?**
    1.  The event contains a tool result (`function_response`) and `skip_summarization` is `True`.
    2.  The event contains a tool call (`function_call`) for a tool marked as `is_long_running=True`. In Java, check if the `longRunningToolIds` list is empty: 
        *   `event.longRunningToolIds().isPresent() && !event.longRunningToolIds().get().isEmpty()` is `true`.
    3.  OR, **all** of the following are met:
        *   No function calls (`get_function_calls()` is empty).
        *   No function responses (`get_function_responses()` is empty).
        *   Not a partial stream chunk (`partial` is not `True`).
        *   Doesn't end with a code execution result that might need further processing/display.
*   **Usage:** Filter the event stream in your application logic.

    === "Python"
        ```python
        # Pseudocode: Handling final responses in application (Python)
        # full_response_text = ""
        # async for event in runner.run_async(...):
        #     # Accumulate streaming text if needed...
        #     if event.partial and event.content and event.content.parts and event.content.parts[0].text:
        #         full_response_text += event.content.parts[0].text
        #
        #     # Check if it's a final, displayable event
        #     if event.is_final_response():
        #         print("\n--- Final Output Detected ---")
        #         if event.content and event.content.parts and event.content.parts[0].text:
        #              # If it's the final part of a stream, use accumulated text
        #              final_text = full_response_text + (event.content.parts[0].text if not event.partial else "")
        #              print(f"Display to user: {final_text.strip()}")
        #              full_response_text = "" # Reset accumulator
        #         elif event.actions and event.actions.skip_summarization and event.get_function_responses():
        #              # Handle displaying the raw tool result if needed
        #              response_data = event.get_function_responses()[0].response
        #              print(f"Display raw tool result: {response_data}")
        #         elif hasattr(event, 'long_running_tool_ids') and event.long_running_tool_ids:
        #              print("Display message: Tool is running in background...")
        #         else:
        #              # Handle other types of final responses if applicable
        #              print("Display: Final non-textual response or signal.")
        ```
    === "Java"
        

By carefully examining these aspects of an event, you can build robust applications that react appropriately to the rich information flowing through the ADK system.

## How Events Flow: Generation and Processing

Events are created at different points and processed systematically by the framework. Understanding this flow helps clarify how actions and history are managed.

*   **Generation Sources:**
    *   **User Input:** The `Runner` typically wraps initial user messages or mid-conversation inputs into an `Event` with `author='user'`.
    *   **Agent Logic:** Agents (`BaseAgent`, `LlmAgent`) explicitly `yield Event(...)` objects (setting `author=self.name`) to communicate responses or signal actions.
    *   **LLM Responses:** The ADK model integration layer translates raw LLM output (text, function calls, errors) into `Event` objects, authored by the calling agent.
    *   **Tool Results:** After a tool executes, the framework generates an `Event` containing the `function_response`. The `author` is typically the agent that requested the tool, while the `role` inside the `content` is set to `'user'` for the LLM history.


*   **Processing Flow:**
    1.  **Yield/Return:** An event is generated and yielded (Python) or returned/emitted (Java) by its source.
    2.  **Runner Receives:** The main `Runner` executing the agent receives the event.
    3.  **SessionService Processing:** The `Runner` sends the event to the configured `SessionService`. This is a critical step:
        *   **Applies Deltas:** The service merges `event.actions.state_delta` into `session.state` and updates internal records based on `event.actions.artifact_delta`. (Note: The actual artifact *saving* usually happened earlier when `context.save_artifact` was called).
        *   **Finalizes Metadata:** Assigns a unique `event.id` if not present, may update `event.timestamp`.
        *   **Persists to History:** Appends the processed event to the `session.events` list.
    4.  **External Yield:** The `Runner` yields (Python) or returns/emits (Java) the processed event outwards to the calling application (e.g., the code that invoked `runner.run_async`).

This flow ensures that state changes and history are consistently recorded alongside the communication content of each event.


## Common Event Examples (Illustrative Patterns)

Here are concise examples of typical events you might see in the stream:

*   **User Input:**
    ```json
    {
      "author": "user",
      "invocation_id": "e-xyz...",
      "content": {"parts": [{"text": "Book a flight to London for next Tuesday"}]}
      // actions usually empty
    }
    ```
*   **Agent Final Text Response:** (`is_final_response() == True`)
    ```json
    {
      "author": "TravelAgent",
      "invocation_id": "e-xyz...",
      "content": {"parts": [{"text": "Okay, I can help with that. Could you confirm the departure city?"}]},
      "partial": false,
      "turn_complete": true
      // actions might have state delta, etc.
    }
    ```
*   **Agent Streaming Text Response:** (`is_final_response() == False`)
    ```json
    {
      "author": "SummaryAgent",
      "invocation_id": "e-abc...",
      "content": {"parts": [{"text": "The document discusses three main points:"}]},
      "partial": true,
      "turn_complete": false
    }
    // ... more partial=True events follow ...
    ```
*   **Tool Call Request (by LLM):** (`is_final_response() == False`)
    ```json
    {
      "author": "TravelAgent",
      "invocation_id": "e-xyz...",
      "content": {"parts": [{"function_call": {"name": "find_airports", "args": {"city": "London"}}}]}
      // actions usually empty
    }
    ```
*   **Tool Result Provided (to LLM):** (`is_final_response()` depends on `skip_summarization`)
    ```json
    {
      "author": "TravelAgent", // Author is agent that requested the call
      "invocation_id": "e-xyz...",
      "content": {
        "role": "user", // Role for LLM history
        "parts": [{"function_response": {"name": "find_airports", "response": {"result": ["LHR", "LGW", "STN"]}}}]
      }
      // actions might have skip_summarization=True
    }
    ```
*   **State/Artifact Update Only:** (`is_final_response() == False`)
    ```json
    {
      "author": "InternalUpdater",
      "invocation_id": "e-def...",
      "content": null,
      "actions": {
        "state_delta": {"user_status": "verified"},
        "artifact_delta": {"verification_doc.pdf": 2}
      }
    }
    ```
*   **Agent Transfer Signal:** (`is_final_response() == False`)
    ```json
    {
      "author": "OrchestratorAgent",
      "invocation_id": "e-789...",
      "content": {"parts": [{"function_call": {"name": "transfer_to_agent", "args": {"agent_name": "BillingAgent"}}}]},
      "actions": {"transfer_to_agent": "BillingAgent"} // Added by framework
    }
    ```
*   **Loop Escalation Signal:** (`is_final_response() == False`)
    ```json
    {
      "author": "CheckerAgent",
      "invocation_id": "e-loop...",
      "content": {"parts": [{"text": "Maximum retries reached."}]}, // Optional content
      "actions": {"escalate": true}
    }
    ```

## Additional Context and Event Details

Beyond the core concepts, here are a few specific details about context and events that are important for certain use cases:

1.  **`ToolContext.function_call_id` (Linking Tool Actions):**
    *   When an LLM requests a tool (FunctionCall), that request has an ID. The `ToolContext` provided to your tool function includes this `function_call_id`.
    *   **Importance:** This ID is crucial for linking actions like authentication back to the specific tool request that initiated them, especially if multiple tools are called in one turn. The framework uses this ID internally.

2.  **How State/Artifact Changes are Recorded:**
    *   When you modify state or save an artifact using `CallbackContext` or `ToolContext`, these changes aren't immediately written to persistent storage.
    *   Instead, they populate the `state_delta` and `artifact_delta` fields within the `EventActions` object.
    *   This `EventActions` object is attached to the *next event* generated after the change (e.g., the agent's response or a tool result event).
    *   The `SessionService.append_event` method reads these deltas from the incoming event and applies them to the session's persistent state and artifact records. This ensures changes are tied chronologically to the event stream.

3.  **State Scope Prefixes (`app:`, `user:`, `temp:`):**
    *   When managing state via `context.state`, you can optionally use prefixes:
        *   `app:my_setting`: Suggests state relevant to the entire application (requires a persistent `SessionService`).
        *   `user:user_preference`: Suggests state relevant to the specific user across sessions (requires a persistent `SessionService`).
        *   `temp:intermediate_result` or no prefix: Typically session-specific or temporary state for the current invocation.
    *   The underlying `SessionService` determines how these prefixes are handled for persistence.

4.  **Error Events:**
    *   An `Event` can represent an error. Check the `event.error_code` and `event.error_message` fields (inherited from `LlmResponse`).
    *   Errors might originate from the LLM (e.g., safety filters, resource limits) or potentially be packaged by the framework if a tool fails critically. Check tool `FunctionResponse` content for typical tool-specific errors.
    ```json
    // Example Error Event (conceptual)
    {
      "author": "LLMAgent",
      "invocation_id": "e-err...",
      "content": null,
      "error_code": "SAFETY_FILTER_TRIGGERED",
      "error_message": "Response blocked due to safety settings.",
      "actions": {}
    }
    ```

These details provide a more complete picture for advanced use cases involving tool authentication, state persistence scope, and error handling within the event stream.

## Best Practices for Working with Events

To use events effectively in your ADK applications:

*   **Clear Authorship:** When building custom agents, ensure correct attribution for agent actions in the history. The framework generally handles authorship correctly for LLM/tool events.
    
    === "Python"
        Use `yield Event(author=self.name, ...)` in `BaseAgent` subclasses.
    === "Java"
        When constructing an `Event` in your custom agent logic, set the author, for example: `Event.builder().author(this.getAgentName()) // ... .build();`

*   **Semantic Content & Actions:** Use `event.content` for the core message/data (text, function call/response). Use `event.actions` specifically for signaling side effects (state/artifact deltas) or control flow (`transfer`, `escalate`, `skip_summarization`).
*   **Idempotency Awareness:** Understand that the `SessionService` is responsible for applying the state/artifact changes signaled in `event.actions`. While ADK services aim for consistency, consider potential downstream effects if your application logic re-processes events.
*   **Use `is_final_response()`:** Rely on this helper method in your application/UI layer to identify complete, user-facing text responses. Avoid manually replicating its logic.
*   **Leverage History:** The session's event list is your primary debugging tool. Examine the sequence of authors, content, and actions to trace execution and diagnose issues.
*   **Use Metadata:** Use `invocation_id` to correlate all events within a single user interaction. Use `event.id` to reference specific, unique occurrences.

Treating events as structured messages with clear purposes for their content and actions is key to building, debugging, and managing complex agent behaviors in ADK.

# Agent Development Kit (ADK)

<p style="text-align:center;"> <b> Build, Evaluate and Deploy agents, seamlessly! </b> </p>

ADK is designed to empower developers
to build, manage, evaluate and deploy AI-powered agents. It provides a robust
and flexible environment for creating both conversational and non-conversational
agents, capable of handling complex tasks and workflows.

![intro_components.png](../assets/adk-components.png)

## Core Concepts

ADK is built around a few key primitives and concepts that make it
powerful and flexible. Here are the essentials:

* **Agent:** The fundamental worker unit designed for specific tasks. Agents can
  use language models (`LlmAgent`) for complex reasoning, or act as deterministic controllers of the execution, which are called "[workflow agents](../agents/workflow-agents/index.md)" (`SequentialAgent`, `ParallelAgent`, `LoopAgent`).
* **Tool:** Gives agents abilities beyond conversation, letting them interact
  with external APIs, search information, run code, or call other services.
* **Callbacks:** Custom code snippets you provide to run at specific points in
  the agent's process, allowing for checks, logging, or behavior modifications.
* **Session Management (`Session` & `State`):** Handles the context of a single
  conversation (`Session`), including its history (`Events`) and the agent's
  working memory for that conversation (`State`).
* **Memory:** Enables agents to recall information about a user across
  *multiple* sessions, providing long-term context (distinct from short-term
  session `State`).
* **Artifact Management (`Artifact`):** Allows agents to save, load, and manage
  files or binary data (like images, PDFs) associated with a session or user.
* **Code Execution:** The ability for agents (usually via Tools) to generate and
  execute code to perform complex calculations or actions.
* **Planning:** An advanced capability where agents can break down complex goals
  into smaller steps and plan how to achieve them like a ReAct planner.
* **Models:** The underlying LLM that powers `LlmAgent`s, enabling their
  reasoning and language understanding abilities.
* **Event:** The basic unit of communication representing things that happen
  during a session (user message, agent reply, tool use), forming the
  conversation history.
* **Runner:** The engine that manages the execution flow, orchestrates agent
  interactions based on Events, and coordinates with backend services.

***Note:** Features like Multimodal Streaming, Evaluation, Deployment,
Debugging, and Trace are also part of the broader ADK ecosystem, supporting
real-time interaction and the development lifecycle.*

## Key Capabilities

ADK offers several key advantages for developers building
agentic applications:

1. **Multi-Agent System Design:** Easily build applications composed of
   multiple, specialized agents arranged hierarchically. Agents can coordinate
   complex tasks, delegate sub-tasks using LLM-driven transfer or explicit
   `AgentTool` invocation, enabling modular and scalable solutions.
2. **Rich Tool Ecosystem:** Equip agents with diverse capabilities. ADK
   supports integrating custom functions (`FunctionTool`), using other agents as
   tools (`AgentTool`), leveraging built-in functionalities like code execution,
   and interacting with external data sources and APIs (e.g., Search,
   Databases). Support for long-running tools allows handling asynchronous
   operations effectively.
3. **Flexible Orchestration:** Define complex agent workflows using built-in
   workflow agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`) alongside
   LLM-driven dynamic routing. This allows for both predictable pipelines and
   adaptive agent behavior.
4. **Integrated Developer Tooling:** Develop and iterate locally with ease.
   ADK includes tools like a command-line interface (CLI) and a Developer
   UI for running agents, inspecting execution steps (events, state changes),
   debugging interactions, and visualizing agent definitions.
5. **Native Streaming Support:** Build real-time, interactive experiences with
   native support for bidirectional streaming (text and audio). This integrates
   seamlessly with underlying capabilities like the
   [Multimodal Live API for the Gemini Developer API](https://ai.google.dev/gemini-api/docs/live)
   (or for
   [Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/multimodal-live)),
   often enabled with simple configuration changes.
6. **Built-in Agent Evaluation:** Assess agent performance systematically. The
   framework includes tools to create multi-turn evaluation datasets and run
   evaluations locally (via CLI or the dev UI) to measure quality and
   guide improvements.
7. **Broad LLM Support:** While optimized for Google's Gemini models, the
   framework is designed for flexibility, allowing integration with various LLMs
   (potentially including open-source or fine-tuned models) through its
   `BaseLlm` interface.
8. **Artifact Management:** Enable agents to handle files and binary data. The
   framework provides mechanisms (`ArtifactService`, context methods) for agents
   to save, load, and manage versioned artifacts like images, documents, or
   generated reports during their execution.
9. **Extensibility and Interoperability:** ADK promotes an open
   ecosystem. While providing core tools, it allows developers to easily
   integrate and reuse tools from other popular agent frameworks including
   LangChain and CrewAI.
10. **State and Memory Management:** Automatically handles short-term
    conversational memory (`State` within a `Session`) managed by the
    `SessionService`. Provides integration points for longer-term `Memory`
    services, allowing agents to recall user information across multiple
    sessions.

![intro_components.png](../assets/adk-lifecycle.png)

## Get Started

* Ready to build your first agent? [Try the quickstart](./quickstart.md)


# Get Started

Agent Development Kit (ADK) is designed to empower developers
to build, manage, evaluate and deploy AI-powered agents. It provides a robust
and flexible environment for creating both conversational and non-conversational
agents, capable of handling complex tasks and workflows.

<div class="grid cards" markdown>

-   :material-console-line: **Installation**

    ---

    Install `google-adk` for Python or Java and get up and running in minutes.

    [:octicons-arrow-right-24: More information](installation.md)

-   :material-console-line: **Quickstart**

    ---

    Create your first ADK agent with tools in minutes.

    [:octicons-arrow-right-24: More information](quickstart.md)

-   :material-console-line: **Quickstart (streaming)**

    ---

    Create your first streaming ADK agent.

    [:octicons-arrow-right-24: More information](streaming/quickstart-streaming.md)

-   :material-console-line: **Tutorial**

    ---

    Create your first ADK multi-agent.

    [:octicons-arrow-right-24: More information](../tutorials/index.md)

-   :material-rocket-launch-outline: **Discover sample agents**

    ---

    Discover sample agents for retail, travel, customer service, and more!

    [:octicons-arrow-right-24: Discover adk-samples](https://github.com/google/adk-samples){:target="_blank"}

-   :material-graph: **About**

    ---

    Learn about the key components of building and deploying ADK agents.

    [:octicons-arrow-right-24: More information](about.md)

</div>


# Installing ADK

=== "Python"

    ## Create & activate virtual environment
    
    We recommend creating a virtual Python environment using
    [venv](https://docs.python.org/3/library/venv.html):
    
    ```shell
    python -m venv .venv
    ```
    
    Now, you can activate the virtual environment using the appropriate command for
    your operating system and environment:
    
    ```
    # Mac / Linux
    source .venv/bin/activate
    
    # Windows CMD:
    .venv\Scripts\activate.bat
    
    # Windows PowerShell:
    .venv\Scripts\Activate.ps1
    ```

    ### Install ADK
    
    ```bash
    pip install google-adk
    ```
    
    (Optional) Verify your installation:
    
    ```bash
    pip show google-adk
    ```

=== "Java"

    You can either use maven or gradle to add the `google-adk` and `google-adk-dev` package.

    `google-adk` is the core Java ADK library. Java ADK also comes with a pluggable example SpringBoot server to run your agents seamlessly. This optional
    package is present as part of `google-adk-dev`.
    
    If you are using maven, add the following to your `pom.xml`:

    ```xml title="pom.xml"
    <dependencies>
      <!-- The ADK Core dependency -->
      <dependency>
        <groupId>com.google.adk</groupId>
        <artifactId>google-adk</artifactId>
        <version>0.1.0</version>
      </dependency>
      
      <!-- The ADK Dev Web UI to debug your agent (Optional) -->
      <dependency>
        <groupId>com.google.adk</groupId>
        <artifactId>google-adk-dev</artifactId>
        <version>0.1.0</version>
      </dependency>
    </dependencies>
    ```

    Here's a [complete pom.xml](https://github.com/google/adk-docs/tree/main/examples/java/cloud-run/pom.xml) file for reference.

    If you are using gradle, add the dependency to your build.gradle:

    ```title="build.gradle"
    dependencies {
        implementation 'com.google.adk:google-adk:0.1.0'
        implementation 'com.google.adk:google-adk-dev:0.1.0'
    }
    ```


## Next steps

* Try creating your first agent with the [**Quickstart**](quickstart.md)


# Quickstart

This quickstart guides you through installing the Agent Development Kit (ADK),
setting up a basic agent with multiple tools, and running it locally either in the terminal or in the interactive, browser-based dev UI.

<!-- <img src="../../assets/quickstart.png" alt="Quickstart setup"> -->

This quickstart assumes a local IDE (VS Code, PyCharm, IntelliJ IDEA, etc.)
with Python 3.10+ or Java 17+ and terminal access. This method runs the
application entirely on your machine and is recommended for internal development.

## 1. Set up Environment & Install ADK {#venv-install}

=== "Python"

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
    pip install google-adk
    ```

=== "Java"

    To install ADK and setup the environment, proceed to the following steps.

## 2. Create Agent Project {#create-agent-project}

### Project structure

=== "Python"

    You will need to create the following project structure:

    ```console
    parent_folder/
        multi_tool_agent/
            __init__.py
            agent.py
            .env
    ```

    Create the folder `multi_tool_agent`:

    ```bash
    mkdir multi_tool_agent/
    ```

    !!! info "Note for Windows users"

        When using ADK on Windows for the next few steps, we recommend creating
        Python files using File Explorer or an IDE because the following commands
        (`mkdir`, `echo`) typically generate files with null bytes and/or incorrect
        encoding.

    ### `__init__.py`

    Now create an `__init__.py` file in the folder:

    ```shell
    echo "from . import agent" > multi_tool_agent/__init__.py
    ```

    Your `__init__.py` should now look like this:

    ```python title="multi_tool_agent/__init__.py"
    from . import agent

    ```

    ### `agent.py`

    Create an `agent.py` file in the same folder:

    ```shell
    touch multi_tool_agent/agent.py
    ```

    Copy and paste the following code into `agent.py`:

    ```python title="multi_tool_agent/agent.py"
    import datetime
    from zoneinfo import ZoneInfo
    from google.adk.agents import Agent

    def get_weather(city: str) -> dict:
        """Retrieves the current weather report for a specified city.

        Args:
            city (str): The name of the city for which to retrieve the weather report.

        Returns:
            dict: status and result or error msg.
        """
        if city.lower() == "new york":
            return {
                "status": "success",
                "report": (
                    "The weather in New York is sunny with a temperature of 25 degrees"
                    " Celsius (77 degrees Fahrenheit)."
                ),
            }
        else:
            return {
                "status": "error",
                "error_message": f"Weather information for '{city}' is not available.",
            }


    def get_current_time(city: str) -> dict:
        """Returns the current time in a specified city.

        Args:
            city (str): The name of the city for which to retrieve the current time.

        Returns:
            dict: status and result or error msg.
        """

        if city.lower() == "new york":
            tz_identifier = "America/New_York"
        else:
            return {
                "status": "error",
                "error_message": (
                    f"Sorry, I don't have timezone information for {city}."
                ),
            }

        tz = ZoneInfo(tz_identifier)
        now = datetime.datetime.now(tz)
        report = (
            f'The current time in {city} is {now.strftime("%Y-%m-%d %H:%M:%S %Z%z")}'
        )
        return {"status": "success", "report": report}


    root_agent = Agent(
        name="weather_time_agent",
        model="gemini-2.5-flash",
        description=(
            "Agent to answer questions about the time and weather in a city."
        ),
        instruction=(
            "You are a helpful agent who can answer user questions about the time and weather in a city."
        ),
        tools=[get_weather, get_current_time],
    )

    ```

    ### `.env`

    Create a `.env` file in the same folder:

    ```shell
    touch multi_tool_agent/.env
    ```

    More instructions about this file are described in the next section on [Set up the model](#set-up-the-model).

=== "Java"

    Java projects generally feature the following project structure:

    ```console
    project_folder/
    ├── pom.xml (or build.gradle)
    ├── src/
    ├── └── main/
    │       └── java/
    │           └── agents/
    │               └── multitool/
    └── test/
    ```

    ### Create `MultiToolAgent.java`

    Create a `MultiToolAgent.java` source file in the `agents.multitool` package
    in the `src/main/java/agents/multitool/` directory.

    Copy and paste the following code into `MultiToolAgent.java`:

    

![intro_components.png](../assets/quickstart-flow-tool.png)

## 3. Set up the model {#set-up-the-model}

Your agent's ability to understand user requests and generate responses is
powered by a Large Language Model (LLM). Your agent needs to make secure calls
to this external LLM service, which requires authentication credentials. Without
valid authentication, the LLM service will deny the agent's requests, and the
agent will be unable to function.

=== "Gemini - Google AI Studio"
    1. Get an API key from [Google AI Studio](https://aistudio.google.com/apikey).
    2. When using Python, open the **`.env`** file located inside (`multi_tool_agent/`)
    and copy-paste the following code.

        ```env title="multi_tool_agent/.env"
        GOOGLE_GENAI_USE_VERTEXAI=FALSE
        GOOGLE_API_KEY=PASTE_YOUR_ACTUAL_API_KEY_HERE
        ```

        When using Java, define environment variables:

        ```console title="terminal"
        export GOOGLE_GENAI_USE_VERTEXAI=FALSE
        export GOOGLE_API_KEY=PASTE_YOUR_ACTUAL_API_KEY_HERE
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
    2. When using Python, open the **`.env`** file located inside (`multi_tool_agent/`). Copy-paste
    the following code and update the project ID and location.

        ```env title="multi_tool_agent/.env"
        GOOGLE_GENAI_USE_VERTEXAI=TRUE
        GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
        GOOGLE_CLOUD_LOCATION=LOCATION
        ```

        When using Java, define environment variables:

        ```console title="terminal"
        export GOOGLE_GENAI_USE_VERTEXAI=TRUE
        export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
        export GOOGLE_CLOUD_LOCATION=LOCATION
        ```

## 4. Run Your Agent {#run-your-agent}

=== "Python"

    Using the terminal, navigate to the parent directory of your agent project
    (e.g. using `cd ..`):

    ```console
    parent_folder/      <-- navigate to this directory
        multi_tool_agent/
            __init__.py
            agent.py
            .env
    ```

    There are multiple ways to interact with your agent:

    === "Dev UI (adk web)"
        Run the following command to launch the **dev UI**.

        ```shell
        adk web
        ```
        
        !!!info "Note for Windows users"

            When hitting the `_make_subprocess_transport NotImplementedError`, consider using `adk web --no-reload` instead.


        **Step 1:** Open the URL provided (usually `http://localhost:8000` or
        `http://127.0.0.1:8000`) directly in your browser.

        **Step 2.** In the top-left corner of the UI, you can select your agent in
        the dropdown. Select "multi_tool_agent".

        !!!note "Troubleshooting"

            If you do not see "multi_tool_agent" in the dropdown menu, make sure you
            are running `adk web` in the **parent folder** of your agent folder
            (i.e. the parent folder of multi_tool_agent).

        **Step 3.** Now you can chat with your agent using the textbox:

        ![adk-web-dev-ui-chat.png](../assets/adk-web-dev-ui-chat.png)


        **Step 4.**  By using the `Events` tab at the left, you can inspect
        individual function calls, responses and model responses by clicking on the
        actions:

        ![adk-web-dev-ui-function-call.png](../assets/adk-web-dev-ui-function-call.png)

        On the `Events` tab, you can also click the `Trace` button to see the trace logs for each event that shows the latency of each function calls:

        ![adk-web-dev-ui-trace.png](../assets/adk-web-dev-ui-trace.png)

        **Step 5.** You can also enable your microphone and talk to your agent:

        !!!note "Model support for voice/video streaming"

            In order to use voice/video streaming in ADK, you will need to use Gemini models that support the Live API. You can find the **model ID(s)** that supports the Gemini Live API in the documentation:

            - [Google AI Studio: Gemini Live API](https://ai.google.dev/gemini-api/docs/models#live-api)
            - [Vertex AI: Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api)

            You can then replace the `model` string in `root_agent` in the `agent.py` file you created earlier ([jump to section](#agentpy)). Your code should look something like:

            ```py
            root_agent = Agent(
                name="weather_time_agent",
                model="replace-me-with-model-id", #e.g. gemini-2.5-flash-live-001
                ...
            ```

        ![adk-web-dev-ui-audio.png](../assets/adk-web-dev-ui-audio.png)

    === "Terminal (adk run)"

        Run the following command, to chat with your Weather agent.

        ```
        adk run multi_tool_agent
        ```

        ![adk-run.png](../assets/adk-run.png)

        To exit, use Cmd/Ctrl+C.

    === "API Server (adk api_server)"

        `adk api_server` enables you to create a local FastAPI server in a single
        command, enabling you to test local cURL requests before you deploy your
        agent.

        ![adk-api-server.png](../assets/adk-api-server.png)

        To learn how to use `adk api_server` for testing, refer to the
        [documentation on testing](testing.md).

=== "Java"

    Using the terminal, navigate to the parent directory of your agent project
    (e.g. using `cd ..`):

    ```console
    project_folder/                <-- navigate to this directory
    ├── pom.xml (or build.gradle)
    ├── src/
    ├── └── main/
    │       └── java/
    │           └── agents/
    │               └── multitool/
    │                   └── MultiToolAgent.java
    └── test/
    ```

    === "Dev UI"

        Run the following command from the terminal to launch the Dev UI.

        **DO NOT change the main class name of the Dev UI server.**

        ```console title="terminal"
        mvn exec:java \
            -Dexec.mainClass="com.google.adk.web.AdkWebServer" \
            -Dexec.args="--adk.agents.source-dir=src/main/java" \
            -Dexec.classpathScope="compile"
        ```

        **Step 1:** Open the URL provided (usually `http://localhost:8080` or
        `http://127.0.0.1:8080`) directly in your browser.

        **Step 2.** In the top-left corner of the UI, you can select your agent in
        the dropdown. Select "multi_tool_agent".

        !!!note "Troubleshooting"

            If you do not see "multi_tool_agent" in the dropdown menu, make sure you
            are running the `mvn` command at the location where your Java source code
            is located (usually `src/main/java`).

        **Step 3.** Now you can chat with your agent using the textbox:

        ![adk-web-dev-ui-chat.png](../assets/adk-web-dev-ui-chat.png)

        **Step 4.** You can also inspect individual function calls, responses and
        model responses by clicking on the actions:

        ![adk-web-dev-ui-function-call.png](../assets/adk-web-dev-ui-function-call.png)

    === "Maven"

        With Maven, run the `main()` method of your Java class
        with the following command:

        ```console title="terminal"
        mvn compile exec:java -Dexec.mainClass="agents.multitool.MultiToolAgent"
        ```

    === "Gradle"

        With Gradle, the `build.gradle` or `build.gradle.kts` build file
        should have the following Java plugin in its `plugins` section:

        ```groovy
        plugins {
            id("java")
            // other plugins
        }
        ```

        Then, elsewhere in the build file, at the top-level,
        create a new task to run the `main()` method of your agent:

        ```groovy
        task runAgent(type: JavaExec) {
            classpath = sourceSets.main.runtimeClasspath
            mainClass = "agents.multitool.MultiToolAgent"
        }
        ```

        Finally, on the command-line, run the following command:

        ```console
        gradle runAgent
        ```



### 📝 Example prompts to try

* What is the weather in New York?
* What is the time in New York?
* What is the weather in Paris?
* What is the time in Paris?

## 🎉 Congratulations!

You've successfully created and interacted with your first agent using ADK!

---

## 🛣️ Next steps

* **Go to the tutorial**: Learn how to add memory, session, state to your agent:
  [tutorial](../tutorials/index.md).
* **Delve into advanced configuration:** Explore the [setup](installation.md)
  section for deeper dives into project structure, configuration, and other
  interfaces.
* **Understand Core Concepts:** Learn about
  [agents concepts](../agents/index.md).


# Streaming Quickstarts

The Agent Development Kit (ADK) enables real-time, interactive experiences with your AI agents through streaming. This allows for features like live voice conversations, real-time tool use, and continuous updates from your agent.

This page provides quickstart examples to get you up and running with streaming capabilities in both Python and Java ADK.

<div class.="grid cards" markdown>

-   :fontawesome-brands-python:{ .lg .middle } **Python ADK: Streaming Quickstart**

    ---
    This example demonstrates how to set up a basic streaming interaction with an agent using Python ADK. It typically involves using the `Runner.run_live()` method and handling asynchronous events.

    [:octicons-arrow-right-24: View Python Streaming Quickstart](quickstart-streaming.md) <br>
    <!-- [:octicons-arrow-right-24: View Python Streaming Quickstart](python/quickstart-streaming.md) -->

<!-- This comment forces a block separation -->

-   :fontawesome-brands-java:{ .lg .middle } **Java ADK: Streaming Quickstart**

    ---
    This example demonstrates how to set up a basic streaming interaction with an agent using Java ADK. It involves using the `Runner.runLive()` method, a `LiveRequestQueue`, and handling the `Flowable<Event>` stream.

    [:octicons-arrow-right-24: View Java Streaming Quickstart](quickstart-streaming-java.md) <br>
    <!-- [:octicons-arrow-right-24: View Java Streaming Quickstart](java/quickstart-streaming-java.md)) -->

</div>


# Quickstart (Streaming / Java) {#adk-streaming-quickstart-java}

This quickstart guide will walk you through the process of creating a basic agent and leveraging ADK Streaming with Java to facilitate low-latency, bidirectional voice interactions.

You'll begin by setting up your Java and Maven environment, structuring your project, and defining the necessary dependencies. Following this, you'll create a simple `ScienceTeacherAgent`, test its text-based streaming capabilities using the Dev UI, and then progress to enabling live audio communication, transforming your agent into an interactive voice-driven application.

## **Create your first agent** {#create-your-first-agent}

### **Prerequisites**

* In this getting started guide, you will be programming in Java. Check if **Java** is installed on your machine. Ideally, you should be using Java 17 or more (you can check that by typing **java \-version**)

* You’ll also be using the **Maven** build tool for Java. So be sure to have [Maven installed](https://maven.apache.org/install.html) on your machine before going further (this is the case for Cloud Top or Cloud Shell, but not necessarily for your laptop).

### **Prepare the project structure**

To get started with ADK Java, let’s create a Maven project with the following directory structure:

```
adk-agents/
├── pom.xml
└── src/
    └── main/
        └── java/
            └── agents/
                └── ScienceTeacherAgent.java
```

Follow the instructions in [Installation](../../get-started/installation.md) page to add `pom.xml` for using the ADK package.

!!! Note
    Feel free to use whichever name you like for the root directory of your project (instead of adk-agents)

### **Running a compilation**

Let’s see if Maven is happy with this build, by running a compilation (**mvn compile** command):

```shell
$ mvn compile
[INFO] Scanning for projects...
[INFO] 
[INFO] --------------------< adk-agents:adk-agents >--------------------
[INFO] Building adk-agents 1.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ adk-demo ---
[INFO] skip non existing resourceDirectory /home/user/adk-demo/src/main/resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ adk-demo ---
[INFO] Nothing to compile - all classes are up to date.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.347 s
[INFO] Finished at: 2025-05-06T15:38:08Z
[INFO] ------------------------------------------------------------------------
```

Looks like the project is set up properly for compilation\!

### **Creating an agent**

Create the **ScienceTeacherAgent.java** file under the `src/main/java/agents/` directory with the following content:



!!!note "Troubleshooting"

    The model `gemini-2.5-flash-exp` will be deprecated in the future. If you see any issues on using it, try using `gemini-2.5-flash-live-001` instead

We will use `Dev UI` to run this agent later. For the tool to automatically recognize the agent, its Java class has to comply with the following two rules:

* The agent should be stored in a global **public static** variable named **ROOT\_AGENT** of type **BaseAgent** and initialized at declaration time.
* The agent definition has to be a **static** method so it can be loaded during the class initialization by the dynamic compiling classloader.

## **Run agent with Dev UI** {#run-agent-with-adk-web-server}

`Dev UI` is a web server where you can quickly run and test your agents for development purpose, without building your own UI application for the agents.

### **Define environment variables**

To run the server, you’ll need to export two environment variables:

* a Gemini key that you can [get from AI Studio](https://ai.google.dev/gemini-api/docs/api-key),
* a variable to specify we’re not using Vertex AI this time.

```shell
export GOOGLE_GENAI_USE_VERTEXAI=FALSE
export GOOGLE_API_KEY=YOUR_API_KEY
```

### **Run Dev UI**

Run the following command from the terminal to launch the Dev UI.

```console title="terminal"
mvn exec:java \
    -Dexec.mainClass="com.google.adk.web.AdkWebServer" \
    -Dexec.args="--adk.agents.source-dir=src/main/java" \
    -Dexec.classpathScope="compile"
```

**Step 1:** Open the URL provided (usually `http://localhost:8080` or
`http://127.0.0.1:8080`) directly in your browser.

**Step 2.** In the top-left corner of the UI, you can select your agent in
the dropdown. Select "science-app".

!!!note "Troubleshooting"

    If you do not see "science-app" in the dropdown menu, make sure you
    are running the `mvn` command at the location where your Java source code
    is located (usually `src/main/java`).

## Try Dev UI with text

With your favorite browser, navigate to: [http://127.0.0.1:8080/](http://127.0.0.1:8080/)

You should see the following interface:

![Dev UI](../../assets/quickstart-streaming-devui.png)

Click the `Token Streaming` switch at the top right, and ask any questions for the science teacher such as `What's the electron?`. Then you should see the output text in streaming on the UI.

As we saw, you do not have to write any specific code in the agent itself for the text streaming capability. It is provided as an ADK Agent feature by default.

### Try with voice and video

To try with voice, reload the web browser, click the microphone button to enable the voice input, and ask the same question in voice. You will hear the answer in voice in real-time.

To try with video, reload the web browser, click the camera button to enable the video input, and ask questions like "What do you see?". The agent will answer what they see in the video input.

### Stop the tool

Stop the tool by pressing `Ctrl-C` on the console.

## **Run agent with a custom live audio app** {#run-agent-with-live-audio}

Now, let's try audio streaming with the agent and a custom live audio application.

### **A Maven pom.xml build file for Live Audio**

Replace your existing pom.xml with the following.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.google.adk.samples</groupId>
  <artifactId>google-adk-sample-live-audio</artifactId>
  <version>0.1.0</version>
  <name>Google ADK - Sample - Live Audio</name>
  <description>
    A sample application demonstrating a live audio conversation using ADK,
    runnable via samples.liveaudio.LiveAudioRun.
  </description>
  <packaging>jar</packaging>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <java.version>17</java.version>
    <auto-value.version>1.11.0</auto-value.version>
    <!-- Main class for exec-maven-plugin -->
    <exec.mainClass>samples.liveaudio.LiveAudioRun</exec.mainClass>
    <google-adk.version>0.1.0</google-adk.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>libraries-bom</artifactId>
        <version>26.53.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>com.google.adk</groupId>
      <artifactId>google-adk</artifactId>
      <version>${google-adk.version}</version>
    </dependency>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version> <!-- Or use a property if defined in a parent POM -->
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.13.0</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
          <parameters>true</parameters>
          <annotationProcessorPaths>
            <path>
              <groupId>com.google.auto.value</groupId>
              <artifactId>auto-value</artifactId>
              <version>${auto-value.version}</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>build-helper-maven-plugin</artifactId>
        <version>3.6.0</version>
        <executions>
          <execution>
            <id>add-source</id>
            <phase>generate-sources</phase>
            <goals>
              <goal>add-source</goal>
            </goals>
            <configuration>
              <sources>
                <source>.</source>
              </sources>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.2.0</version>
        <configuration>
          <mainClass>${exec.mainClass}</mainClass>
          <classpathScope>runtime</classpathScope>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### **Creating Live Audio Run tool**

Create the **LiveAudioRun.java** file under the `src/main/java/` directory with the following content. This tool runs the agent on it with live audio input and output.



### **Run the Live Audio Run tool**

To run Live Audio Run tool, use the following command on the `adk-agents` directory:

```
mvn compile exec:java
```

Then you should see:

```
$ mvn compile exec:java
...
Initializing microphone input and speaker output...
Conversation started. Press Enter to stop...
Speaker initialized.
Microphone initialized. Start speaking...
```

With this message, the tool is ready to take voice input. Talk to the agent with a question like `What's the electron?`.

!!! Caution
    When you observe the agent keep speaking by itself and doesn't stop, try using earphones to suppress the echoing.

## **Summary** {#summary}

Streaming for ADK enables developers to create agents capable of low-latency, bidirectional voice and video communication, enhancing interactive experiences. The article demonstrates that text streaming is a built-in feature of ADK Agents, requiring no additional specific code, while also showcasing how to implement live audio conversations for real-time voice interaction with an agent. This allows for more natural and dynamic communication, as users can speak to and hear from the agent seamlessly.


# Quickstart (Streaming / Python) {#adk-streaming-quickstart}

With this quickstart, you'll learn to create a simple agent and use ADK Streaming to enable voice and video communication with it that is low-latency and bidirectional. We will install ADK, set up a basic "Google Search" agent, try running the agent with Streaming with `adk web` tool, and then explain how to build a simple asynchronous web app by yourself using ADK Streaming and [FastAPI](https://fastapi.tiangolo.com/).

**Note:** This guide assumes you have experience using a terminal in Windows, Mac, and Linux environments.

## Supported models for voice/video streaming {#supported-models}

In order to use voice/video streaming in ADK, you will need to use Gemini models that support the Live API. You can find the **model ID(s)** that supports the Gemini Live API in the documentation:

- [Google AI Studio: Gemini Live API](https://ai.google.dev/gemini-api/docs/models#live-api)
- [Vertex AI: Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api)

## 1. Setup Environment & Install ADK {#1.-setup-installation}

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
pip install google-adk
```

## 2. Project Structure {#2.-project-structure}

Create the following folder structure with empty files:

```console
adk-streaming/  # Project folder
└── app/ # the web app folder
    ├── .env # Gemini API key
    └── google_search_agent/ # Agent folder
        ├── __init__.py # Python package
        └── agent.py # Agent definition
```

### agent.py

Copy-paste the following code block into the `agent.py` file.

For `model`, please double check the model ID as described earlier in the [Models section](#supported-models).

```py
from google.adk.agents import Agent
from google.adk.tools import google_search  # Import the tool

root_agent = Agent(
   # A unique name for the agent.
   name="basic_search_agent",
   # The Large Language Model (LLM) that agent will use.
   # Please fill in the latest model id that supports live from
   # https://google.github.io/adk-docs/get-started/streaming/quickstart-streaming/#supported-models
   model="...",  # for example: model="gemini-2.5-flash-live-001" or model="gemini-2.5-flash-live-preview-04-09"
   # A short description of the agent's purpose.
   description="Agent to answer questions using Google Search.",
   # Instructions to set the agent's behavior.
   instruction="You are an expert researcher. You always stick to the facts.",
   # Add google_search tool to perform grounding with Google search.
   tools=[google_search]
)
```

`agent.py` is where all your agent(s)' logic will be stored, and you must have a `root_agent` defined.

Notice how easily you integrated [grounding with Google Search](https://ai.google.dev/gemini-api/docs/grounding?lang=python#configure-search) capabilities.  The `Agent` class and the `google_search` tool handle the complex interactions with the LLM and grounding with the search API, allowing you to focus on the agent's *purpose* and *behavior*.

![intro_components.png](../../assets/quickstart-streaming-tool.png)

Copy-paste the following code block to `__init__.py` file.

```py title="__init__.py"
from . import agent
```

## 3\. Set up the platform {#3.-set-up-the-platform}

To run the agent, choose a platform from either Google AI Studio or Google Cloud Vertex AI:

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

## 4. Try the agent with `adk web` {#4.-try-it-adk-web}

Now it's ready to try the agent. Run the following command to launch the **dev UI**. First, make sure to set the current directory to `app`:

```shell
cd app
```

Also, set `SSL_CERT_FILE` variable with the following command. This is required for the voice and video tests later.

```shell
export SSL_CERT_FILE=$(python -m certifi)
```

Then, run the dev UI:

```shell
adk web
```

!!!info "Note for Windows users"

    When hitting the `_make_subprocess_transport NotImplementedError`, consider using `adk web --no-reload` instead.


Open the URL provided (usually `http://localhost:8000` or
`http://127.0.0.1:8000`) **directly in your browser**. This connection stays
entirely on your local machine. Select `google_search_agent`.

### Try with text

Try the following prompts by typing them in the UI.

* What is the weather in New York?
* What is the time in New York?
* What is the weather in Paris?
* What is the time in Paris?

The agent will use the google_search tool to get the latest information to answer those questions.

### Try with voice and video

To try with voice, reload the web browser, click the microphone button to enable the voice input, and ask the same question in voice. You will hear the answer in voice in real-time.

To try with video, reload the web browser, click the camera button to enable the video input, and ask questions like "What do you see?". The agent will answer what they see in the video input.

(Just clicking the microphone or camera button once is enough. Your voice or video will be streamed to models and the model response will be streamed back continuously. Clicking on the microphone or camera button multiple times is not supported.)

### Stop the tool

Stop `adk web` by pressing `Ctrl-C` on the console.

### Note on ADK Streaming

The following features will be supported in the future versions of the ADK Streaming: Callback, LongRunningTool, ExampleTool, and Shell agent (e.g. SequentialAgent).

Congratulations\! You've successfully created and interacted with your first Streaming agent using ADK\!

## Next steps: build custom streaming app

In [Custom Audio Streaming app](../../streaming/custom-streaming.md) tutorial, it overviews the server and client code for a custom asynchronous web app built with ADK Streaming and [FastAPI](https://fastapi.tiangolo.com/), enabling real-time, bidirectional audio and text communication.


# Testing your Agents

Before you deploy your agent, you should test it to ensure that it is working as
intended. The easiest way to test your agent in your development environment is
to use the ADK web UI with the following commands. 

=== "Python"

    ```py
    adk api_server
    ```

=== "Java"

    Make sure to update the port number.

    
    In Java, both the Dev UI and the API server are bundled together.

This command will launch a local web
server, where you can run cURL commands or send API requests to test your agent.

## Local testing

Local testing involves launching a local web server, creating a session, and
sending queries to your agent. First, ensure you are in the correct working
directory:

```console
parent_folder/
└── my_sample_agent/
    └── agent.py (or Agent.java)
```

**Launch the Local Server**

Next, launch the local server using the commands listed above.

The output should appear similar to:

=== "Python"

    ```shell
    INFO:     Started server process [12345]
    INFO:     Waiting for application startup.
    INFO:     Application startup complete.
    INFO:     Uvicorn running on http://localhost:8000 (Press CTRL+C to quit)
    ```

=== "Java"

    ```shell
    2025-05-13T23:32:08.972-06:00  INFO 37864 --- [ebServer.main()] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path '/'
    2025-05-13T23:32:08.980-06:00  INFO 37864 --- [ebServer.main()] com.google.adk.web.AdkWebServer          : Started AdkWebServer in 1.15 seconds (process running for 2.877)
    2025-05-13T23:32:08.981-06:00  INFO 37864 --- [ebServer.main()] com.google.adk.web.AdkWebServer          : AdkWebServer application started successfully.
    ```

Your server is now running locally. Ensure you use the correct **_port number_** in all the subsequent commands.

**Create a new session**

With the API server still running, open a new terminal window or tab and create
a new session with the agent using:

```shell
curl -X POST http://localhost:8000/apps/my_sample_agent/users/u_123/sessions/s_123 \
  -H "Content-Type: application/json" \
  -d '{"state": {"key1": "value1", "key2": 42}}'
```

Let's break down what's happening:

* `http://localhost:8000/apps/my_sample_agent/users/u_123/sessions/s_123`: This
  creates a new session for your agent `my_sample_agent`, which is the name of
  the agent folder, for a user ID (`u_123`) and for a session ID (`s_123`). You
