# Production-Style AI Agent on Kubernetes — Complete Hands-On Lab Series

## Overview

This lab series builds a small production-style AI agent from scratch and gradually introduces:

1. Claude API integration
2. Agent tools
3. Tool-calling loop
4. Docker containerization
5. Kubernetes Deployment
6. Kubernetes Service
7. Kubernetes Secrets
8. Health probes
9. Horizontal Pod Autoscaler (HPA)
10. Concurrent load testing
11. Queue-based workers
12. Prometheus and Grafana observability
13. Retry, timeout, and rate-limit handling
14. Multi-agent orchestration
15. Production architecture

### Target environment

- Ubuntu
- Python 3.10+
- Docker
- Minikube
- kubectl
- Claude/Anthropic API key
- Internet access from the machine running the agent

> **Important:** Claude Pro and the Anthropic API are separate products. These labs require an Anthropic API key with API access.

---

# Lab 0 — Prerequisites and Environment Setup

## Objective

Prepare Ubuntu, Docker, Minikube, kubectl, and Python.

## 1. Check Python

```bash
python3 --version
```

Create the project:

```bash
mkdir -p ~/ai-agent-k8s
cd ~/ai-agent-k8s
```

Create a virtual environment:

```bash
python3 -m venv venv
```

Activate it:

```bash
source venv/bin/activate
```

Verify:

```bash
which python
python --version
```

## 2. Check Docker

```bash
docker --version
```

Check:

```bash
docker ps
```

## 3. Check Minikube

```bash
minikube version
```

Start it:

```bash
minikube start
```

Check:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

## 4. Enable useful Minikube addons

```bash
minikube addons enable metrics-server
minikube addons enable ingress
```

Check:

```bash
kubectl get pods -A
```

## 5. Check metrics

Wait a few minutes and run:

```bash
kubectl top nodes
```

If metrics are not available immediately, wait and retry.

---

# Lab 1 — Build a Basic Claude Agent API

## Objective

Create a simple Python HTTP service that sends a user question to Claude.

Architecture:

```text
User
 |
 v
Python API
 |
 v
Claude API
 |
 v
Response
```

## 1. Create the project

```bash
cd ~/ai-agent-k8s
mkdir -p agent
cd agent
```

Create the virtual environment if you have not already:

```bash
python3 -m venv venv
source venv/bin/activate
```

## 2. Install packages

```bash
pip install anthropic fastapi uvicorn python-dotenv
```

Create `requirements.txt`:

```text
anthropic
fastapi
uvicorn[standard]
python-dotenv
```

## 3. Set the API key

For local testing:

```bash
export ANTHROPIC_API_KEY="YOUR_API_KEY"
```

Verify that it exists without printing the secret:

```bash
test -n "$ANTHROPIC_API_KEY" && echo "API key is set"
```

## 4. Create `app.py`

```python
import os

from anthropic import Anthropic
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="AI Agent")

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


class AskRequest(BaseModel):
    question: str


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post("/ask")
def ask(request: AskRequest):
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=500,
        messages=[
            {
                "role": "user",
                "content": request.question
            }
        ]
    )

    text = "".join(
        block.text
        for block in response.content
        if getattr(block, "type", None) == "text"
    )

    return {"answer": text}
```

> If the model name available to your API account differs, replace it with a model available to your account.

## 5. Run the application

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

In another terminal:

```bash
curl http://localhost:8000/health
```

Test the agent:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Explain Kubernetes in simple words."}'
```

## What you learned

This is an LLM-backed API, but it is not yet a sophisticated agent.

---

# Lab 2 — Add a Tool to the Agent

## Objective

Teach Claude to request an external operation.

We will create a fake order database.

Architecture:

```text
User
 |
 v
Claude
 |
 | decides to use tool
 v
get_order_status()
 |
 v
Fake Order DB
 |
 v
Claude
 |
 v
User
```

## 1. Create `agent_tools.py`

```python
ORDERS = {
    "ORD1001": {
        "status": "Shipped",
        "customer": "Amit",
        "expected_delivery": "2026-10-22"
    },
    "ORD1002": {
        "status": "Processing",
        "customer": "Priya",
        "expected_delivery": "2026-10-25"
    }
}


def get_order_status(order_id: str):
    return ORDERS.get(
        order_id,
        {"status": "Order not found"}
    )
```

## 2. Replace `app.py`

```python
import json
import os

from anthropic import Anthropic
from fastapi import FastAPI
from pydantic import BaseModel

from agent_tools import get_order_status

app = FastAPI(title="Customer Support Agent")

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


