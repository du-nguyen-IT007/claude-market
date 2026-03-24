---
name: adk-deploy
description: >
  Deploy Google ADK agents to production environments including Cloud Run,
  Vertex AI Agent Engine, and GKE. Use when the user asks to "deploy agent",
  "deploy to cloud run", "deploy to vertex ai", "deploy to GKE",
  "containerize agent", "production deployment", "adk deploy", "create Dockerfile
  for agent", "FastAPI agent server", or needs guidance on deploying ADK agents
  to any cloud environment.
metadata:
  version: "0.1.0"
  framework: "google-adk-python"
---

# ADK Deploy

Deploy Google ADK agents to production — Cloud Run, Vertex AI Agent Engine, or GKE.

## Deployment Options Overview

| Platform | Best For | Complexity | Scaling |
|----------|---------|------------|---------|
| Vertex AI Agent Engine | Fastest to production | Low | Fully managed |
| Cloud Run (adk CLI) | Simple container deploy | Low | Auto-scaling |
| Cloud Run (gcloud) | Custom FastAPI setup | Medium | Auto-scaling |
| GKE | Full Kubernetes control | High | Kubernetes-managed |

## Prerequisites

### Agent Structure
```
my_agent/
├── __init__.py          # Contains: from . import agent
├── agent.py             # Must export: root_agent
└── requirements.txt     # google-adk + dependencies
```

### Environment Variables
```bash
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1
export GOOGLE_GENAI_USE_VERTEXAI=True
```

## 1. Vertex AI Agent Engine (Simplest)

Fully managed deployment — deploy with ~5 lines:

```python
# Install
# pip install google-cloud-aiplatform[adk,agent_engines]

import vertexai

vertexai.init(
    project="your-project-id",
    location="us-central1",
    staging_bucket="gs://your-bucket"
)

# Deploy
from vertexai import agent_engines

remote_app = agent_engines.create(
    agent_engine=root_agent,
    requirements=["google-cloud-aiplatform[adk,agent_engines]"]
)

print(remote_app.resource_name)
# projects/PROJECT/locations/LOCATION/reasoningEngines/ID
```

### Testing on Agent Engine
```python
# Create session
session = remote_app.create_session(user_id="u_123")

# Query
for event in remote_app.stream_query(
    user_id="u_123",
    session_id=session["id"],
    message="What's the weather?"
):
    print(event)

# List sessions
remote_app.list_sessions(user_id="u_123")
```

### Local Testing First
```python
from vertexai.preview import reasoning_engines

app = reasoning_engines.AdkApp(agent=root_agent, enable_tracing=True)
session = app.create_session(user_id="u_local")

for event in app.stream_query(
    user_id="u_local", session_id=session.id, message="Hello!"
):
    print(event)
```

### Cleanup
```python
remote_app.delete(force=True)  # Deletes sessions too
```

## 2. Cloud Run — adk CLI (Recommended)

```bash
# Authenticate
gcloud auth login
gcloud config set project your-project-id

# Deploy (minimal)
adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    ./my_agent

# Deploy with UI
adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    --service_name=my-agent-service \
    --app_name=my-agent-app \
    --with_ui \
    ./my_agent
```

### CLI Options
| Flag | Default | Description |
|------|---------|-------------|
| `--project` | Required | GCP project ID |
| `--region` | Required | Cloud region |
| `--service_name` | `adk-default-service-name` | Cloud Run service name |
| `--app_name` | Agent directory name | Application name |
| `--with_ui` | Off | Deploy dev UI alongside API |
| `--port` | 8000 | Container port |
| `--agent_engine_id` | None | Vertex AI managed session service |

## 3. Cloud Run — gcloud CLI (Custom FastAPI)

### Project Structure
```
your-project/
├── my_agent/
│   ├── __init__.py
│   └── agent.py
├── main.py
├── requirements.txt
└── Dockerfile
```

### main.py
```python
import os
import uvicorn
from google.adk.cli.fast_api import get_fast_api_app

AGENT_DIR = os.path.dirname(os.path.abspath(__file__))

app = get_fast_api_app(
    agents_dir=AGENT_DIR,
    session_service_uri="sqlite:///./sessions.db",
    allow_origins=["*"],
    web=True  # Enable dev UI
)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

### Dockerfile
```dockerfile
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

### requirements.txt
```
google_adk
```

### Deploy
```bash
gcloud run deploy my-agent-service \
    --source . \
    --region $GOOGLE_CLOUD_LOCATION \
    --project $GOOGLE_CLOUD_PROJECT \
    --allow-unauthenticated \
    --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=True"
```

### Multiple Agents in One Service
```
your-project/
├── agent_a/
│   ├── __init__.py
│   └── agent.py    # root_agent
├── agent_b/
│   ├── __init__.py
│   └── agent.py    # root_agent
├── main.py
└── Dockerfile
```

## 4. GKE Deployment

### Dockerfile (same as Cloud Run)

### Kubernetes Manifests

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adk-agent
spec:
  replicas: 2
  selector:
    matchLabels:
      app: adk-agent
  template:
    metadata:
      labels:
        app: adk-agent
    spec:
      containers:
      - name: adk-agent
        image: gcr.io/PROJECT_ID/adk-agent:latest
        ports:
        - containerPort: 8080
        env:
        - name: GOOGLE_CLOUD_PROJECT
          value: "your-project-id"
        - name: GOOGLE_CLOUD_LOCATION
          value: "us-central1"
        - name: GOOGLE_GENAI_USE_VERTEXAI
          value: "True"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

**service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: adk-agent-service
spec:
  type: LoadBalancer
  selector:
    app: adk-agent
  ports:
  - port: 80
    targetPort: 8080
```

### Deploy to GKE
```bash
# Build and push
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/adk-agent

# Apply
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## 5. Testing Deployed Agents

### Via curl (API)
```bash
SERVICE_URL="https://your-service.run.app"

# Create session
curl -X POST "$SERVICE_URL/apps/my_agent/users/u1/sessions" \
  -H "Content-Type: application/json" \
  -d '{}'

# Send query
curl -X POST "$SERVICE_URL/run_sse" \
  -H "Content-Type: application/json" \
  -d '{"app_name":"my_agent","user_id":"u1","session_id":"SESSION_ID","new_message":{"role":"user","parts":[{"text":"Hello!"}]}}'
```

### Via UI
Navigate to the service URL in browser (if `--with_ui` was used).

## 6. Dev Server (Local)

```bash
# Start dev UI
adk web ./my_agent
# Opens at http://localhost:8000

# API server only
adk api_server ./my_agent
```

## Production Best Practices

1. **Use DatabaseSessionService** or **VertexAiSessionService** — not InMemory
2. **Set up proper authentication** — don't use `--allow-unauthenticated` in production
3. **Use Secret Manager** for API keys — not environment variables
4. **Enable tracing** — `enable_tracing=True` for observability
5. **Set resource limits** in Kubernetes/Cloud Run
6. **Use health checks** for container orchestrators
7. **Version your agents** — use semantic versioning in deployment tags

## Additional Resources

- **`references/adk-deploy-full.md`** — Complete ADK deployment documentation for all platforms
