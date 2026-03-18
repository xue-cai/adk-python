# ADK Deployment & Hosting — Where to Run Your Agents

You've built an agent with Google ADK — now where do you host it? This document
covers Google's hosting platforms, deployment CLI commands, the official samples
repository, and how the ecosystem compares to alternatives from AWS, Azure,
Databricks, OpenAI, and Anthropic.

---

## Table of Contents

1. [Ecosystem Comparison: All Major Providers](#1-ecosystem-comparison-all-major-providers)
2. [Google Hosting Platforms](#2-google-hosting-platforms)
3. [ADK Deploy CLI](#3-adk-deploy-cli)
4. [Other Providers In Depth](#4-other-providers-in-depth)
5. [Official Samples Repository](#5-official-samples-repository)
6. [Additional Resources](#6-additional-resources)
7. [Summary](#7-summary)

---

## 1. Ecosystem Comparison: All Major Providers

Every major cloud and AI provider now offers an agent SDK paired with some form
of managed hosting. Here is a side-by-side comparison:

| Capability | **Google** | **AWS** | **Azure / Microsoft** | **Databricks** | **OpenAI** | **Anthropic** |
|---|---|---|---|---|---|---|
| **Agent SDK** | [ADK](https://github.com/google/adk-python) | [Strands Agents](https://github.com/strands-agents/sdk-python) | [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) | [Mosaic AI Agent Framework](https://www.databricks.com/product/machine-learning/retrieval-augmented-generation) | [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python) |
| **Managed Runtime** | [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) | [Bedrock Agentcore](https://aws.amazon.com/bedrock/agentcore/) | [Foundry Agent Service](https://azure.microsoft.com/en-us/products/ai-foundry/agent-service/) | [Model Serving](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-framework-notebook) | [OpenAI Platform](https://platform.openai.com/) | Partner hosting (Cloudflare, Modal, etc.) |
| **Serverless Hosting** | [Cloud Run](https://cloud.google.com/run) | Lambda / ECS | [Azure Container Apps](https://azure.microsoft.com/en-us/products/container-apps/) | Serverless endpoints | — | — |
| **Container Hosting** | [GKE](https://cloud.google.com/kubernetes-engine) | EKS | [AKS](https://azure.microsoft.com/en-us/products/kubernetes-service/) | — | Self-host (any cloud) | Self-host (any cloud) |
| **Samples Repo** | [google/adk-samples](https://github.com/google/adk-samples) | [agentcore-samples](https://github.com/awslabs/amazon-bedrock-agentcore-samples) | [Agent-Framework-Samples](https://github.com/microsoft/Agent-Framework-Samples) | [Tutorials](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-framework-notebook) | [Examples in SDK repo](https://github.com/openai/openai-agents-python/tree/main/examples) | [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) |
| **Deploy CLI** | `adk deploy` | Console / CDK | `az` CLI + ARM/Bicep | Databricks UI / REST | Self-host | Self-host |
| **A2A Protocol** | [A2A (open standard)](https://github.com/google/A2A) | — | A2A support | — | — | — |
| **MCP Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **SDK Languages** | Python, Java, Go, TS | Python | Python, .NET | Python | Python, TS | Python, TS |

**Key observations**:

- **Google** is the only provider with a built-in `deploy` CLI that packages,
  containerizes, and deploys in one command. Same `agent.py` runs everywhere.
- **Azure** has the deepest enterprise integration (Entra ID, Microsoft 365,
  1,400+ Logic Apps connectors).
- **Databricks** is unique in providing data-native agent hosting with Unity
  Catalog governance and MLflow-based evaluation built in.
- **OpenAI** and **Anthropic** provide SDKs but rely on self-hosting or
  third-party cloud providers for deployment infrastructure.

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

## 4. Other Providers In Depth

### 4.1 Azure / Microsoft

**SDK**: [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)
(MIT license, Python + .NET)

The Microsoft Agent Framework is the convergence of AutoGen (multi-agent
research) and Semantic Kernel (enterprise workflows) into a single, unified
open-source SDK. It launched in public preview in October 2025.

**Managed Runtime**: [Azure AI Foundry Agent Service](https://azure.microsoft.com/en-us/products/ai-foundry/agent-service/)

A fully-managed cloud platform for building, scaling, and deploying AI agents
with enterprise-grade security (Entra ID for agent identity, RBAC, audit
logging).

**Key features**:

- Agent and workflow orchestration (autonomous + deterministic patterns)
- Entra ID-based agent identity with granular RBAC
- OpenTelemetry + Azure Monitor for observability
- 1,400+ Azure Logic Apps connectors for enterprise integration
- Interoperates with Microsoft 365 Copilot, Teams, Power Platform
- Supports A2A and MCP open standards
- Declarative agent/workflow definitions via YAML/JSON

**Samples**: [microsoft/Agent-Framework-Samples](https://github.com/microsoft/Agent-Framework-Samples)

**Docs**: [Microsoft Learn — Agent Framework Overview](https://learn.microsoft.com/en-us/agent-framework/overview/)

---

### 4.2 Databricks

**SDK**: [Mosaic AI Agent Framework](https://www.databricks.com/product/machine-learning/retrieval-augmented-generation)
(integrated into the Databricks platform)

Databricks provides an agent development and hosting platform that is uniquely
data-native — agents are built, evaluated, and deployed alongside your data
lakehouse with Unity Catalog governance.

**Managed Runtime**: Databricks Model Serving endpoints with auto-scaling

**Key features**:

- Build agents with LangGraph, LangChain, or custom Python
- Unity Catalog for data governance, tool registration, and access control
- MLflow integration for experiment tracking, evaluation, and versioning
- Built-in Agent Evaluation framework (LLM-as-judge metrics, hallucination
  detection, retrieval accuracy)
- Deploy as auto-scaling REST endpoints via Model Serving
- AI Playground for visual prototyping before code export
- MCP support for tool integration

**Deploy workflow**:

1. Build agent in Databricks notebook or IDE
2. Package as MLflow model (`mlflow.pyfunc.ChatAgent`)
3. Register in Unity Catalog
4. Deploy via Model Serving endpoint
5. Monitor with built-in observability + Arize integration

**Tutorials**: [Build, evaluate, and deploy a retrieval agent](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-framework-notebook)

**Docs**: [Mosaic AI Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/index.html)

---

### 4.3 OpenAI

**SDK**: [OpenAI Agents SDK](https://github.com/openai/openai-agents-python)
(Python + TypeScript, open source)

A lightweight framework for building agentic applications with modular agents,
multi-agent handoffs, guardrails, tool calling, and built-in tracing.

**Managed Runtime**: OpenAI Platform (API-hosted) + self-host anywhere

Unlike cloud providers that offer managed container hosting, OpenAI's approach
is API-centric: your agents call the OpenAI API for inference, but you host the
orchestration logic yourself.

**Key features**:

- Multi-agent orchestration with handoffs and routing
- Guardrails for input/output validation (Pydantic integration)
- Built-in tracing and debugging utilities
- Real-time agent support (voice/chat with streaming)
- Session/memory management (SQLite, Redis, SQLAlchemy adapters)
- Cloud-agnostic: self-host on any platform (AWS, GCP, Azure, Railway, etc.)
- Hosted tools (file search, code interpreter, container shell)

**Hosting options**:

- **OpenAI Platform**: API-backed, managed inference
- **Railway**: One-click deploy templates with FastAPI + Gradio
- **Self-host**: Docker on any cloud provider

**Samples**: [examples/ in SDK repo](https://github.com/openai/openai-agents-python/tree/main/examples)
— includes customer service, financial research, voice agents, memory patterns

**Docs**: [OpenAI Agents SDK Guide](https://developers.openai.com/api/docs/guides/agents-sdk)

---

### 4.4 Anthropic

**SDK**: [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python)
(Python + [TypeScript](https://github.com/anthropics/claude-agent-sdk-typescript))

The Claude Agent SDK (formerly Claude Code SDK) provides the same agentic
capabilities that power Claude Code: file operations, shell commands, code
editing, web search, and multi-step workflows.

**Managed Runtime**: No first-party managed PaaS — uses partner hosting

Anthropic does not operate its own agent hosting platform. Instead, the SDK is
designed for deployment in third-party managed sandboxes.

**Key features**:

- Built-in tools: file ops, shell commands, web search, code editing
- Hooks, subagents, and orchestration patterns for complex workflows
- Secure sandboxing (per-session ephemeral or persistent containers)
- MCP support for custom tool integration
- Streaming (stateless `query()`) and interactive (multi-turn sessions)

**Hosting options** (partner ecosystem):

| Provider | Type | Use Case |
|---|---|---|
| Cloudflare Sandboxes | Managed | Edge-deployed agents |
| Modal | Managed | Serverless GPU/CPU containers |
| Daytona | Managed | Dev environment agents |
| E2B | Managed | Code execution sandboxes |
| Fly Machines | Managed | Global edge containers |
| Docker / Firecracker | Self-host | Full control |

**Docs**:

- Overview: https://platform.claude.com/docs/en/agent-sdk/overview
- Hosting guide: https://platform.claude.com/docs/en/agent-sdk/hosting

---

## 5. Official Samples Repository

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

## 6. Additional Resources

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
| **Microsoft Agent Framework** | https://github.com/microsoft/agent-framework |
| **Microsoft Agent Framework Samples** | https://github.com/microsoft/Agent-Framework-Samples |
| **Azure Foundry Agent Service** | https://azure.microsoft.com/en-us/products/ai-foundry/agent-service/ |
| **Databricks Agent Framework** | https://www.databricks.com/product/machine-learning/retrieval-augmented-generation |
| **OpenAI Agents SDK** | https://github.com/openai/openai-agents-python |
| **OpenAI Agents SDK Docs** | https://developers.openai.com/api/docs/guides/agents-sdk |
| **Claude Agent SDK (Python)** | https://github.com/anthropics/claude-agent-sdk-python |
| **Claude Agent SDK (TypeScript)** | https://github.com/anthropics/claude-agent-sdk-typescript |
| **Claude Agent SDK Hosting Guide** | https://platform.claude.com/docs/en/agent-sdk/hosting |

---

## 7. Summary

**Q: Once I wrote an agent with Google ADK, does Google offer a hosting platform
to run the agent?**

**A: Yes — three of them**, each serving different needs:

| Platform | Best For | Deploy Command |
|---|---|---|
| **Vertex AI Agent Engine** | Managed production workloads with built-in state, security, and monitoring | `adk deploy agent_engine` |
| **Cloud Run** | Serverless, cost-efficient hosting with zero infrastructure management | `adk deploy cloud_run` |
| **GKE** | Organizations with existing Kubernetes infrastructure | `adk deploy gke` |

**Q: How does this compare to other providers?**

| Provider | SDK | Hosting Model | Maturity |
|---|---|---|---|
| **Google** | ADK | Managed (Agent Engine, Cloud Run, GKE) + one-command deploy CLI | Production-ready, 3 deploy targets |
| **AWS** | Strands Agents | Managed (Bedrock Agentcore) + Lambda/ECS | Production-ready |
| **Azure** | Microsoft Agent Framework | Managed (Foundry Agent Service) + AKS/Container Apps | Public preview (Oct 2025) |
| **Databricks** | Mosaic AI Agent Framework | Platform-native (Model Serving + Unity Catalog) | Production-ready for data-centric orgs |
| **OpenAI** | OpenAI Agents SDK | API-hosted inference + self-host orchestration | SDK mature, no managed container hosting |
| **Anthropic** | Claude Agent SDK | Partner-hosted sandboxes (Cloudflare, Modal, etc.) | SDK available, no first-party hosting |

The Google ADK workflow is:

1. **Build locally**: `adk web` or `adk run` for development and testing
2. **Deploy to cloud**: `adk deploy <target>` — one command handles packaging,
   containerization, and deployment
3. **Iterate**: Same agent code runs everywhere with zero changes

The [google/adk-samples](https://github.com/google/adk-samples) repo provides
ready-to-use examples, and the
[agent-starter-pack](https://github.com/GoogleCloudPlatform/agent-starter-pack)
provides production CI/CD templates for rapid deployment.