tools = [
    {
        "name": "get_order_status",
        "description": "Get the current status of a customer order.",
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "Customer order ID such as ORD1001"
                }
            },
            "required": ["order_id"]
        }
    }
]


class AskRequest(BaseModel):
    question: str


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post("/ask")
def ask(request: AskRequest):

    messages = [
        {
            "role": "user",
            "content": request.question
        }
    ]

    first_response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=500,
        tools=tools,
        messages=messages
    )

    messages.append(
        {
            "role": "assistant",
            "content": first_response.content
        }
    )

    for block in first_response.content:

        if block.type != "tool_use":
            continue

        if block.name == "get_order_status":

            result = get_order_status(
                block.input["order_id"]
            )

            messages.append(
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": json.dumps(result)
                        }
                    ]
                }
            )

    final_response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=500,
        tools=tools,
        messages=messages
    )

    text = "".join(
        block.text
        for block in final_response.content
        if getattr(block, "type", None) == "text"
    )

    return {"answer": text}
```

## 3. Test

Restart Uvicorn if necessary:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

Call:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the status of order ORD1001?"}'
```

Try:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the status of order ORD1002?"}'
```

## Key concept

The important flow is:

```text
User
 ↓
Claude
 ↓
Tool request
 ↓
Your Python code
 ↓
Tool result
 ↓
Claude
 ↓
Final answer
```

Claude does not directly access your database. Your application executes the tool.

---

# Lab 3 — Make the Tool-Calling Loop More Robust

## Objective

Understand why production agents need a loop.

A single request may require multiple tool calls:

```text
User
 |
 v
Claude
 |
 +--> Tool A
 |
 +--> Tool B
 |
 v
Claude
 |
 v
Final answer
```

Create two tools:

```python
def get_customer(customer_id: str):
    return {
        "customer_id": customer_id,
        "name": "Amit",
        "tier": "Gold"
    }


def get_order_status(order_id: str):
    return ORDERS.get(order_id, {"status": "Not found"})
```

The production pattern should continue processing tool calls until Claude produces a normal final response.

Pseudo-flow:

```python
while True:

    response = call_claude(messages)

    if response.stop_reason != "tool_use":
        break

    add_assistant_response()

    for tool_call in response.content:
        execute_tool(tool_call)
        add_tool_result()

return final_answer
```

## Exercise

Modify your application so that it can execute multiple tool calls in one conversation.

Test with:

```text
Tell me the status of ORD1001 and give me the customer information.
```

---

# Lab 4 — Containerize the Agent

## Objective

Package the agent into a Docker image.

Project:

```text
agent/
├── app.py
├── agent_tools.py
├── requirements.txt
└── Dockerfile
```

## 1. Create Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 2. Build

From the `agent` directory:

```bash
docker build -t ai-agent:1.0 .
```

Check:

```bash
docker images | grep ai-agent
```

## 3. Run

```bash
docker run --rm \
  -p 8000:8000 \
  -e ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  ai-agent:1.0
```

Test:

```bash
curl http://localhost:8000/health
```

## Security lesson

Do not put this in the Dockerfile:

```dockerfile
ENV ANTHROPIC_API_KEY="sk-..."
```

Secrets should be injected at runtime.

---

# Lab 5 — Deploy the Agent to Minikube

## Objective

Deploy the container as a Kubernetes Deployment.

## 1. Build the image inside Minikube

Configure your shell:

```bash
eval $(minikube docker-env)
```

Build:

```bash
docker build -t ai-agent:1.0 .
```

Verify:

```bash
docker images | grep ai-agent
```

## 2. Create namespace

```bash
kubectl create namespace ai-agent
```

## 3. Create Kubernetes Secret

```bash
kubectl create secret generic claude-secret \
  -n ai-agent \
  --from-literal=ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY"
```

Check:

```bash
kubectl get secret -n ai-agent
```

## 4. Create `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
  namespace: ai-agent
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
    spec:
      containers:
        - name: agent
          image: ai-agent:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          env:
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: claude-secret
                  key: ANTHROPIC_API_KEY
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get pods -n ai-agent
```

---

# Lab 6 — Expose the Agent with a Kubernetes Service

## Objective

Allow other applications to access the Agent Pods without knowing individual Pod IP addresses.

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-agent
  namespace: ai-agent
spec:
  selector:
    app: ai-agent
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get svc -n ai-agent
```

Access locally:

```bash
kubectl port-forward \
  -n ai-agent \
  svc/ai-agent 8080:80
```

In another terminal:

```bash
curl http://localhost:8080/health
```

Test the agent:

```bash
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the status of ORD1001?"}'
```

Architecture:

