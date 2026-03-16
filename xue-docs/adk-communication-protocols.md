# ADK Communication Protocols — Agent ↔ User/Client

How does a hosted ADK agent communicate with end-users and other agents? This
document covers the protocols, endpoints, request/response formats, and
streaming mechanisms available in Google ADK.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Primary User-Facing Protocol: Custom HTTP/JSON API](#2-primary-user-facing-protocol-custom-httpjson-api)
3. [Streaming Mechanisms](#3-streaming-mechanisms)
4. [A2A Protocol (Agent-to-Agent)](#4-a2a-protocol-agent-to-agent)
5. [Complete Endpoint Reference](#5-complete-endpoint-reference)
6. [Practical Example: Travel Agent Webapp](#6-practical-example-travel-agent-webapp)
7. [Summary](#7-summary)

---

## 1. Overview

ADK provides **multiple protocols** for communication between a hosted agent
and its consumers:

| Protocol | Use Case | Format | Streaming | Bidirectional |
|----------|----------|--------|-----------|---------------|
| **REST API** | Standard HTTP requests | JSON (Pydantic) | No | No |
| **SSE** | Real-time text updates to UI | JSON per SSE event | Yes | No |
| **WebSocket** | Live conversation / real-time audio | JSON frames | Yes | Yes |
| **A2A RPC** | Agent-to-agent calls | JSON-RPC (standard A2A) | Yes | Yes |

There is no single universal "standard protocol" for user↔agent communication
today. ADK uses its own well-structured JSON/HTTP API for that. For agent↔agent
communication, ADK supports the **A2A open standard**.

---

## 2. Primary User-Facing Protocol: Custom HTTP/JSON API

When you host an ADK agent (via `adk api_server` or `adk web`), it exposes a
**FastAPI-based HTTP server** with a custom JSON API. The key execution
endpoints are:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /run` | Synchronous | Send a message, get back a list of `Event` objects (JSON) |
| `POST /run_sse` | Streaming | Send a message, receive events as **Server-Sent Events** |
| `WS /run_live` | Bidirectional | Real-time WebSocket for live audio/text conversation |

### Request Format

A typical request to `/run` or `/run_sse`:

```json
{
  "app_name": "travel_agent",
  "user_id": "user_123",
  "session_id": "session_456",
  "new_message": {
    "role": "user",
    "parts": [{"text": "Book me a flight to Tokyo next Friday"}]
  }
}
```

This corresponds to the Pydantic model `RunAgentRequest`:

```python
class RunAgentRequest(BaseModel):
    app_name: str
    user_id: str
    session_id: str
    new_message: Optional[types.Content] = None
    streaming: bool = False
    state_delta: Optional[dict[str, Any]] = None
    invocation_id: Optional[str] = None
```

### Response Format

The response is a list of **ADK Event objects** serialized as JSON. Each Event
represents a step the agent took — an LLM response, a tool call, a tool result,
etc. Events are serialized via:

```python
event.model_dump_json(exclude_none=True, by_alias=True)
```

---

## 3. Streaming Mechanisms

### 3.1 Server-Sent Events (SSE) — `/run_sse`

- **Content-Type**: `text/event-stream`
- **Format**: `data: {JSON}\n\n` per event
- **Content**: Streams ADK Event objects as JSON, one per SSE message
- **Features**:
  - Supports splitting artifact deltas from content when both are present
  - Error handling: sends `{"error": "..."}` on exceptions
  - Used by ADK Web UI for real-time updates

**Best for**: Text-based chat UIs where the server pushes events progressively.

### 3.2 WebSocket — `/run_live`

- **URL**: `/run_live?app_name=...&user_id=...&session_id=...&modalities=...`
- **Query Parameters**:
  - `modalities`: `["TEXT"]` or `["AUDIO"]` (default: `["AUDIO"]`)
  - `proactive_audio`: Optional boolean
  - `enable_affective_dialog`: Optional boolean
  - `enable_session_resumption`: Optional boolean
- **Protocol**: WebSocket frames containing JSON
  - **Client → Server**: `LiveRequest` JSON objects
  - **Server → Client**: `Event` JSON objects
- **Closure**: Client sends `LiveRequest(close=True)` or closes the socket

The `LiveRequest` model:

```python
class LiveRequest(BaseModel):
    content: Optional[types.Content] = None
    blob: Optional[types.Blob] = None
    activity_start: Optional[types.ActivityStart] = None
    activity_end: Optional[types.ActivityEnd] = None
    close: bool = False
```

**Best for**: Voice/real-time UIs with full bidirectional communication.

### 3.3 Synchronous — `/run`

- Waits for full completion, then returns all events at once
- Simplest integration path but no progressive updates

---

## 4. A2A Protocol (Agent-to-Agent)

ADK supports the **[A2A (Agent-to-Agent) protocol](https://github.com/google/A2A)**,
an open standard developed by Google for inter-agent communication. This is
currently **experimental**.

### Enabling A2A

```bash
adk api_server --a2a --port 8000 path/to/agents
```

This exposes additional endpoints:

| Endpoint | Purpose |
|----------|---------|
| `/a2a/{app_name}` | JSON-RPC endpoint following the A2A spec |
| `/a2a/{app_name}/.well-known/a2a/agent` | Machine-readable **Agent Card** |

### How A2A Works

1. The server scans for `agent.json` files (Agent Cards) in the agents
   directory.
2. It creates `A2AStarletteApplication` instances for each agent with A2A
   support.
3. Incoming A2A requests are converted from A2A format to ADK format, processed
   by the ADK runner, and responses are converted back.

### Key A2A Components

- **`A2aAgentExecutor`** — Receives A2A `RequestContext`, converts to ADK
  `AgentRunRequest`, runs the agent, converts `Event` objects back to A2A
  events.
- **`RemoteA2aAgent`** — Allows a local ADK agent to call a remote A2A agent.
  Accepts an `AgentCard` (URL, file path, or object) and uses the A2A client to
  communicate.
- **Part Converters** — Bidirectional conversion between ADK
  `google.genai.types.Part` and A2A `Part` objects (text, file, data parts).
- **Event Converters** — Maps ADK Events to A2A `TaskStatusUpdateEvent` and
  `TaskArtifactUpdateEvent`.

### A2A Use Cases

- **Agent delegation**: Your travel agent calling a specialized hotel-booking
  agent hosted elsewhere.
- **Multi-vendor agents**: Connecting agents built with different frameworks
  that all speak A2A.
- **OAuth flows**: A2A agents can surface OAuth flows to parent agents for
  authentication.

---

## 5. Complete Endpoint Reference

### Session Management

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /apps/{app_name}/users/{user_id}/sessions` | POST | Create session |
| `GET /apps/{app_name}/users/{user_id}/sessions` | GET | List sessions |
| `GET /apps/{app_name}/users/{user_id}/sessions/{session_id}` | GET | Get session |
| `DELETE /apps/{app_name}/users/{user_id}/sessions/{session_id}` | DELETE | Delete session |
| `PATCH /apps/{app_name}/users/{user_id}/memory` | PATCH | Update memory |

### Agent Execution

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /run` | POST | Synchronous execution |
| `POST /run_sse` | POST | Server-Sent Events streaming |
| `WS /run_live` | WebSocket | Bidirectional real-time |

### Agent Metadata

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /apps/{app_name}` | GET | Get app details |
| `GET /list-apps` | GET | List all agents |
| `GET /health` | GET | Server health check |
| `GET /version` | GET | Version info |

### Artifacts

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /artifacts/save` | POST | Save artifacts |
| `GET /artifacts/...` | GET | Get artifact metadata |

### Evaluation

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /eval/run` | POST | Run evaluations |
| `GET /eval/results` | GET | Get evaluation results |

### Debugging

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /debug/trace/{event_id}` | GET | Get event trace |
| `GET /debug/trace/session/{session_id}` | GET | Get session trace |

---

## 6. Practical Example: Travel Agent Webapp

Consider a travel agent built with ADK that can research travel plans, book
flights and hotels, and create Google Calendar events. Here is how the protocols
apply:

### Frontend ↔ Agent

Your webapp frontend calls `/run_sse` (for streaming chat) or `/run` (for
simple request/response). These use **ADK's custom JSON API**. You build your
frontend to match this format.

```
┌─────────────┐     POST /run_sse      ┌────────────────┐
│   Web App    │ ───────────────────▶   │   ADK Server   │
│  (React,     │   RunAgentRequest      │  (FastAPI)     │
│   Vue, etc.) │ ◀─────────────────── │                  │
│              │   SSE Event stream     │  travel_agent  │
└─────────────┘                         └────────────────┘
```

### Agent ↔ Other Agents

If your travel agent delegates to other agents (e.g., a specialized
hotel-booking agent hosted elsewhere), it can use **A2A** via
`RemoteA2aAgent`:

```
┌────────────────┐     A2A JSON-RPC     ┌────────────────┐
│  travel_agent  │ ───────────────────▶ │  hotel_agent   │
│  (local ADK)   │                      │  (remote A2A)  │
│                │ ◀─────────────────── │                │
└────────────────┘                       └────────────────┘
```

### Session Management

ADK manages per-user sessions automatically. Each user gets their own `user_id`
and `session_id`, so conversation state is isolated between users.

---

## 7. Summary

| Communication Path | Protocol | Standard? |
|--------------------|----------|-----------|
| **User ↔ Agent** | ADK HTTP/JSON API (REST, SSE, WebSocket) | ADK-specific |
| **Agent ↔ Agent** | A2A (JSON-RPC) | Open standard (experimental) |
| **Agent ↔ Tools** | Function calls via LLM | Internal to ADK |

**Key takeaways:**

- **User-facing**: ADK exposes its own custom HTTP/JSON API. Your frontend
  needs to speak this format. There is no universal user↔agent standard yet.
- **Agent-facing**: ADK supports the **A2A open standard** for agent-to-agent
  communication, enabling interoperability with agents built on other
  frameworks.
- **Streaming**: SSE for progressive text, WebSocket for real-time bidirectional
  (audio + text).
- **Session isolation**: Built-in per-user session management keeps
  conversations separate.

As the A2A ecosystem matures, it could become the standard for client↔agent
communication as well.
