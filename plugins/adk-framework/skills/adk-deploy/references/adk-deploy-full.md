# Deploy to Vertex AI Agent Engine

![python_only](https://img.shields.io/badge/Currently_supported_in-Python-blue){ title="Vertex AI Agent Engine currently supports only Python."}

[Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
is a fully managed Google Cloud service enabling developers to deploy, manage,
and scale AI agents in production. Agent Engine handles the infrastructure to
scale agents in production so you can focus on creating intelligent and
impactful applications.

```python
from vertexai import agent_engines

remote_app = agent_engines.create(
    agent_engine=root_agent,
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]",
    ]
)
```

## Install Vertex AI SDK

Agent Engine is part of the Vertex AI SDK for Python. For more information, you can review the [Agent Engine quickstart documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/quickstart).

### Install the Vertex AI SDK

```shell
pip install google-cloud-aiplatform[adk,agent_engines]
```

!!!info
    Agent Engine only supported Python version >=3.10 and <=3.12.

### Initialization

```py
import vertexai

PROJECT_ID = "your-project-id"
LOCATION = "us-central1"
STAGING_BUCKET = "gs://your-google-cloud-storage-bucket"

vertexai.init(
    project=PROJECT_ID,
    location=LOCATION,
    staging_bucket=STAGING_BUCKET,
)
```

For `LOCATION`, you can check out the list of [supported regions in Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview#supported-regions).

### Create your agent

You can use the sample agent below, which has two tools (to get weather or retrieve the time in a specified city):

```python
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

### Prepare your agent for Agent Engine

Use `reasoning_engines.AdkApp()` to wrap your agent to make it deployable to Agent Engine

```py
from vertexai.preview import reasoning_engines

app = reasoning_engines.AdkApp(
    agent=root_agent,
    enable_tracing=True,
)
```

### Try your agent locally

You can try it locally before deploying to Agent Engine.

#### Create session (local)

```py
session = app.create_session(user_id="u_123")
session
```

Expected output for `create_session` (local):

```console
Session(id='c6a33dae-26ef-410c-9135-b434a528291f', app_name='default-app-name', user_id='u_123', state={}, events=[], last_update_time=1743440392.8689594)
```

#### List sessions (local)

```py
app.list_sessions(user_id="u_123")
```

Expected output for `list_sessions` (local):

```console
ListSessionsResponse(session_ids=['c6a33dae-26ef-410c-9135-b434a528291f'])
```

#### Get a specific session (local)

```py
session = app.get_session(user_id="u_123", session_id=session.id)
session
```

Expected output for `get_session` (local):

```console
Session(id='c6a33dae-26ef-410c-9135-b434a528291f', app_name='default-app-name', user_id='u_123', state={}, events=[], last_update_time=1743681991.95696)
```

#### Send queries to your agent (local)

```py
for event in app.stream_query(
    user_id="u_123",
    session_id=session.id,
    message="whats the weather in new york",
):
print(event)
```

Expected output for `stream_query` (local):

```console
{'parts': [{'function_call': {'id': 'af-a33fedb0-29e6-4d0c-9eb3-00c402969395', 'args': {'city': 'new york'}, 'name': 'get_weather'}}], 'role': 'model'}
{'parts': [{'function_response': {'id': 'af-a33fedb0-29e6-4d0c-9eb3-00c402969395', 'name': 'get_weather', 'response': {'status': 'success', 'report': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}}}], 'role': 'user'}
{'parts': [{'text': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}], 'role': 'model'}
```

### Deploy your agent to Agent Engine

```python
from vertexai import agent_engines

remote_app = agent_engines.create(
    agent_engine=root_agent,
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]"   
    ]
)
```

This step may take several minutes to finish. Each deployed agent has a unique identifier. You can run the following command to get the resource_name identifier for your deployed agent:

```python
remote_app.resource_name
```

The response should look like the following string:

```
f"projects/{PROJECT_NUMBER}/locations/{LOCATION}/reasoningEngines/{RESOURCE_ID}"
```

For additional details, you can visit the Agent Engine documentation [deploying an agent](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy) and [managing deployed agents](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/manage/overview).

### Try your agent on Agent Engine

#### Create session (remote)

```py
remote_session = remote_app.create_session(user_id="u_456")
remote_session
```

Expected output for `create_session` (remote):

```console
{'events': [],
'user_id': 'u_456',
'state': {},
'id': '7543472750996750336',
'app_name': '7917477678498709504',
'last_update_time': 1743683353.030133}
```

`id` is the session ID, and `app_name` is the resource ID of the deployed agent on Agent Engine.

#### List sessions (remote)

```py
remote_app.list_sessions(user_id="u_456")
```

#### Get a specific session (remote)

```py
remote_app.get_session(user_id="u_456", session_id=remote_session["id"])
```

!!!note
    While using your agent locally, session ID is stored in `session.id`, when using your agent remotely on Agent Engine, session ID is stored in `remote_session["id"]`.

#### Send queries to your agent (remote)

```py
for event in remote_app.stream_query(
    user_id="u_456",
    session_id=remote_session["id"],
    message="whats the weather in new york",
):
    print(event)
```

Expected output for `stream_query` (remote):

```console
{'parts': [{'function_call': {'id': 'af-f1906423-a531-4ecf-a1ef-723b05e85321', 'args': {'city': 'new york'}, 'name': 'get_weather'}}], 'role': 'model'}
{'parts': [{'function_response': {'id': 'af-f1906423-a531-4ecf-a1ef-723b05e85321', 'name': 'get_weather', 'response': {'status': 'success', 'report': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}}}], 'role': 'user'}
{'parts': [{'text': 'The weather in New York is sunny with a temperature of 25 degrees Celsius (41 degrees Fahrenheit).'}], 'role': 'model'}
```



## Clean up

After you have finished, it is a good practice to clean up your cloud resources.
You can delete the deployed Agent Engine instance to avoid any unexpected
charges on your Google Cloud account.

```python
remote_app.delete(force=True)
```

`force=True` will also delete any child resources that were generated from the deployed agent, such as sessions.


# Deploy to Cloud Run

[Cloud Run](https://cloud.google.com/run)
is a fully managed platform that enables you to run your code directly on top of Google's scalable infrastructure.

To deploy your agent, you can use either the `adk deploy cloud_run` command _(recommended for Python)_, or with `gcloud run deploy` command through Cloud Run.

## Agent sample

For each of the commands, we will reference a the `Capital Agent` sample defined on the [LLM agent](../agents/llm-agents.md) page. We will assume it's in a directory (eg: `capital_agent`).

To proceed, confirm that your agent code is configured as follows:

=== "Python"

    1. Agent code is in a file called `agent.py` within your agent directory.
    2. Your agent variable is named `root_agent`.
    3. `__init__.py` is within your agent directory and contains `from . import agent`.

=== "Java"

    1. Agent code is in a file called `CapitalAgent.java` within your agent directory.
    2. Your agent variable is global and follows the format `public static BaseAgent ROOT_AGENT`.
    3. Your agent definition is present in a static class method.

    Refer to the following section for more details. You can also find a [sample app](https://github.com/google/adk-docs/tree/main/examples/java/cloud-run) in the Github repo.

## Environment variables

Set your environment variables as described in the [Setup and Installation](../get-started/installation.md) guide.

```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1 # Or your preferred location
export GOOGLE_GENAI_USE_VERTEXAI=True
```

*(Replace `your-project-id` with your actual GCP project ID)*

## Deployment commands

=== "Python - adk CLI"

    ###  adk CLI

    The `adk deploy cloud_run` command deploys your agent code to Google Cloud Run.

    Ensure you have authenticated with Google Cloud (`gcloud auth login` and `gcloud config set project <your-project-id>`).

    #### Setup environment variables

    Optional but recommended: Setting environment variables can make the deployment commands cleaner.

    ```bash
    # Set your Google Cloud Project ID
    export GOOGLE_CLOUD_PROJECT="your-gcp-project-id"

    # Set your desired Google Cloud Location
    export GOOGLE_CLOUD_LOCATION="us-central1" # Example location

    # Set the path to your agent code directory
    export AGENT_PATH="./capital_agent" # Assuming capital_agent is in the current directory

    # Set a name for your Cloud Run service (optional)
    export SERVICE_NAME="capital-agent-service"

    # Set an application name (optional)
    export APP_NAME="capital-agent-app"
    ```

    #### Command usage

    ##### Minimal command

    ```bash
    adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    $AGENT_PATH
    ```

    ##### Full command with optional flags

    ```bash
    adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    --service_name=$SERVICE_NAME \
    --app_name=$APP_NAME \
    --with_ui \
    $AGENT_PATH
    ```

    ##### Arguments

    * `AGENT_PATH`: (Required) Positional argument specifying the path to the directory containing your agent's source code (e.g., `$AGENT_PATH` in the examples, or `capital_agent/`). This directory must contain at least an `__init__.py` and your main agent file (e.g., `agent.py`).

    ##### Options

    * `--project TEXT`: (Required) Your Google Cloud project ID (e.g., `$GOOGLE_CLOUD_PROJECT`).
    * `--region TEXT`: (Required) The Google Cloud location for deployment (e.g., `$GOOGLE_CLOUD_LOCATION`, `us-central1`).
    * `--service_name TEXT`: (Optional) The name for the Cloud Run service (e.g., `$SERVICE_NAME`). Defaults to `adk-default-service-name`.
    * `--app_name TEXT`: (Optional) The application name for the ADK API server (e.g., `$APP_NAME`). Defaults to the name of the directory specified by `AGENT_PATH` (e.g., `capital_agent` if `AGENT_PATH` is `./capital_agent`).
    * `--agent_engine_id TEXT`: (Optional) If you are using a managed session service via Vertex AI Agent Engine, provide its resource ID here.
    * `--port INTEGER`: (Optional) The port number the ADK API server will listen on within the container. Defaults to 8000.
    * `--with_ui`: (Optional) If included, deploys the ADK dev UI alongside the agent API server. By default, only the API server is deployed.
    * `--temp_folder TEXT`: (Optional) Specifies a directory for storing intermediate files generated during the deployment process. Defaults to a timestamped folder in the system's temporary directory. *(Note: This option is generally not needed unless troubleshooting issues).*
    * `--help`: Show the help message and exit.

    ##### Authenticated access 
    During the deployment process, you might be prompted: `Allow unauthenticated invocations to [your-service-name] (y/N)?`.

    * Enter `y` to allow public access to your agent's API endpoint without authentication.
    * Enter `N` (or press Enter for the default) to require authentication (e.g., using an identity token as shown in the "Testing your agent" section).

    Upon successful execution, the command will deploy your agent to Cloud Run and provide the URL of the deployed service.

=== "Python - gcloud CLI"

    ### gcloud CLI

    Alternatively, you can deploy using the standard `gcloud run deploy` command with a `Dockerfile`. This method requires more manual setup compared to the `adk` command but offers flexibility, particularly if you want to embed your agent within a custom [FastAPI](https://fastapi.tiangolo.com/) application.

    Ensure you have authenticated with Google Cloud (`gcloud auth login` and `gcloud config set project <your-project-id>`).

    #### Project Structure

    Organize your project files as follows:

    ```txt
    your-project-directory/
    ├── capital_agent/
    │   ├── __init__.py
    │   └── agent.py       # Your agent code (see "Agent sample" tab)
    ├── main.py            # FastAPI application entry point
    ├── requirements.txt   # Python dependencies
    └── Dockerfile         # Container build instructions
    ```

    Create the following files (`main.py`, `requirements.txt`, `Dockerfile`) in the root of `your-project-directory/`.

    #### Code files

    1. This file sets up the FastAPI application using `get_fast_api_app()` from ADK:

        ```python title="main.py"
        import os

        import uvicorn
        from google.adk.cli.fast_api import get_fast_api_app

        # Get the directory where main.py is located
        AGENT_DIR = os.path.dirname(os.path.abspath(__file__))
        # Example session DB URL (e.g., SQLite)
        SESSION_DB_URL = "sqlite:///./sessions.db"
        # Example allowed origins for CORS
        ALLOWED_ORIGINS = ["http://localhost", "http://localhost:8080", "*"]
        # Set web=True if you intend to serve a web interface, False otherwise
        SERVE_WEB_INTERFACE = True

        # Call the function to get the FastAPI app instance
        # Ensure the agent directory name ('capital_agent') matches your agent folder
        app = get_fast_api_app(
            agents_dir=AGENT_DIR,
            session_service_uri=SESSION_DB_URL,
            allow_origins=ALLOWED_ORIGINS,
            web=SERVE_WEB_INTERFACE,
        )

        # You can add more FastAPI routes or configurations below if needed
        # Example:
        # @app.get("/hello")
        # async def read_root():
        #     return {"Hello": "World"}

        if __name__ == "__main__":
            # Use the PORT environment variable provided by Cloud Run, defaulting to 8080
            uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
        ```

        *Note: We specify `agent_dir` to the directory `main.py` is in and use `os.environ.get("PORT", 8080)` for Cloud Run compatibility.*

    2. List the necessary Python packages:

        ```txt title="requirements.txt"
        google_adk
        # Add any other dependencies your agent needs
        ```

    3. Define the container image:

        ```dockerfile title="Dockerfile"
        FROM python:3.13-slim
        WORKDIR /app

        COPY requirements.txt .
        RUN pip install --no-cache-dir -r requirements.txt

        RUN adduser --disabled-password --gecos "" myuser && \
            chown -R myuser:myuser /app

        COPY . .

        USER myuser

        ENV PATH="/home/myuser/.local/bin:$PATH"

        CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port $PORT"]
        ```

    #### Defining Multiple Agents

    You can define and deploy multiple agents within the same Cloud Run instance by creating separate folders in the root of `your-project-directory/`. Each folder represents one agent and must define a `root_agent` in its configuration.

    Example structure:

    ```txt
    your-project-directory/
    ├── capital_agent/
    │   ├── __init__.py
    │   └── agent.py       # contains `root_agent` definition
    ├── population_agent/
    │   ├── __init__.py
    │   └── agent.py       # contains `root_agent` definition
    └── ...
    ```

    #### Deploy using `gcloud`

    Navigate to `your-project-directory` in your terminal.

    ```bash
    gcloud run deploy capital-agent-service \
    --source . \
    --region $GOOGLE_CLOUD_LOCATION \
    --project $GOOGLE_CLOUD_PROJECT \
    --allow-unauthenticated \
    --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=$GOOGLE_GENAI_USE_VERTEXAI"
    # Add any other necessary environment variables your agent might need
    ```

    * `capital-agent-service`: The name you want to give your Cloud Run service.
    * `--source .`: Tells gcloud to build the container image from the Dockerfile in the current directory.
    * `--region`: Specifies the deployment region.
    * `--project`: Specifies the GCP project.
    * `--allow-unauthenticated`: Allows public access to the service. Remove this flag for private services.
    * `--set-env-vars`: Passes necessary environment variables to the running container. Ensure you include all variables required by ADK and your agent (like API keys if not using Application Default Credentials).

    `gcloud` will build the Docker image, push it to Google Artifact Registry, and deploy it to Cloud Run. Upon completion, it will output the URL of your deployed service.

    For a full list of deployment options, see the [`gcloud run deploy` reference documentation](https://cloud.google.com/sdk/gcloud/reference/run/deploy).


=== "Java - gcloud CLI"

    ### gcloud CLI

    You can deploy Java Agents using the standard `gcloud run deploy` command and a `Dockerfile`. This is the current recommended way to deploy Java Agents to Google Cloud Run.

    Ensure you are [authenticated](https://cloud.google.com/docs/authentication/gcloud) with Google Cloud.
    Specifically, run the commands `gcloud auth login` and `gcloud config set project <your-project-id>` from your terminal.

    #### Project Structure

    Organize your project files as follows:

    ```txt
    your-project-directory/
    ├── src/
    │   └── main/
    │       └── java/
    │             └── agents/
    │                 ├── capitalagent/
    │                     └── CapitalAgent.java    # Your agent code
    ├── pom.xml                                    # Java adk and adk-dev dependencies
    └── Dockerfile                                 # Container build instructions
    ```

    Create the `pom.xml` and `Dockerfile` in the root of your project directory. Your Agent code file (`CapitalAgent.java`) inside a directory as shown above.

    #### Code files

    1. This is our Agent definition. This is the same code as present in [LLM agent](../agents/llm-agents.md) with two caveats:
       
           * The Agent is now initialized as a **global public static variable**.
    
           * The definition of the agent can be exposed in a static method or inlined during declaration.

        

    2. Add the following dependencies and plugin to the pom.xml file.

        ```xml title="pom.xml"
        <dependencies>
          <dependency>
             <groupId>com.google.adk</groupId>
             <artifactId>google-adk</artifactId>
             <version>0.1.0</version>
          </dependency>
          <dependency>
             <groupId>com.google.adk</groupId>
             <artifactId>google-adk-dev</artifactId>
             <version>0.1.0</version>
          </dependency>
        </dependencies>
        
        <plugin>
          <groupId>org.codehaus.mojo</groupId>
          <artifactId>exec-maven-plugin</artifactId>
          <version>3.2.0</version>
          <configuration>
            <mainClass>com.google.adk.web.AdkWebServer</mainClass>
            <classpathScope>compile</classpathScope>
          </configuration>
        </plugin>
        ```

    3.  Define the container image:

        ```dockerfile title="Dockerfile"
        # Use an official Maven image with a JDK. Choose a version appropriate for your project.
        FROM maven:3.8-openjdk-17 AS builder

        WORKDIR /app

        COPY pom.xml .
        RUN mvn dependency:go-offline -B

        COPY src ./src

        # Expose the port your application will listen on.
        # Cloud Run will set the PORT environment variable, which your app should use.
        EXPOSE 8080

        # The command to run your application.
        # TODO(Developer): Update the "adk.agents.source-dir" to the directory that contains your agents.
        # You can have multiple agents in this directory and all of them will be available in the Dev UI.
        ENTRYPOINT ["mvn", "exec:java", \
            "-Dexec.mainClass=com.google.adk.web.AdkWebServer", \
            "-Dexec.classpathScope=compile", \
            "-Dexec.args=--server.port=${PORT} --adk.agents.source-dir=src/main/java" \
        ]
        ```

    #### Deploy using `gcloud`

    Navigate to `your-project-directory` in your terminal.

    ```bash
    gcloud run deploy capital-agent-service \
    --source . \
    --region $GOOGLE_CLOUD_LOCATION \
    --project $GOOGLE_CLOUD_PROJECT \
    --allow-unauthenticated \
    --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=$GOOGLE_GENAI_USE_VERTEXAI"
    # Add any other necessary environment variables your agent might need
    ```

    * `capital-agent-service`: The name you want to give your Cloud Run service.
    * `--source .`: Tells gcloud to build the container image from the Dockerfile in the current directory.
    * `--region`: Specifies the deployment region.
    * `--project`: Specifies the GCP project.
    * `--allow-unauthenticated`: Allows public access to the service. Remove this flag for private services.
    * `--set-env-vars`: Passes necessary environment variables to the running container. Ensure you include all variables required by ADK and your agent (like API keys if not using Application Default Credentials).

    `gcloud` will build the Docker image, push it to Google Artifact Registry, and deploy it to Cloud Run. Upon completion, it will output the URL of your deployed service.

    For a full list of deployment options, see the [`gcloud run deploy` reference documentation](https://cloud.google.com/sdk/gcloud/reference/run/deploy).



## Testing your agent

Once your agent is deployed to Cloud Run, you can interact with it via the deployed UI (if enabled) or directly with its API endpoints using tools like `curl`. You'll need the service URL provided after deployment.

=== "UI Testing"

    ### UI Testing

    If you deployed your agent with the UI enabled:

    *   **adk CLI:** You included the `--with_ui` flag during deployment.
    *   **gcloud CLI:** You set `SERVE_WEB_INTERFACE = True` in your `main.py`.

    You can test your agent by simply navigating to the Cloud Run service URL provided after deployment in your web browser.

    ```bash
    # Example URL format
    # https://your-service-name-abc123xyz.a.run.app
    ```

    The ADK dev UI allows you to interact with your agent, manage sessions, and view execution details directly in the browser.

    To verify your agent is working as intended, you can:

    1. Select your agent from the dropdown menu.
    2. Type a message and verify that you receive an expected response from your agent.

    If you experience any unexpected behavior, check the [Cloud Run](https://console.cloud.google.com/run) console logs.

=== "API Testing (curl)"

    ### API Testing (curl)

    You can interact with the agent's API endpoints using tools like `curl`. This is useful for programmatic interaction or if you deployed without the UI.

    You'll need the service URL provided after deployment and potentially an identity token for authentication if your service isn't set to allow unauthenticated access.

    #### Set the application URL

    Replace the example URL with the actual URL of your deployed Cloud Run service.

    ```bash
    export APP_URL="YOUR_CLOUD_RUN_SERVICE_URL"
    # Example: export APP_URL="https://adk-default-service-name-abc123xyz.a.run.app"
    ```

    #### Get an identity token (if needed)

    If your service requires authentication (i.e., you didn't use `--allow-unauthenticated` with `gcloud` or answered 'N' to the prompt with `adk`), obtain an identity token.

    ```bash
    export TOKEN=$(gcloud auth print-identity-token)
    ```

    *If your service allows unauthenticated access, you can omit the `-H "Authorization: Bearer $TOKEN"` header from the `curl` commands below.*

    #### List available apps

    Verify the deployed application name.

    ```bash
    curl -X GET -H "Authorization: Bearer $TOKEN" $APP_URL/list-apps
    ```

    *(Adjust the `app_name` in the following commands based on this output if needed. The default is often the agent directory name, e.g., `capital_agent`)*.

    #### Create or Update a Session

    Initialize or update the state for a specific user and session. Replace `capital_agent` with your actual app name if different. The values `user_123` and `session_abc` are example identifiers; you can replace them with your desired user and session IDs.

    ```bash
    curl -X POST -H "Authorization: Bearer $TOKEN" \
        $APP_URL/apps/capital_agent/users/user_123/sessions/session_abc \
        -H "Content-Type: application/json" \
        -d '{"state": {"preferred_language": "English", "visit_count": 5}}'
    ```

    #### Run the Agent

    Send a prompt to your agent. Replace `capital_agent` with your app name and adjust the user/session IDs and prompt as needed.

    ```bash
    curl -X POST -H "Authorization: Bearer $TOKEN" \
        $APP_URL/run_sse \
        -H "Content-Type: application/json" \
        -d '{
        "app_name": "capital_agent",
        "user_id": "user_123",
        "session_id": "session_abc",
        "new_message": {
            "role": "user",
            "parts": [{
            "text": "What is the capital of Canada?"
            }]
        },
        "streaming": false
        }'
    ```

    * Set `"streaming": true` if you want to receive Server-Sent Events (SSE).
    * The response will contain the agent's execution events, including the final answer.


# Deploy to GKE

[GKE](https://cloud.google.com/gke) is Google Clouds managed Kubernetes service. It allows you to deploy and manage containerized applications using Kubernetes.

To deploy your agent you will need to have a Kubernetes cluster running on GKE. You can create a cluster using the Google Cloud Console or the `gcloud` command line tool.

In this example we will deploy a simple agent to GKE. The agent will be a FastAPI application that uses `Gemini 2.0 Flash` as the LLM. We can use Vertex AI or AI Studio as the LLM provider using a Environment variable.

## Agent sample

For each of the commands, we will reference a `capital_agent` sample defined in on the [LLM agent](../agents/llm-agents.md) page. We will assume it's in a `capital_agent` directory.

To proceed, confirm that your agent code is configured as follows:

1. Agent code is in a file called `agent.py` within your agent directory.
2. Your agent variable is named `root_agent`.
3. `__init__.py` is within your agent directory and contains `from . import agent`.

## Environment variables

Set your environment variables as described in the [Setup and Installation](../get-started/installation.md) guide. You also need to install the `kubectl` command line tool. You can find instructions to do so in the [Google Kubernetes Engine Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl).

```bash
export GOOGLE_CLOUD_PROJECT=your-project-id # Your GCP project ID
export GOOGLE_CLOUD_LOCATION=us-central1 # Or your preferred location
export GOOGLE_GENAI_USE_VERTEXAI=true # Set to true if using Vertex AI
export GOOGLE_CLOUD_PROJECT_NUMBER=$(gcloud projects describe --format json $GOOGLE_CLOUD_PROJECT | jq -r ".projectNumber")
```

If you don't have `jq` installed, you can use the following command to get the project number:

```bash
gcloud projects describe $GOOGLE_CLOUD_PROJECT
```

And copy the project number from the output.

```bash
export GOOGLE_CLOUD_PROJECT_NUMBER=YOUR_PROJECT_NUMBER
```

## Deployment options

### Option 1: Manual Deployment using gcloud and kubectl

You can deploy your agent to GKE either **manually using Kubernetes manifests** or **automatically using the `adk deploy gke` command**. Choose the approach that best suits your workflow.

Ensure you have authenticated with Google Cloud (`gcloud auth login` and `gcloud config set project <your-project-id>`).

### Enable APIs

Enable the necessary APIs for your project. You can do this using the `gcloud` command line tool.

```bash
gcloud services enable \
    container.googleapis.com \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com \
    aiplatform.googleapis.com