```text
Client
  |
  v
Service
  |
  +----> Pod 1
  |
  +----> Pod 2
```

The Service provides stable networking while Kubernetes handles Pod selection.

---

# Lab 7 — Add Health Probes

## Objective

Teach Kubernetes whether the agent is alive and ready to receive traffic.

Update the container section:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl describe pod -n ai-agent
```

## Understand the difference

### Liveness

Question:

```text
"Is this container alive?"
```

If it fails repeatedly, Kubernetes can restart the container.

### Readiness

Question:

```text
"Should this Pod receive traffic?"
```

If it fails, Kubernetes removes the Pod from ready Service endpoints.

---

# Lab 8 — Configure CPU/Memory Requests and Limits

## Objective

Give Kubernetes resource information needed by scheduling and autoscaling.

Add:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Your container section becomes approximately:

```yaml
containers:
  - name: agent
    image: ai-agent:1.0
    ports:
      - containerPort: 8000

    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"

    env:
      - name: ANTHROPIC_API_KEY
        valueFrom:
          secretKeyRef:
            name: claude-secret
            key: ANTHROPIC_API_KEY
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl describe pod -n ai-agent
```

---

# Lab 9 — Horizontal Pod Autoscaler

## Objective

Automatically increase/decrease the number of Agent Pods.

## 1. Verify metrics-server

```bash
kubectl top pods -n ai-agent
```

If it works, continue.

## 2. Create HPA

```bash
kubectl autoscale deployment ai-agent \
  -n ai-agent \
  --cpu-percent=50 \
  --min=2 \
  --max=10
```

Check:

```bash
kubectl get hpa -n ai-agent
```

You should see something similar to:

```text
NAME       TARGETS   MINPODS   MAXPODS   REPLICAS
ai-agent   .../50%   2         10        2
```

## 3. Watch

Terminal 1:

```bash
kubectl get pods -n ai-agent -w
```

Terminal 2:

```bash
kubectl get hpa -n ai-agent -w
```

## Important observation

The HPA is currently scaling based on **CPU**, not Claude token usage.

That is useful for learning Kubernetes autoscaling, but it is not necessarily the ideal signal for an AI workload.

---

# Lab 10 — Generate Load

## Objective

Create enough concurrent traffic to observe scaling.

Install ApacheBench:

```bash
sudo apt-get update
sudo apt-get install -y apache2-utils
```

Port-forward:

```bash
kubectl port-forward \
  -n ai-agent \
  svc/ai-agent 8080:80
```

Health-load test:

```bash
ab -n 1000 -c 20 http://127.0.0.1:8080/health
```

Watch:

```bash
kubectl get hpa -n ai-agent -w
```

and:

```bash
kubectl get pods -n ai-agent -w
```

## Important

Do **not** send thousands of real Claude API requests just to generate CPU load. That can create unnecessary API usage and cost.

For autoscaling experiments, create a `/cpu-test` endpoint that performs local CPU work, or use a separate test worker. Keep real Claude calls limited.

---

# Lab 11 — Understand Why CPU HPA Is Not Enough for AI Agents

Consider:

```text
100 users
    |
    v
Agent API
    |
    v
Claude API
    |
    v
Waiting...
```

Your Pod may use little CPU while waiting for the LLM response.

Therefore:

```text
CPU = 20%
```

does not necessarily mean:

```text
Agent workload = low
```

Useful AI-agent metrics can include:

```text
Requests per second
Active requests
Queue depth
Request latency
LLM latency
Tokens per second
Tool execution latency
Failed requests
Rate-limit errors
```

This leads to the next architecture.

---

# Lab 12 — Queue-Based Agent Architecture

## Objective

Separate request intake from agent execution.

Architecture:

```text
User
 |
 v
Agent API
 |
 v
Queue
 |
 +--------+--------+
 |        |        |
 v        v        v
Worker  Worker  Worker
 Pod     Pod     Pod
 |        |        |
 +--------+--------+
          |
          v
      Claude API
```

A queue allows work to wait safely instead of requiring every incoming request to immediately consume an agent worker.

For the lab, use Redis.

## 1. Start Redis

```bash
kubectl create deployment redis \
  -n ai-agent \
  --image=redis:7
```

Expose it:

```bash
kubectl expose deployment redis \
  -n ai-agent \
  --port=6379
```

Check:

```bash
kubectl get pods -n ai-agent
```

## 2. Worker concept

The worker receives:

```text
{
  "question": "What is order ORD1001?"
}
```

then:

```text
Worker
  |
  v
Claude
  |
  v
