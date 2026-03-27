---
name: agentscope-deploy
description: >
  Deploy AgentScope agents to production with tracing and observability.
  Use when the user asks to "deploy agent", "add tracing", "monitor agent",
  "deploy to Docker", "deploy to Kubernetes", "deploy to Cloud Run",
  "setup OpenTelemetry for AgentScope", "production deployment", or needs
  guidance on observability, containerization, and scaling AgentScope apps.
metadata:
  version: "0.1.0"
  framework: "agentscope-python"
---

# AgentScope Deploy & Observability

Deploy AgentScope agents từ local đến production với OpenTelemetry tracing.

## Tracing với OpenTelemetry

AgentScope sử dụng OpenTelemetry — tương thích với mọi OTel-compatible backend.

```python
import agentscope

# Khởi tạo global tracing (gọi trước khi tạo agents)
agentscope.init(
    tracing_url="http://localhost:4317",  # OTLP/gRPC endpoint
)

# Built-in traces cho:
# - LLM API calls (latency, tokens, model)
# - Tool executions (function name, args, result)
# - Agent steps (ReAct reasoning steps)
# - Formatter operations
```

### Kết nối Langfuse

```python
import agentscope

agentscope.init(
    tracing_url="https://cloud.langfuse.com/api/public/otel",
    # Set headers qua env vars:
    # OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64_key>"
)
```

### Kết nối Arize Phoenix (local)

```bash
pip install arize-phoenix
python -m phoenix.server.main &  # Start Phoenix at localhost:6006
```

```python
agentscope.init(
    tracing_url="http://localhost:4317",  # Phoenix OTLP endpoint
)
```

### Environment Variables cho OTel

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer your-token"
export OTEL_SERVICE_NAME="my-agentscope-app"
```

## Docker Deployment

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source
COPY . .

# Run
CMD ["python", "main.py"]
```

### requirements.txt

```
agentscope>=0.1.0
python-dotenv>=1.0.0
# mem0ai  # uncomment nếu dùng long-term memory
```

### docker-compose.yml

```yaml
version: "3.9"
services:
  agent:
    build: .
    environment:
      - DASHSCOPE_API_KEY=${DASHSCOPE_API_KEY}
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
    depends_on:
      - otel-collector

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-config.yaml:/etc/otel/config.yaml
    command: ["--config=/etc/otel/config.yaml"]
    ports:
      - "4317:4317"  # gRPC
      - "4318:4318"  # HTTP
```

## Kubernetes Deployment

### Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agentscope-app
  labels:
    app: agentscope-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: agentscope-app
  template:
    metadata:
      labels:
        app: agentscope-app
    spec:
      containers:
      - name: agent
        image: your-registry/agentscope-app:latest
        env:
        - name: DASHSCOPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: dashscope-api-key
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector.monitoring:4317"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Secret cho API keys

```bash
kubectl create secret generic api-keys \
  --from-literal=dashscope-api-key=$DASHSCOPE_API_KEY \
  --from-literal=openai-api-key=$OPENAI_API_KEY
```

## Serverless (Google Cloud Run)

```bash
# Build và push image
docker build -t gcr.io/my-project/agentscope-app .
docker push gcr.io/my-project/agentscope-app

# Deploy
gcloud run deploy agentscope-app \
  --image gcr.io/my-project/agentscope-app \
  --platform managed \
  --region asia-southeast1 \
  --set-env-vars DASHSCOPE_API_KEY=$DASHSCOPE_API_KEY \
  --set-env-vars OTEL_EXPORTER_OTLP_ENDPOINT=$OTEL_ENDPOINT \
  --memory 512Mi \
  --timeout 300
```

## Agent-as-a-Service (HTTP API)

```python
# main.py — Wrap agent với FastAPI
from fastapi import FastAPI
from pydantic import BaseModel
from agentscope.agent import ReActAgent
from agentscope.message import Msg

app = FastAPI()

# Khởi tạo agent một lần
agent = ReActAgent(
    name="API_Agent",
    sys_prompt="You are a helpful API assistant.",
    model=model,
    memory=InMemoryMemory(),
    toolkit=toolkit,
)

class ChatRequest(BaseModel):
    message: str
    session_id: str = "default"

@app.post("/chat")
async def chat(request: ChatRequest):
    msg = Msg("user", request.message, "user")
    response = await agent(msg)
    return {"response": response.get_text_content()}
```

```bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

## Deployment Checklist

- [ ] API keys đặt trong environment variables (không hardcode)
- [ ] `agentscope.init(tracing_url=...)` được gọi trước khi tạo agents
- [ ] Docker image test locally trước khi push
- [ ] Health check endpoint implement nếu dùng K8s/Cloud Run
- [ ] Resource limits (`memory`, `cpu`) được set
- [ ] Graceful shutdown handle (`SIGTERM`)
- [ ] Logging level phù hợp (`INFO` cho production)

## Additional Resources

- **`references/agentscope-deploy-full.md`** — Full deployment reference
- [AgentScope Runtime](https://github.com/agentscope-ai/agentscope-runtime) — Production runtime với secure sandboxing
