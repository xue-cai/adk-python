# ADK Deployment & Hosting — Where to Run Your Agents

You've built an agent with Google ADK — now where do you host it? This document
covers Google's hosting platforms, deployment CLI commands, the official samples
repository, and how the ecosystem compares to alternatives like AWS.

---

## Table of Contents

1. [Ecosystem Comparison: Google vs. AWS](#1-ecosystem-comparison-google-vs-aws)
2. [Google Hosting Platforms](#2-google-hosting-platforms)
3. [ADK Deploy CLI](#3-adk-deploy-cli)
4. [Official Samples Repository](#4-official-samples-repository)
5. [Additional Resources](#5-additional-resources)
6. [Summary](#6-summary)

---

## 1. Ecosystem Comparison: Google vs. AWS

Both Google and AWS offer an agent SDK paired with a managed hosting runtime and
a samples repository. Here is a side-by-side comparison:

| Capability | **Google** | **AWS** |
|---|---|---|
| **Agent SDK** | [Agent Development Kit (ADK)](https://github.com/google/adk-python) | [Strands Agents SDK](https://github.com/strands-agents/sdk-python) |
| **Managed Runtime** | [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) | [Bedrock Agentcore](https://aws.amazon.com/bedrock/agentcore/) |
| **Serverless Hosting** | [Cloud Run](https://cloud.google.com/run) | Lambda / ECS |
| **Container Hosting** | [GKE](https://cloud.google.com/kubernetes-engine) | EKS |
| **Samples Repo** | [google/adk-samples](https://github.com/google/adk-samples) | [amazon-bedrock-agentcore-samples](https://github.com/awslabs/amazon-bedrock-agentcore-samples) |
| **Starter Templates** | [GoogleCloudPlatform/agent-starter-pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) | — |
| **Deploy CLI** | `adk deploy cloud_run / agent_engine / gke` | Console / CDK |
| **Agent-to-Agent Protocol** | [A2A (open standard)](https://github.com/google/A2A) | — |

**Key differentiator**: ADK has a built-in CLI (`adk deploy`) that handles
packaging, containerization, and deployment in a single command. The same
`agent.py` runs locally, on Cloud Run, on Agent Engine, and on GKE with zero
code changes.

---

## 2. Google Hosting Platforms

### 2.1 Vertex AI Agent Engine (Managed, Production-Grade)

**What it is**: Google's fully-managed service purpose-built for agentic
workloads. Think of it as the Google equivalent of AWS Bedrock Agentcore.

**Features**:

- Scalable orchestration with automatic scaling
- Built-in session and memory management
- Agent identity with granular IAM control (SPIFFE-aligned)
- Integrated logging, monitoring, and tracing
- Tool integration with secure Google API access
- Support for MCP tools via custom installation scripts
- Express Mode deployment using just an API key

**When to use**: Production agents that need managed infrastructure, persistent
state, enterprise security, and observability out of the box.

**Docs**:

- Quickstart: https://docs.cloud.google.com/agent-builder/agent-engine/quickstart-adk
- ADK Deploy Guide: https://google.github.io/adk-docs/deploy/agent-engine/

### 2.2 Cloud Run (Serverless, Containerized)

**What it is**: Fully-managed serverless platform for containerized
applications. ADK generates the Dockerfile and deploys with a single CLI command.

**Features**:

- Automatic HTTPS endpoints
- Scale to zero (pay only when handling requests)
- IAM-based access control
- Optional Web UI for interactive testing (`--with_ui`)
- CORS configuration with regex support
- Cloud Trace and OpenTelemetry integration
- A2A endpoint support (`--a2a`)

**When to use**: Agents that need serverless hosting with minimal configuration,
cost-efficient scaling, and fast iteration.

**Docs**:

- Official Guide: https://docs.cloud.google.com/run/docs/ai/build-and-deploy-ai-agents/deploy-adk-agent

### 2.3 Google Kubernetes Engine (GKE)

**What it is**: Managed Kubernetes for organizations that need full control over
orchestration, networking, and scaling policies.

**Features**:

- Full Kubernetes deployment with auto-generated manifests
- LoadBalancer service for external access
- Cloud Build integration for container image building
- Standard Kubernetes labels for fleet management
- Compatible with existing GKE workloads and policies

**When to use**: When you already run workloads on Kubernetes and need your
agents to fit into existing clusters, RBAC policies, and networking setups.

---

## 3. ADK Deploy CLI

The ADK CLI provides three deployment targets. All three share the same local
development → cloud deployment workflow: you build and test locally with
`adk web` or `adk run`, then deploy with `adk deploy <target>`.

The deployment logic lives in `src/google/adk/cli/cli_deploy.py` and the CLI
definitions are in `src/google/adk/cli/cli_tools_click.py`.

### 3.1 Deploy to Cloud Run

```bash
adk deploy cloud_run [OPTIONS] AGENT
```

**Key options**:

| Option | Description | Default |
|---|---|---|
| `--project` | GCP project ID | gcloud default |
| `--region` | GCP region | gcloud default |
| `--service_name` | Cloud Run service name | `adk-default-service-name` |
| `--port` | API server port | `8000` |
| `--with_ui` | Deploy with ADK Web UI | API server only |
| `--a2a` | Enable A2A endpoint | disabled |
| `--trace_to_cloud` | Export traces to Cloud Trace | disabled |
| `--otel_to_cloud` | Export telemetry to GCP | disabled |
| `--session_service_uri` | Custom session service URI | in-memory |
| `--artifact_service_uri` | Custom artifact service URI | in-memory |
| `--memory_service_uri` | Custom memory service URI | — |
| `--allow_origins` | CORS origins (supports `regex:` prefix) | — |
| `--adk_version` | ADK version in container | current version |

**What happens under the hood**:

1. Copies agent source to a temp folder
2. Generates a Dockerfile (Python 3.11-slim, non-root user, installs ADK)
3. Runs `gcloud run deploy` to build, push, and deploy the container
4. Labels the service with `created-by=adk`

**Example**:

```bash
adk deploy cloud_run \
  --project=my-project \
  --region=us-central1 \
  --with_ui \
  path/to/my_agent
```

### 3.2 Deploy to Agent Engine

```bash
adk deploy agent_engine [OPTIONS] AGENT
```

**Key options**:

| Option | Description | Default |
|---|---|---|
| `--project` | GCP project ID | `.env` GOOGLE_CLOUD_PROJECT |
| `--region` | GCP region | `.env` GOOGLE_CLOUD_LOCATION |
| `--api_key` | API key for Express Mode | — |
| `--agent_engine_id` | Existing engine ID to update | create new |
| `--display_name` | Display name in Agent Engine | — |
| `--description` | Description in Agent Engine | — |
| `--trace_to_cloud` | Export traces to Cloud Trace | disabled |
| `--adk_app` | Python file for AdkApp wrapper | `agent_engine_app.py` |
| `--adk_app_object` | Root object: `root_agent` or `app` | `root_agent` |
| `--env_file` | Path to `.env` file | — |
| `--requirements_file` | Custom requirements.txt | auto-detected |
| `--agent_engine_config_file` | Configuration JSON file | auto-detected |

**Two authentication modes**:

- **Project/Region mode**: Uses GCP credentials (standard for organizations)
- **Express Mode**: Uses an API key directly (quick prototyping)

**What happens under the hood**:

1. Stages agent files in a temp folder (respects `.ae_ignore`)
2. Resolves dependencies and merges environment variables
3. Generates an `agent_engine_app.py` wrapper that creates an `AdkApp`
4. Deploys to Vertex AI Agent Engine (creates new or updates existing)

**Example — Express Mode**:

```bash
adk deploy agent_engine --api_key=YOUR_KEY my_agent
```

**Example — Project Mode**:

```bash
adk deploy agent_engine \
  --project=my-project \
  --region=us-central1 \
  my_agent
```

### 3.3 Deploy to GKE

```bash
adk deploy gke [OPTIONS] AGENT
```

**Key options**:

| Option | Description | Default |
|---|---|---|
| `--project` | GCP project ID | gcloud default |
| `--region` | GCP region | gcloud default |
| `--cluster_name` | GKE cluster name | **required** |
| `--service_name` | Kubernetes service name | `adk-default-service-name` |
| `--port` | API server port | `8000` |
| `--with_ui` | Deploy with ADK Web UI | API server only |
| `--trace_to_cloud` | Export traces to Cloud Trace | disabled |
| `--session_service_uri` | Custom session service URI | in-memory |
| `--artifact_service_uri` | Custom artifact service URI | in-memory |
| `--adk_version` | ADK version in container | current version |

**What happens under the hood**:

1. Copies agent source and resolves dependencies
2. Generates a Dockerfile and a `deployment.yaml` Kubernetes manifest
3. Builds the container image with Cloud Build, pushes to GCR
4. Gets cluster credentials and runs `kubectl apply`

**Example**:

```bash
adk deploy gke \
  --project=my-project \
  --region=us-central1 \
  --cluster_name=my-cluster \
  path/to/my_agent
```

---

## 4. Official Samples Repository

### google/adk-samples

📦 **https://github.com/google/adk-samples**

This is the primary samples repository — the Google equivalent of
[amazon-bedrock-agentcore-samples](https://github.com/awslabs/amazon-bedrock-agentcore-samples).

**Contents**:

- Sample agents in **Python, TypeScript, Go, and Java**
- Domains: customer service, data engineering, financial advising, travel,
  and more
- Each agent includes a README, code, and customization instructions
- Deployment examples for Cloud Run and Agent Engine

### GoogleCloudPlatform/agent-starter-pack

📦 **https://github.com/GoogleCloudPlatform/agent-starter-pack**

Production-ready templates with CI/CD, observability, and security built in.
Supports one-click deployment to Agent Engine and Cloud Run.

### Agent Garden

🌱 **https://developers.googleblog.com/en/agent-garden-samples-for-learning-discovering-and-building/**

Google's curated catalog of sample agents. Includes both simple bots and
sophisticated multi-agent systems. Supports one-click deployment via Agent
Starter Pack and customization in Firebase Studio.

---

## 5. Additional Resources

| Resource | Link |
|---|---|
| **ADK Deployment Docs** | https://google.github.io/adk-docs/deploy/ |
| **Agent Engine Quickstart** | https://docs.cloud.google.com/agent-builder/agent-engine/quickstart-adk |
| **Cloud Run Deploy Guide** | https://docs.cloud.google.com/run/docs/ai/build-and-deploy-ai-agents/deploy-adk-agent |
| **Multi-Agent Deployment Lab** | https://www.skills.google/course_templates/1275 |
| **ADK Python Source** | https://github.com/google/adk-python |
| **ADK Samples** | https://github.com/google/adk-samples |
| **Agent Starter Pack** | https://github.com/GoogleCloudPlatform/agent-starter-pack |
| **A2A Protocol** | https://github.com/google/A2A |

---

## 6. Summary

**Q: Once I wrote an agent with Google ADK, does Google offer a hosting platform
to run the agent?**

**A: Yes — three of them**, each serving different needs:

| Platform | Best For | Deploy Command |
|---|---|---|
| **Vertex AI Agent Engine** | Managed production workloads with built-in state, security, and monitoring | `adk deploy agent_engine` |
| **Cloud Run** | Serverless, cost-efficient hosting with zero infrastructure management | `adk deploy cloud_run` |
| **GKE** | Organizations with existing Kubernetes infrastructure | `adk deploy gke` |

The workflow is:

1. **Build locally**: `adk web` or `adk run` for development and testing
2. **Deploy to cloud**: `adk deploy <target>` — one command handles packaging,
   containerization, and deployment
3. **Iterate**: Same agent code runs everywhere with zero changes

The [google/adk-samples](https://github.com/google/adk-samples) repo provides
ready-to-use examples, and the
[agent-starter-pack](https://github.com/GoogleCloudPlatform/agent-starter-pack)
provides production CI/CD templates for rapid deployment.