Result
```

## Exercise

Create:

```text
api.py
worker.py
```

The API puts requests into Redis.

The worker consumes requests from Redis.

---

# Lab 13 — Scale Workers Based on Queue Depth

## Objective

Understand production-style event-driven scaling.

Instead of:

```text
CPU → Pods
```

we want:

```text
Queue depth → Workers
```

Example:

```text
Queue = 2
 ↓
2 workers

Queue = 50
 ↓
5 workers

Queue = 500
 ↓
10 workers
```

For production Kubernetes environments, tools such as KEDA can scale workloads based on event sources and queue metrics.

Conceptually:

```text
Redis
  |
  | queue length
  v
KEDA
  |
  v
Deployment
  |
  +--> Worker Pod
  +--> Worker Pod
  +--> Worker Pod
```

## Exercise

Install KEDA in Minikube and create a Redis-based `ScaledObject`.

Before applying any KEDA manifest, verify the KEDA version and Redis scaler configuration supported by the version installed in your cluster.

The learning objective is:

```text
Queue depth
    ↓
KEDA
    ↓
Deployment replicas
    ↓
More agent workers
```

---

# Lab 14 — Add Prometheus Metrics

## Objective

Expose metrics from your Agent API.

Install the Python client:

```bash
pip install prometheus-client
```

Add metrics such as:

```text
agent_requests_total
agent_request_duration_seconds
agent_claude_requests_total
agent_claude_errors_total
agent_tool_calls_total
```

Example:

```python
from prometheus_client import Counter, Histogram

REQUESTS = Counter(
    "agent_requests_total",
    "Total agent requests"
)

LATENCY = Histogram(
    "agent_request_duration_seconds",
    "Agent request latency"
)
```

Increment:

```python
REQUESTS.inc()
```

Observe duration around your request processing.

Expose `/metrics`.

Test:

```bash
curl http://localhost:8000/metrics
```

---

# Lab 15 — Grafana Dashboard

## Objective

Visualize:

```text
Requests
Latency
Errors
Pod count
CPU
Memory
```

Install Prometheus/Grafana using a suitable Helm chart or monitoring stack.

Typical architecture:

```text
Agent Pods
    |
    | /metrics
    v
Prometheus
    |
    v
Grafana
```

Create dashboard panels:

### Panel 1

Request rate:

```text
rate(agent_requests_total[5m])
```

### Panel 2

Error rate:

```text
rate(agent_claude_errors_total[5m])
```

### Panel 3

Latency

Use the histogram metrics to calculate p50/p95/p99 latency.

### Panel 4

Pod count

Use Kubernetes metrics.

---

# Lab 16 — Add Timeout and Retry Handling

## Objective

A production agent must handle external API failures.

Possible failures:

```text
Claude API timeout
Claude API 5xx
Network failure
Rate limit
Tool failure
Invalid tool input
```

Do not blindly retry everything.

Conceptual policy:

```text
Request
 |
 v
Claude
 |
 +-- success --> response
 |
 +-- timeout --> limited retry
 |
 +-- 429 -----> backoff/retry
 |
 +-- permanent error --> fail safely
```

Implement:

- request timeout
- limited retries
- exponential backoff
- maximum retry count
- structured error logging

Example backoff:

```text
attempt 1 → 1 second
attempt 2 → 2 seconds
attempt 3 → 4 seconds
```

Never create an infinite retry loop.

---

# Lab 17 — Graceful Shutdown

## Objective

Understand what happens when Kubernetes terminates an Agent Pod.

Typical flow:

```text
kubectl delete pod
        |
        v
SIGTERM
        |
        v
Application stops accepting work
        |
        v
Finish/abandon safely
        |
        v
Container exits
```

Configure Kubernetes:

```yaml
terminationGracePeriodSeconds: 30
```

Your application should handle shutdown so that an in-progress request is not unnecessarily interrupted.

This is especially important for long-running agent tasks.

---

# Lab 18 — Add Agent State

## Objective

Understand why state should generally not live only inside a Pod.

Bad architecture:

```text
Pod 1
 |
 └── conversation memory
```

If Pod 1 disappears:

```text
conversation memory = lost
```

Better:

```text
             ┌── Pod 1
User → Service ├── Pod 2
             └── Pod 3
                  |
                  v
              Redis/DB
                  |
                  v
            Conversation State
```

Store persistent state in an external system.

Possible data:

```text
conversation_id
user_id
messages
tool results
agent status
task status
```

---

# Lab 19 — Multi-Agent Architecture

## Objective

Build multiple specialized agents.

Example:

```text
                  Supervisor Agent
                         |
            +------------+------------+
            |            |            |
            v            v            v
       Research      Support       Billing
        Agent         Agent         Agent