```
### Option 1: Manual Deployment using gcloud and kubectl

### Create a GKE cluster

You can create a GKE cluster using the `gcloud` command line tool. This example creates an Autopilot cluster named `adk-cluster` in the `us-central1` region.

> If creating a GKE Standard cluster, make sure [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) is enabled. Workload Identity is enabled by default in an AutoPilot cluster.

```bash
gcloud container clusters create-auto adk-cluster \
    --location=$GOOGLE_CLOUD_LOCATION \
    --project=$GOOGLE_CLOUD_PROJECT
```

After creating the cluster, you need to connect to it using `kubectl`. This command configures `kubectl` to use the credentials for your new cluster.

```bash
gcloud container clusters get-credentials adk-cluster \
    --location=$GOOGLE_CLOUD_LOCATION \
    --project=$GOOGLE_CLOUD_PROJECT
```

### Project Structure

Organize your project files as follows:

```txt
your-project-directory/
├── capital_agent/
│   ├── __init__.py
│   └── agent.py       # Your agent code (see "Agent sample" tab)
├── main.py            # FastAPI application entry point
├── requirements.txt   # Python dependencies
└── Dockerfile         # Container build instructions
```

Create the following files (`main.py`, `requirements.txt`, `Dockerfile`) in the root of `your-project-directory/`.

### Code files

1. This file sets up the FastAPI application using `get_fast_api_app()` from ADK:

    ```python title="main.py"
    import os

    import uvicorn
    from fastapi import FastAPI
    from google.adk.cli.fast_api import get_fast_api_app

    # Get the directory where main.py is located
    AGENT_DIR = os.path.dirname(os.path.abspath(__file__))
    # Example session DB URL (e.g., SQLite)
    SESSION_DB_URL = "sqlite:///./sessions.db"
    # Example allowed origins for CORS
    ALLOWED_ORIGINS = ["http://localhost", "http://localhost:8080", "*"]
    # Set web=True if you intend to serve a web interface, False otherwise
    SERVE_WEB_INTERFACE = True

    # Call the function to get the FastAPI app instance
    # Ensure the agent directory name ('capital_agent') matches your agent folder
    app: FastAPI = get_fast_api_app(
        agents_dir=AGENT_DIR,
        session_db_url=SESSION_DB_URL,
        allow_origins=ALLOWED_ORIGINS,
        web=SERVE_WEB_INTERFACE,
    )

    # You can add more FastAPI routes or configurations below if needed
    # Example:
    # @app.get("/hello")
    # async def read_root():
    #     return {"Hello": "World"}

    if __name__ == "__main__":
        # Use the PORT environment variable provided by Cloud Run, defaulting to 8080
        uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
    ```

    *Note: We specify `agent_dir` to the directory `main.py` is in and use `os.environ.get("PORT", 8080)` for Cloud Run compatibility.*

2. List the necessary Python packages:

    ```txt title="requirements.txt"
    google_adk
    # Add any other dependencies your agent needs
    ```

3. Define the container image:

    ```dockerfile title="Dockerfile"
    FROM python:3.13-slim
    WORKDIR /app

    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt

    RUN adduser --disabled-password --gecos "" myuser && \
        chown -R myuser:myuser /app

    COPY . .

    USER myuser

    ENV PATH="/home/myuser/.local/bin:$PATH"

    CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port $PORT"]
    ```

### Build the container image

You need to create a Google Artifact Registry repository to store your container images. You can do this using the `gcloud` command line tool.

```bash
gcloud artifacts repositories create adk-repo \
    --repository-format=docker \
    --location=$GOOGLE_CLOUD_LOCATION \
    --description="ADK repository"
