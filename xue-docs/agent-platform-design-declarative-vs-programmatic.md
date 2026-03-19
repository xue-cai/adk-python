# Agent Platform Design — Declarative vs Programmatic Agents

How should a cloud platform (Azure, AWS, GCP) design its agent hosting
infrastructure to support both declarative and programmatic agent-building
approaches? This document analyzes the two paradigms, proposes a naming
convention, and addresses six critical design dimensions.

---

## Table of Contents

1. [Two Approaches to Building Agents](#1-two-approaches-to-building-agents)
2. [Naming: Declarative vs Programmatic](#2-naming-declarative-vs-programmatic)
3. [How to Define an Agent — Is IaC Involved?](#3-how-to-define-an-agent--is-iac-involved)
4. [How to Deploy and Update an Agent](#4-how-to-deploy-and-update-an-agent)
5. [Agent Identity and Permission Management](#5-agent-identity-and-permission-management)
6. [Where Does the Agent Run? VPC/VNet Integration](#6-where-does-the-agent-run-vpcvnet-integration)
7. [How Does the Agent Call Tools? Customer Resources](#7-how-does-the-agent-call-tools-customer-resources)
8. [Where Is Memory Stored? Customer-Controlled Storage](#8-where-is-memory-stored-customer-controlled-storage)
9. [Unified Platform Architecture](#9-unified-platform-architecture)
10. [Summary](#10-summary)

---

## 1. Two Approaches to Building Agents

There are two fundamental approaches to defining and building agents:

### Approach 1 — Declarative Agent

The developer specifies:

- The LLM to use (model selection)
- A system prompt (behavioral instructions)
- A list of tools the agent can call
- Memory and session strategies

The **platform provides** the agent loop — the reason-act orchestration that
processes user prompts, calls the LLM, invokes tools, and manages conversation
state. The developer does not write loop logic.

**Example** (ADK):

```python
root_agent = Agent(
    name="travel_assistant",
    model="gemini-2.5-flash",
    instruction="You are a helpful travel assistant...",
    tools=[google_search, book_flight, check_weather],
)
```

### Approach 2 — Programmatic Agent

The developer writes the agent's code — the loop, the workflow, the
orchestration logic. The developer may or may not use an agent framework SDK.
The code handles:

- Custom agent loops or predefined workflows
- State management and error handling
- Tool invocation logic
- Packaging with dependencies for deployment

**Example** (custom code):

```python
async def run_agent(user_message: str, session: Session):
    # Custom orchestration logic
    plan = await planner_llm.generate(user_message)
    for step in plan.steps:
        tool = resolve_tool(step.tool_name)
        result = await tool.execute(step.args)
        if result.needs_human_review:
            await pause_for_review(result)
    return synthesize_response(results)
```

### Should Both Run on the Same Platform?

**Yes.** Approach 1 (declarative) is a specialization of Approach 2
(programmatic), not a separate paradigm. The industry is converging on this —
every major provider (Google, AWS, Azure, Databricks) offers both a
declarative/config-driven path and a code-first path that deploy to the same
underlying infrastructure.

---

## 2. Naming: What Do We Call These Two Types of Agents?

The two approaches need clear, distinct names for platform documentation, IaC
resource types, CLI commands, and API fields. Here is a comprehensive analysis
of naming options.

### Top Recommendations

| Rank | Approach 1 Name | Approach 2 Name | Best Context |
|---|---|---|---|
| 🥇 | **Declarative Agent** | **Code-first Agent** | Platform docs, marketing — combines precision with developer appeal |
| 🥈 | **Declarative Agent** | **Programmatic Agent** | API/IaC resource types — formal and unambiguous |
| 🥉 | **Configured Agent** | **Coded Agent** | Casual/internal — simple and intuitive |

### Full Naming Landscape

Below is every reasonable naming option, analyzed across five criteria:

| # | Approach 1 | Approach 2 | Precision | Simplicity | Value-neutral | Industry precedent | Best for |
|---|---|---|---|---|---|---|---|
| 1 | **Declarative** | **Code-first** | ★★★★ | ★★★★ | ★★★★★ | K8s, Terraform, ADK | Developer-facing docs, marketing |
| 2 | **Declarative** | **Programmatic** | ★★★★★ | ★★★ | ★★★★★ | K8s, Azure | API types, IaC resources, specs |
| 3 | **Declarative** | **Imperative** | ★★★★★ | ★★★ | ★★★★★ | K8s, Terraform | Platform architecture docs |
| 4 | **Config-driven** | **Code-driven** | ★★★★ | ★★★★★ | ★★★★★ | General industry | Internal shorthand, CLI flags |
| 5 | **Configured** | **Coded** | ★★★ | ★★★★★ | ★★★★★ | — | Casual conversation, onboarding |
| 6 | **Specification-based** | **Implementation-based** | ★★★★★ | ★★ | ★★★★★ | CS theory | Academic / architecture docs |
| 7 | **Schema-defined** | **Developer-defined** | ★★★★ | ★★★ | ★★★★★ | JSON Schema, OpenAPI | API/schema-centric platforms |
| 8 | **Platform-orchestrated** | **Self-orchestrated** | ★★★★ | ★★★ | ★★★★★ | Cloud-native | Platform provider perspective |
| 9 | **Managed-loop** | **Custom-loop** | ★★★★★ | ★★★ | ★★★★★ | — | Technical deep-dives |
| 10 | **Hosted-logic** | **Bring-your-own-logic** (BYOL) | ★★★ | ★★★ | ★★★★★ | BYOK, BYOD patterns | Cloud enterprise sales |
| 11 | **Blueprint** | **Bespoke** | ★★★ | ★★★★ | ★★★★ | Architecture metaphor | Marketing, non-technical audience |
| 12 | **Turnkey** | **Custom-built** | ★★★ | ★★★★ | ★★★★ | Business/SaaS | Executive / business audience |
| 13 | **Template** | **Freeform** | ★★★ | ★★★★ | ★★★★ | — | Visual builder UIs |
| 14 | **Low-code** | **Pro-code** | ★★★ | ★★★★ | ★★ | Microsoft Power Platform | ⚠️ Not recommended — see below |
| 15 | **No-code** | **Full-code** | ★★ | ★★★★★ | ★ | Webflow, Bubble | ⚠️ Not recommended — see below |
| 16 | **Composed** | **Crafted** | ★★★ | ★★★★ | ★★★★ | — | Creative / marketing copy |
| 17 | **Intent-driven** | **Logic-driven** | ★★★ | ★★★ | ★★★★★ | — | Product vision documents |
| 18 | **Assembled** | **Engineered** | ★★★ | ★★★★ | ★★★ | Manufacturing metaphor | ⚠️ Implies quality gap |

### Detailed Analysis of Top Contenders

#### 🥇 Declarative vs Code-first

```
type: "declarative"   →  Developer declares WHAT the agent should be
type: "code-first"    →  Developer writes code for HOW the agent behaves
```

**Why this wins for developer-facing docs:**

- "Declarative" is universally understood from K8s, Terraform, and IaC
- "Code-first" is the exact term ADK uses in its own philosophy — "ADK was
  designed to make agent development feel more like software development"
- It carries a positive connotation for developers without diminishing the
  declarative approach
- Sounds natural: *"Is this a declarative agent or a code-first agent?"*

#### 🥈 Declarative vs Programmatic

```
type: "declarative"    →  Specification interpreted by platform
type: "programmatic"   →  Code executed by platform
```

**Why this wins for API/IaC naming:**

- Both words are formal, parallel in construction, and value-neutral
- Works well as enum values: `agent_type: declarative | programmatic`
- Precise: "programmatic" means "done through programming" — exactly right
- No ambiguity about what each term means

#### 🥉 Config-driven vs Code-driven

```
type: "config-driven"   →  Behavior driven by configuration
type: "code-driven"     →  Behavior driven by code
```

**Why this is good for CLI/flags:**

- Parallel structure makes it scannable: `--type=config-driven`
- Self-explanatory without needing documentation
- Con: "config" undersells the declarative approach (prompt engineering and
  tool orchestration strategy are more than "config")

#### ⚠️ Why NOT Low-code / No-code vs Pro-code

- "Low-code" implies the declarative approach is for less skilled developers.
  It's not — designing a multi-agent system with sophisticated tool selection,
  memory strategies, and prompt engineering is high-skill work.
- "No-code" is factually wrong — declarative agents in ADK are defined in
  Python.
- These terms create a value hierarchy that alienates one audience or the
  other.

#### ⚠️ Why NOT Managed vs Custom

- This conflates the agent **definition** (what it is) with the
  **deployment** (where it runs). A declarative agent could run self-hosted
  on GKE. A programmatic agent could run on fully-managed Cloud Run.
- "Managed" should describe infrastructure, not agent type.

### Context-Specific Recommendations

| Context | Recommended Terms |
|---|---|
| **IaC / API resource types** | `declarative` / `programmatic` |
| **Developer docs / tutorials** | Declarative Agent / Code-first Agent |
| **CLI flags and config files** | `--agent-type=declarative` / `--agent-type=code-first` |
| **Platform architecture docs** | Declarative / Programmatic (or Imperative) |
| **Marketing / landing pages** | Declarative / Code-first |
| **Internal shorthand** | Config-driven / Code-driven |
| **Executive / business audience** | Turnkey / Custom-built |
| **Enterprise sales** | Hosted-logic / Bring-your-own-logic (BYOL) |

### Industry Precedent

| Platform / Tool | Their Terminology | Notes |
|---|---|---|
| **Kubernetes** | Declarative vs Imperative | `kubectl apply` (declarative) vs `kubectl create` (imperative) |
| **Terraform / Pulumi** | Declarative vs Procedural | Terraform = declarative HCL; Pulumi = programmatic code |
| **Azure Foundry Agent Service** | Declarative agent definitions | YAML/JSON agent specs with Entra ID identity |
| **Google ADK** | "Code-first" philosophy | ADK docs: "designed to make agent development feel like software development" |
| **AWS CDK** | L1 (declarative) vs L3 (opinionated constructs) | Different abstraction levels for the same resources |
| **Microsoft Power Platform** | Low-code vs Pro-code | ⚠️ Creates a skill hierarchy — not recommended for agent naming |
| **Databricks** | Notebook (visual) vs SDK (programmatic) | Similar split: visual config vs code |

---

## 3. How to Define an Agent — Is IaC Involved?

### Design: Two Definition Tiers, Unified Deployment Artifact

| | Declarative Agent | Programmatic Agent |
|---|---|---|
| **Definition format** | Structured config (YAML/JSON/Python dataclass) specifying model, prompt, tools, memory | Arbitrary code: `agent.py` with custom logic, workflows, dependencies |
| **What the platform sees** | A config object that the platform's built-in runtime interprets | A container image or code package the platform executes |
| **IaC story** | The agent config **is** the IaC — a declarative resource like a Terraform `platform_agent` resource | IaC manages the infrastructure (Cloud Run service, K8s deployment, IAM bindings), but the agent logic is opaque code inside |

### Platform Recommendation

- **Define a unified Agent Resource** in the control plane. This resource has
  metadata (name, version, identity, permissions) regardless of whether it's
  declarative or programmatic.
- For **Declarative Agents**: The agent definition *is* the resource spec. Think
  of it like a CRD (Custom Resource Definition) in Kubernetes. The platform
  provides the controller (runtime) that interprets it.
- For **Programmatic Agents**: The agent definition is a pointer to a deployable
  artifact (container image, code package). The platform treats it as a
  workload, similar to how Cloud Run or Lambda treats a container.
- **IaC should wrap both**: Whether you use Terraform, Bicep, or CDK, the IaC
  resource should be `platform_agent` with a `type` field (`declarative` vs
  `programmatic`). For declarative agents, the config is inline. For
  programmatic agents, the config points to a build artifact.

### How ADK Does This Today

ADK's `adk deploy` CLI is an early form of this unification:

- `adk deploy agent_engine` deploys a declarative agent to a managed runtime
- `adk deploy cloud_run` and `adk deploy gke` deploy the same code as a
  container

The agent definition (`agent.py`) is identical in both cases, but the
deployment target changes. What's missing is a formal IaC resource model (no
Terraform provider for "an ADK agent" as a first-class resource, unlike Azure
which has ARM/Bicep templates for Foundry Agent Service).

---

## 4. How to Deploy and Update an Agent

### Design: Immutable Versioned Deployments with Traffic Splitting

| Concern | Declarative Agent | Programmatic Agent |
|---|---|---|
| **Artifact** | Config version (stored in control plane DB) | Container image tag / code package version |
| **Update mechanism** | Config patch → platform re-evaluates (no rebuild needed) | New image build → rolling deployment |
| **Rollback** | Revert to previous config version | Revert to previous image tag |
| **Canary/gradual rollout** | Traffic split between config versions | Traffic split between container revisions |

### Platform Recommendation

- Treat every agent deployment as an **immutable revision**. This is what Cloud
  Run already does (revision-based deployments with traffic splitting).
- For **Declarative Agents**, a config change creates a new revision instantly
  (no build step). This is a huge advantage — you can A/B test different system
  prompts, tool sets, or models without any CI/CD pipeline.
- For **Programmatic Agents**, a code change requires a build step (container
  build, dependency resolution), then creates a new revision.
- Support **gradual rollout** for both: route 10% of traffic to the new
  revision, monitor, then promote.
- **CI/CD integration**: A production platform needs GitOps support — push to a
  branch, trigger a pipeline that runs evaluation (`adk eval`), then deploy if
  metrics pass.

### How ADK Does This Today

Google has the primitives (Cloud Run revisions, Agent Engine updates via
`--agent_engine_id`), but the orchestration layer (GitOps, gradual rollout,
evaluation gates) is left to the developer via the
[agent-starter-pack](https://github.com/GoogleCloudPlatform/agent-starter-pack)
templates.

---

## 5. Agent Identity and Permission Management

**This is the hardest problem and the biggest gap in current platforms.**

### Design: Every Agent Gets a First-Class Identity

| Concern | Recommendation |
|---|---|
| **Agent identity** | Each agent gets a platform-managed identity (like a K8s ServiceAccount or Azure Managed Identity). Not a shared service account. |
| **Identity type** | SPIFFE-compatible workload identity (Vertex AI Agent Engine is "SPIFFE-aligned") |
| **Permission model** | RBAC with least-privilege: the agent identity has only the permissions it needs to call its declared tools |
| **User impersonation** | For tools that act on behalf of a user, the platform supports delegation (OAuth2 on-behalf-of flow, credential forwarding) |
| **Agent-to-agent auth** | When Agent A calls Agent B (via A2A), the platform handles mutual TLS or token exchange automatically |

### Platform Recommendation

- **Declarative Agents**: The platform knows exactly what tools the agent will
  call (they're declared in the config). The platform can **auto-generate** the
  minimal IAM policy. This is a major advantage — the platform can enforce
  least-privilege automatically.
- **Programmatic Agents**: The platform doesn't know what the code will do. The
  developer must declare required permissions (like a K8s `Role` or AWS IAM
  policy). The platform should provide a **permission manifest** that ships
  alongside the code.
- **Both approaches** should use **workload identity federation** to avoid
  long-lived credentials. The agent gets a short-lived token from the
  platform's identity provider, which it uses to authenticate to tools and
  services.

### How ADK Does This Today

ADK's auth system is sophisticated (OAuth2, service accounts, OIDC, credential
lifecycle management), but it's **tool-level auth** — it handles "how does a
tool authenticate to an external API" but not "who is this agent, and what is
it allowed to do?" That question is left to the hosting platform:

- Vertex AI Agent Engine: IAM control (SPIFFE-aligned identity)
- Cloud Run: service account attached to the Cloud Run service
- GKE: Kubernetes workload identity

**Azure's approach is the most mature here**: Entra ID gives each agent a
first-class identity with RBAC, and the Foundry Agent Service enforces it.

---

## 6. Where Does the Agent Run? VPC/VNet Integration

### Design: Three Deployment Tiers Mapped to Network Isolation Needs

| Tier | Description | Network Model | Best For |
|---|---|---|---|
| **Shared managed** | Platform runs the agent in a multi-tenant environment | Platform VPC, no customer VPC access | Declarative agents with public tools, prototyping |
| **Customer-connected managed** | Platform runs the agent but connects to customer VPC | VPC peering, Private Service Connect, Private Link | Declarative agents that call customer DBs/services |
| **Customer-hosted** | Agent runs inside customer's VPC/VNet | Full customer network control | Programmatic agents, regulated industries, data sovereignty |

### Platform Recommendation

- **Declarative Agents** naturally fit Tier 1 or Tier 2. The platform provides
  the runtime, and the agent config specifies what it needs. If it needs access
  to a customer database, the platform provisions a VPC peering or Private
  Service Connect endpoint.
- **Programmatic Agents** can run at any tier, but Tier 3 is common because
  custom code often has complex networking needs (connecting to internal
  microservices, databases, legacy systems).
- The platform should offer **all three tiers** and make it easy to move
  between them. An agent that starts as a quick prototype (Tier 1) should be
  deployable to a customer VPC (Tier 3) without rewriting code.

### How ADK Does This Today

Google offers the spectrum:

- Vertex AI Agent Engine (Tier 1/2)
- Cloud Run with VPC connectors (Tier 1/2)
- GKE (Tier 3)

A gap: the docs don't address how a managed Agent Engine deployment accesses
customer VPC resources. AWS Bedrock Agentcore is interesting here — it
explicitly supports deploying agent "workers" inside customer VPCs while the
control plane stays managed.

---

## 7. How Does the Agent Call Tools? Customer Resources

### Design: A Tool Registry with Connectivity and Auth as First-Class Concerns

| Tool Type | Connectivity | Auth Model |
|---|---|---|
| **Platform-provided tools** (Google Search, code execution) | Direct from platform | Platform-managed (no customer auth needed) |
| **Public API tools** (Stripe, Slack, GitHub) | Internet egress from agent runtime | OAuth2/API key managed by platform's credential service |
| **Customer-hosted tools** (internal APIs, MCP servers) | VPC peering / Private Link | Customer-managed credentials, possibly workload identity federation |
| **Customer databases** (Spanner, BigQuery, RDS, Cosmos DB) | VPC peering / Private Link | Service account with customer-granted IAM roles |

### Platform Recommendation

- **Tool Registry**: The platform should maintain a catalog of available tools.
  For declarative agents, tools are selected from this catalog by name. For
  programmatic agents, tools are imported in code but still registered for
  observability and permission enforcement.
- **Connectivity layer**: The platform should provide a **tool proxy** or
  **tool gateway** that handles:
  - Network routing (reach tools in customer VPC without exposing the full
    network)
  - Auth injection (add credentials to requests automatically)
  - Rate limiting and circuit breaking
  - Audit logging (every tool call logged with agent identity, tool name,
    parameters)
- **MCP as the universal tool protocol**: ADK already supports MCP with three
  transport protocols (stdio, SSE, Streamable HTTP). MCP is a good candidate
  for the standard interface between agents and customer-hosted tools — the
  customer deploys an MCP server in their VPC, the platform connects via
  Private Link.

### How ADK Does This Today

ADK's tool calling is sophisticated (parallel execution via `asyncio.gather()`,
full auth lifecycle, OpenTelemetry tracing), but it's **agent-side logic**. The
platform needs to complement this with **infrastructure-side** capabilities:
network connectivity, credential vaulting, and audit logging.

---

## 8. Where Is Memory Stored? Customer-Controlled Storage

### Design: Pluggable Storage Backends with Customer-Controlled Options

| Storage Type | Default (Platform-Managed) | Customer-Controlled Option |
|---|---|---|
| **Session state** | Platform's session service (in-memory or managed DB) | Customer Spanner / Firestore / DynamoDB via URI config |
| **Long-term memory** | Platform's memory service | Customer Vertex AI Search / RAG Engine / custom implementation |
| **Artifacts** | Platform's artifact service | Customer GCS bucket / S3 bucket |
| **Credentials** | Platform's credential service | Customer Secret Manager / HashiCorp Vault |

### Platform Recommendation

- **All storage should be pluggable via URI configuration.** ADK already
  supports this: `--session_service_uri`, `--artifact_service_uri`,
  `--memory_service_uri` flags in `adk deploy`. This is the right pattern.
- **For regulated customers**, all data must stay in their VPC. The platform
  should support:
  - Bring-your-own-database (BYODB) for sessions and memory
  - Customer-managed encryption keys (CMEK) for platform-managed storage
  - Data residency guarantees (store in specific regions)
- **Memory is especially sensitive** because it contains conversation history,
  which may include PII, business secrets, or regulated data. The platform must
  offer:
  - Customer-controlled memory backends (deploy a memory service in customer
    VPC)
  - Retention policies (auto-delete after N days)
  - Encryption at rest with customer keys

### How ADK Does This Today

ADK's `BaseSessionService` and `BaseCredentialService` are abstract interfaces
that can be implemented with any backend — good architecture for platform
flexibility. The default deployments (Vertex AI Agent Engine) use
platform-managed storage, and the CLI flags (`--session_service_uri`, etc.)
allow switching to customer-controlled backends without code changes.

---

## 9. Unified Platform Architecture

Both agent types should share the same control plane:

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Agent        │  │ Identity &   │  │ Tool Registry    │  │
│  │ Registry     │  │ Permission   │  │ & Connectivity   │  │
│  │              │  │ Service      │  │ Gateway          │  │
│  │ - Declarative│  │              │  │                  │  │
│  │   agents     │  │ - Workload   │  │ - MCP catalog    │  │
│  │ - Program-   │  │   identity   │  │ - Auth vault     │  │
│  │   matic      │  │ - RBAC       │  │ - VPC endpoints  │  │
│  │   agents     │  │ - Audit log  │  │ - Rate limiting  │  │
│  │ - Versions   │  │              │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Deployment   │  │ Observability│  │ Memory &         │  │
│  │ Orchestrator │  │ (Traces,     │  │ Session Service  │  │
│  │              │  │  Metrics,    │  │                  │  │
│  │ - Revisions  │  │  Logs)       │  │ - Platform DB    │  │
│  │ - Traffic    │  │              │  │ - Customer BYODB │  │
│  │   splitting  │  │              │  │ - CMEK support   │  │
│  │ - Rollback   │  │              │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
    │ Managed       │ │ Connected    │ │ Customer-Hosted  │
    │ Runtime       │ │ Managed      │ │ Runtime          │
    │               │ │ Runtime      │ │                  │
    │ Declarative   │ │ Either type  │ │ Programmatic     │
    │ agents run    │ │ with VPC     │ │ agents on        │
    │ here with     │ │ peering to   │ │ customer K8s/VMs │
    │ platform-     │ │ customer     │ │ with full network│
    │ provided loop │ │ resources    │ │ control          │
    └──────────────┘ └──────────────┘ └──────────────────┘
```

**Key principle**: The control plane is the same regardless of agent type. A
declarative agent and a programmatic agent both appear as entries in the Agent
Registry, both get identities from the same Identity Service, both have their
tool calls logged, and both can be deployed with traffic splitting and rollback.

The difference is only in the **data plane**: declarative agents run on a
platform-provided runtime engine, while programmatic agents run as
developer-provided containers.

This is analogous to how Kubernetes treats **Deployments** (you provide the
container) vs **Knative/Cloud Run** (you provide the code, the platform provides
the container) — same control plane, different levels of abstraction for the
workload.

---

## 10. Summary

### Naming Convention

| Context | Approach 1 | Approach 2 |
|---|---|---|
| **Developer docs** | Declarative Agent | Code-first Agent |
| **API / IaC types** | `declarative` | `programmatic` |
| **Internal shorthand** | Config-driven | Code-driven |

All names describe the same fundamental distinction:

| Term | Meaning |
|---|---|
| **Declarative / Config-driven Agent** | Developer declares model + prompt + tools + memory. Platform provides the runtime loop. |
| **Code-first / Programmatic Agent** | Developer writes the agent code, controls the loop and workflow. Platform provides infrastructure. |

See [Section 2](#2-naming-what-do-we-call-these-two-types-of-agents) for the
full naming analysis with 18 alternatives evaluated.

### Six Design Dimensions at a Glance

| Dimension | Declarative Agent | Code-first / Programmatic Agent |
|---|---|---|
| **Definition** | Config as IaC resource spec | Code artifact (container/package) |
| **Deployment** | Config patch → instant revision | Image build → rolling deployment |
| **Identity** | Auto-generated least-privilege policy | Developer-declared permission manifest |
| **Network** | Managed runtime + optional VPC peering | Any tier, often customer-hosted |
| **Tool calling** | Tools from platform catalog, auto-connected | Code-imported tools, registered for observability |
| **Memory/storage** | Platform-managed or customer BYODB | Same pluggable backends via URI config |

### Core Design Principle

> Both agent types share the same **control plane** (registry, identity,
> observability, deployment orchestration). The difference is the **data plane**:
> declarative agents run on a platform-provided engine, code-first/programmatic
> agents run as developer-provided containers.