```

The supervisor decides:

```text
Question
   |
   v
Supervisor
   |
   +--> Research Agent
   |
   +--> Support Agent
   |
   +--> Billing Agent
```

Each agent can have its own tools.

Example:

```text
Research Agent
  └── search tool

Support Agent
  └── order-status tool

Billing Agent
  └── invoice tool
```

---

# Lab 20 — Production-Style Final Architecture

At the end of the series, your architecture should look conceptually like:

```text
                           Users
                             |
                             v
                       Load Balancer
                             |
                             v
                         Ingress
                             |
                             v
                    ┌─────────────────┐
                    │ Agent API       │
                    │ Deployment      │
                    └────────┬────────┘
                             |
                       request routing
                             |
                             v
                         Queue
                             |
                ┌────────────┼────────────┐
                v            v            v
             Worker       Worker       Worker
               Pod          Pod          Pod
                |            |            |
                +------------+------------+
                             |
                    Claude / LLM API
                             |
               +-------------+-------------+
               |             |             |
               v             v             v
             Tools         APIs           DB
               |
               v
             MCP servers
```

Observability:

```text
Agent Pods
    |
    +---- Logs ------> Logging
    |
    +---- Metrics ---> Prometheus ---> Grafana
    |
    +---- Traces ----> OpenTelemetry
```

Autoscaling:

```text
                    ┌───────────────┐
                    │ Agent Metrics │
                    └───────┬───────┘
                            |
              +-------------+-------------+
              |                           |
              v                           v
          CPU/Memory                 Queue depth
              |                           |
              v                           v
             HPA                         KEDA
              |                           |
              +-------------+-------------+
                            |
                            v
                       Agent Workers
```

---

# Final Production Checklist

Before calling an AI agent production-ready, check:

## Application

- [ ] Tool schemas are well defined
- [ ] Tool inputs are validated
- [ ] Tool permissions are restricted
- [ ] Agent has bounded loops
- [ ] Timeouts exist
- [ ] Retry policy exists
- [ ] Errors are handled
- [ ] Graceful shutdown exists

## Kubernetes

- [ ] Deployment
- [ ] Service
- [ ] Secret
- [ ] Resource requests
- [ ] Resource limits
- [ ] Readiness probe
- [ ] Liveness probe
- [ ] HPA/KEDA
- [ ] Pod disruption strategy
- [ ] Rolling deployment strategy

## Security

- [ ] API key is not in source code
- [ ] API key is not in Docker image
- [ ] Secrets are managed securely
- [ ] RBAC is least privilege
- [ ] Network access is restricted
- [ ] Tool permissions are restricted

## Observability

- [ ] Request count
- [ ] Error count
- [ ] Latency
- [ ] LLM latency
- [ ] Tool latency
- [ ] Token usage/cost tracking
- [ ] Logs
- [ ] Metrics
- [ ] Traces

## Scaling

- [ ] CPU scaling tested
- [ ] Memory scaling considered
- [ ] Request concurrency measured
- [ ] Queue-based scaling tested
- [ ] Maximum replica count defined
- [ ] LLM provider rate limits considered
- [ ] Cost impact considered

---

# Suggested Learning Sequence

Do not try to build everything on Day 1.

Use this sequence:

```text
Day 1
  Lab 1 → Lab 2 → Lab 3

Day 2
  Lab 4 → Lab 5 → Lab 6

Day 3
  Lab 7 → Lab 8 → Lab 9 → Lab 10

Day 4
  Lab 11 → Lab 12 → Lab 13

Day 5
  Lab 14 → Lab 15 → Lab 16

Day 6
  Lab 17 → Lab 18 → Lab 19

Day 7
  Lab 20 + production review
```

## The most important concept

Do not think of production agent deployment as:

```text
Claude + Kubernetes + HPA
```

Think of it as:

```text
                Agent Platform

     ┌───────────────────────────────────┐
     │                                   │
     │  API / Ingress                    │
     │       ↓                           │
     │  Agent Runtime                    │
     │       ↓                           │
     │  Tools / MCP                      │
     │       ↓                           │
     │  Queue / State                    │
     │       ↓                           │
     │  Agent Workers                    │
     │       ↓                           │
     │  LLM Provider                     │
     │                                   │
     │  Observability                    │
     │  Security                         │
     │  Autoscaling                      │
     │  Reliability                      │
     └───────────────────────────────────┘
```

Kubernetes provides the **platform layer**. Your Python application provides the **agent runtime**. Claude provides the **reasoning/model capability**. External systems provide **tools, state, queues, and data**.

That separation is the key concept to understand before moving from Minikube to a managed production Kubernetes platform.