```

Build the container image using the `gcloud` command line tool. This example builds the image and tags it as `adk-repo/adk-agent:latest`.

```bash
gcloud builds submit \
    --tag $GOOGLE_CLOUD_LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/adk-repo/adk-agent:latest \
    --project=$GOOGLE_CLOUD_PROJECT \
    .
```

Verify the image is built and pushed to the Artifact Registry:

```bash
gcloud artifacts docker images list \
  $GOOGLE_CLOUD_LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/adk-repo \
  --project=$GOOGLE_CLOUD_PROJECT
```

### Configure Kubernetes Service Account for Vertex AI

If your agent uses Vertex AI, you need to create a Kubernetes service account with the necessary permissions. This example creates a service account named `adk-agent-sa` and binds it to the `Vertex AI User` role.

> If you are using AI Studio and accessing the model with an API key you can skip this step.

```bash
kubectl create serviceaccount adk-agent-sa
```

```bash
gcloud projects add-iam-policy-binding projects/${GOOGLE_CLOUD_PROJECT} \
    --role=roles/aiplatform.user \
    --member=principal://iam.googleapis.com/projects/${GOOGLE_CLOUD_PROJECT_NUMBER}/locations/global/workloadIdentityPools/${GOOGLE_CLOUD_PROJECT}.svc.id.goog/subject/ns/default/sa/adk-agent-sa \
    --condition=None
```

### Create the Kubernetes manifest files

Create a Kubernetes deployment manifest file named `deployment.yaml` in your project directory. This file defines how to deploy your application on GKE.

```yaml title="deployment.yaml"
cat <<  EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adk-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adk-agent
  template:
    metadata:
      labels:
        app: adk-agent
    spec:
      serviceAccount: adk-agent-sa
      containers:
      - name: adk-agent
        imagePullPolicy: Always
        image: $GOOGLE_CLOUD_LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/adk-repo/adk-agent:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
            ephemeral-storage: "128Mi"
          requests:
            memory: "128Mi"
            cpu: "500m"
            ephemeral-storage: "128Mi"
        ports:
        - containerPort: 8080
        env:
          - name: PORT
            value: "8080"
          - name: GOOGLE_CLOUD_PROJECT
            value: GOOGLE_CLOUD_PROJECT
          - name: GOOGLE_CLOUD_LOCATION
            value: GOOGLE_CLOUD_LOCATION
          - name: GOOGLE_GENAI_USE_VERTEXAI
            value: GOOGLE_GENAI_USE_VERTEXAI
          # If using AI Studio, set GOOGLE_GENAI_USE_VERTEXAI to false and set the following:
          # - name: GOOGLE_API_KEY
          #   value: GOOGLE_API_KEY
          # Add any other necessary environment variables your agent might need
---
apiVersion: v1
kind: Service
metadata:
  name: adk-agent
spec:       
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: adk-agent
EOF
```

### Deploy the Application

Deploy the application using the `kubectl` command line tool. This command applies the deployment and service manifest files to your GKE cluster.

```bash
kubectl apply -f deployment.yaml
```

After a few moments, you can check the status of your deployment using:

```bash
kubectl get pods -l=app=adk-agent
```

This command lists the pods associated with your deployment. You should see a pod with a status of `Running`.

Once the pod is running, you can check the status of the service using:

```bash
kubectl get service adk-agent
```

If the output shows a `External IP`, it means your service is accessible from the internet. It may take a few minutes for the external IP to be assigned.

You can get the external IP address of your service using:

```bash
kubectl get svc adk-agent -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Option 2: Automated Deployment using `adk deploy gke`

ADK provides a CLI command to streamline GKE deployment. This avoids the need to manually build images, write Kubernetes manifests, or push to Artifact Registry.

#### Prerequisites

Before you begin, ensure you have the following set up:

1. **A running GKE cluster:** You need an active Kubernetes cluster on Google Cloud.

2. **`gcloud` CLI:** The Google Cloud CLI must be installed, authenticated, and configured to use your target project. Run `gcloud auth login` and `gcloud config set project [YOUR_PROJECT_ID]`.

3. **Required IAM Permissions:** The user or service account running the command needs, at a minimum, the following roles:

   * **Kubernetes Engine Developer** (`roles/container.developer`): To interact with the GKE cluster.

   * **Artifact Registry Writer** (`roles/artifactregistry.writer`): To push the agent's container image.

4. **Docker:** The Docker daemon must be running on your local machine to build the container image.

### The `deploy gke` Command

The command takes the path to your agent and parameters specifying the target GKE cluster.

#### Syntax

```bash
adk deploy gke [OPTIONS] AGENT_PATH
```

### Arguments & Options

| Argument    | Description | Required |
| -------- | ------- | ------  |
| AGENT_PATH  | The local file path to your agent's root directory.    |Yes |
| --project | The Google Cloud Project ID where your GKE cluster is located.     | Yes | 
| --cluster_name   | The name of your GKE cluster.    | Yes |
| --region    | The Google Cloud region of your cluster (e.g., us-central1).    | Yes |
| --with_ui   | Deploys both the agent's back-end API and a companion front-end user interface.    | No |
| --verbosity   | Sets the logging level for the deployment process. Options: debug, info, warning, error.     | No |


### How It Works
When you run the `adk deploy gke` command, the ADK performs the following steps automatically:

- Containerization: It builds a Docker container image from your agent's source code.

- Image Push: It tags the container image and pushes it to your project's Artifact Registry.

- Manifest Generation: It dynamically generates the necessary Kubernetes manifest files (a `Deployment` and a `Service`).

- Cluster Deployment: It applies these manifests to your specified GKE cluster, which triggers the following:

The `Deployment` instructs GKE to pull the container image from Artifact Registry and run it in one or more Pods.

The `Service` creates a stable network endpoint for your agent. By default, this is a LoadBalancer service, which provisions a public IP address to expose your agent to the internet.


### Example Usage
Here is a practical example of deploying an agent located at `~/agents/multi_tool_agent/` to a GKE cluster named test.

```bash
adk deploy gke \
    --project myproject \
    --cluster_name test \
    --region us-central1 \
    --with_ui \
    --verbosity info \
    ~/agents/multi_tool_agent/
```

### Verifying Your Deployment
If you used `adk deploy gke`, verify the deployment using `kubectl`:

1. Check the Pods: Ensure your agent's pods are in the Running state.

```bash
kubectl get pods
```
You should see output like `adk-default-service-name-xxxx-xxxx ... 1/1 Running` in the default namespace.

2. Find the External IP: Get the public IP address for your agent's service.

```bash
kubectl get service
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
adk-default-service-name   LoadBalancer   34.118.228.70   34.63.153.253   80:32581/TCP   5d20h
```

We can navigate to the external IP and interact with the agent via UI
![alt text](../assets/agent-gke-deployment.png)

## Testing your agent

Once your agent is deployed to GKE, you can interact with it via the deployed UI (if enabled) or directly with its API endpoints using tools like `curl`. You'll need the service URL provided after deployment.

=== "UI Testing"

    ### UI Testing

    If you deployed your agent with the UI enabled:

    You can test your agent by simply navigating to the kubernetes service URL in your web browser.

    The ADK dev UI allows you to interact with your agent, manage sessions, and view execution details directly in the browser.

    To verify your agent is working as intended, you can:

    1. Select your agent from the dropdown menu.
    2. Type a message and verify that you receive an expected response from your agent.

    If you experience any unexpected behavior, check the pod logs for your agent using:

    ```bash
    kubectl logs -l app=adk-agent
    ```

=== "API Testing (curl)"

    ### API Testing (curl)

    You can interact with the agent's API endpoints using tools like `curl`. This is useful for programmatic interaction or if you deployed without the UI.

    #### Set the application URL

    Replace the example URL with the actual URL of your deployed Cloud Run service.

    ```bash
    export APP_URL="KUBERNETES_SERVICE_URL"
    ```
