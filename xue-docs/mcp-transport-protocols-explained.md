# MCP Transport Protocols: stdio, SSE, and Streamable HTTP

## Context

This document explains the three transport protocols used by the Model Context
Protocol (MCP) in ADK Python — **stdio**, **SSE**, and **Streamable HTTP** — why
each exists, how they are used in the codebase, and how they compare with
traditional web transport protocols (HTTP/1.1, HTTP/2, WebSocket).

## What is MCP?

The **Model Context Protocol (MCP)** is a standardized protocol that lets AI
agents call external tools hosted in separate processes or servers. Think of it
as a "USB-C for AI tools" — a universal interface so any agent can talk to any
tool server. The question then becomes: **how do the bytes actually travel**
between the agent and the MCP tool server? That's where these three transports
come in.

## The Three MCP Transports

### 1. Stdio (Standard I/O)

**How it works:** The ADK spawns the MCP tool server as a **child process** on
the same machine. Communication happens through the process's **stdin**
(agent → server) and **stdout** (server → agent) — the same pipes you use when
you `echo "hello" | grep hello`.

**Why it exists:** It's the simplest, most reliable transport. No networking, no
ports, no TLS, no CORS — just two processes talking through OS pipes. Zero
configuration.

**When to use it:** Local development, CLI tools, or when the tool server is
bundled with your agent and runs on the same host.

**In ADK code:** `StdioConnectionParams` wraps `StdioServerParameters` (from the
MCP SDK). The session manager calls `stdio_client()` which spawns the child
process. It doesn't support HTTP headers (there's no HTTP involved), and there's
exactly one session key (`'stdio_session'`) since there's only one process.

```python
# Example: Run a local MCP server via stdio
McpToolset(connection_params=StdioConnectionParams(
    server_params=StdioServerParameters(
        command="npx", args=["-y", "@anthropic/mcp-server-filesystem", "/tmp"]
    )
))
```

### 2. SSE (Server-Sent Events)

**How it works:** The agent connects to a remote MCP server over HTTP. It uses
**Server-Sent Events** (a standard W3C API) for the **server → agent** direction
(streaming responses), and regular **HTTP POST** requests for the **agent →
server** direction (sending tool calls).

**Why it exists:** It enables **remote** MCP servers — the tool server can run
anywhere on the network. SSE is a mature, well-supported standard that works
through proxies, load balancers, and CDNs.

**When to use it:** When the MCP server is a remote service (e.g., running in a
cloud), especially when you need one-way streaming from the server.

**In ADK code:** `SseConnectionParams` takes a URL, optional headers (for auth
tokens, etc.), and a separate `sse_read_timeout` (defaults to 5 minutes) because
SSE connections are long-lived.

```python
# Example: Connect to a remote MCP server via SSE
McpToolset(connection_params=SseConnectionParams(
    url="https://my-mcp-server.example.com/sse",
    headers={"Authorization": "Bearer <token>"},
))
```

### 3. Streamable HTTP

**How it works:** This is the **newest** MCP transport. It uses standard HTTP
requests where the server **may optionally upgrade** the response to an SSE
stream. If the response is short, it returns as a normal HTTP response. If it's
long or streaming, it seamlessly upgrades to SSE within the same request.

**Why it exists:** It combines the simplicity of plain HTTP (stateless,
cacheable, works everywhere) with the streaming capability of SSE — **without
requiring a persistent connection**. It also supports session resumption and is
designed to be more robust for production deployments.

**When to use it:** Production deployments, especially when you need the
flexibility of both request/response and streaming patterns, or when you want
better compatibility with HTTP infrastructure.

**In ADK code:** `StreamableHTTPConnectionParams` adds a `httpx_client_factory`
for custom HTTPX client configuration (e.g., mTLS, custom proxies) and
`terminate_on_close` for lifecycle management.

```python
# Example: Connect via streamable HTTP
McpToolset(connection_params=StreamableHTTPConnectionParams(
    url="https://my-mcp-server.example.com/mcp",
    headers={"Authorization": "Bearer <token>"},
))
```

## How ADK Selects and Uses Transports

### Configuration

Each transport has a corresponding config class in `mcp_session_manager.py`:

| Config Class                      | Transport      | Key Fields                                  |
|-----------------------------------|----------------|---------------------------------------------|
| `StdioConnectionParams`          | Stdio          | `server_params`, `timeout`                  |
| `SseConnectionParams`            | SSE            | `url`, `headers`, `timeout`, `sse_read_timeout` |
| `StreamableHTTPConnectionParams` | Streamable HTTP | `url`, `headers`, `timeout`, `sse_read_timeout`, `httpx_client_factory` |

The `McpToolsetConfig` validator enforces that **exactly one** transport is
configured — they are mutually exclusive.

### Factory Pattern

In `MCPSessionManager._create_client()`, the transport type is determined by the
type of connection params:

```
StdioConnectionParams      → stdio_client()          (from mcp.client.stdio)
SseConnectionParams        → sse_client()            (from mcp.client.sse)
StreamableHTTPConnectionParams → streamablehttp_client() (from mcp.client.streamable_http)
```

### Session Pooling

- **Stdio:** Single session key (`'stdio_session'`) — one process, one session.
- **SSE / Streamable HTTP:** Session keys are MD5 hashes of merged headers,
  enabling per-identity session pooling.

### Timeout Handling

- **Stdio:** Uses `timeout` for read operations (connection is instant — it's a
  local process).
- **SSE / Streamable HTTP:** Uses `sse_read_timeout` (default 5 minutes) for
  read operations, since these are long-lived network connections.

## Comparison with Traditional Web Transport Protocols

| Aspect | HTTP/1.1 | HTTP/2 | WebSocket | MCP Stdio | MCP SSE | MCP Streamable HTTP |
|--------|----------|--------|-----------|-----------|---------|---------------------|
| **Direction** | Request/Response | Request/Response (multiplexed) | Full duplex | Full duplex (pipes) | Half duplex (POST up, SSE down) | Adaptive (req/res or streaming) |
| **Connection** | One req per conn (or keep-alive) | Multiplexed streams on one conn | Persistent, upgraded from HTTP | OS process pipes | Long-lived HTTP + SSE | Stateless HTTP (optionally streaming) |
| **Statefulness** | Stateless | Stateless | Stateful | Stateful (process lifetime) | Stateful (SSE connection) | **Stateless** (session ID optional) |
| **Proxy/LB friendly** | ✅ Very | ✅ Very | ⚠️ Needs special support | N/A (local only) | ✅ Yes (standard HTTP) | ✅ Very (pure HTTP) |
| **Streaming** | Chunked transfer only | Native stream support | Native | Native (pipes) | Server → Client only | Bidirectional (optional upgrade) |
| **Overhead** | High (headers per request) | Low (binary, compressed headers) | Very low (frames) | ~Zero | Moderate (HTTP + SSE) | Low-Moderate |
| **Use case** | Traditional web apps | Modern web apps, gRPC | Real-time apps (chat, games) | Local tool servers | Remote tool servers | Production tool servers |

### Key Insights

#### MCP SSE is built ON TOP of HTTP/1.1 (or HTTP/2)

SSE itself is just a special `Content-Type: text/event-stream` HTTP response. It
uses a standard HTTP connection — the innovation is that the response never
"ends." The server keeps pushing events down the pipe. For the agent → server
direction, it's plain HTTP POST. So MCP SSE is essentially **HTTP/1.1 + SSE**
working together.

#### MCP Streamable HTTP avoids WebSocket's problems

WebSockets were the traditional answer to "I need bidirectional streaming." But
they have real drawbacks: they require a protocol upgrade handshake, many
proxies/CDNs/load balancers don't handle them well, they're stateful (if the
connection drops, you lose context), and they bypass HTTP semantics (no caching,
no standard auth headers per message). Streamable HTTP keeps everything in
HTTP-land. Short interactions are plain request/response. Long interactions
seamlessly upgrade the *response body* to SSE — but the *request* is still a
normal HTTP POST. This means it works perfectly with all existing HTTP
infrastructure.

#### Stdio is analogous to Unix domain sockets / IPC

In traditional web app architecture, you might use Unix domain sockets or named
pipes for inter-process communication on the same host. MCP stdio serves the
same purpose — fast, zero-overhead local communication.

#### Why not just use WebSockets for MCP?

WebSockets require maintaining a persistent connection and managing reconnection
logic. They don't play well with serverless/FaaS environments. HTTP-based
transports (SSE and Streamable HTTP) are more resilient — if a connection drops,
the client can just make a new HTTP request. Streamable HTTP even supports
session resumption with session IDs. MCP is primarily a **request/response**
protocol (agent calls a tool, tool returns a result). Full-duplex streaming is
rarely needed. The occasional server→client notification (like progress updates)
is well-handled by SSE.

#### Evolution mirrors the broader web industry

Just as the web moved from HTTP/1.1 → HTTP/2 → HTTP/3 to get more efficient
multiplexing while staying HTTP-compatible, MCP is moving from SSE (the first
remote transport) → Streamable HTTP (more flexible, stateless, production-ready)
while keeping the same protocol semantics.

## Summary

| Transport | Think of it as... | Best for |
|-----------|-------------------|----------|
| **Stdio** | Unix pipes / IPC | Local dev, CLI tools, bundled servers |
| **SSE** | HTTP/1.1 + Server-Sent Events | Remote servers, simple deployments |
| **Streamable HTTP** | HTTP/2-style adaptive streaming | Production, serverless, scalable deployments |

The progression is: **local (stdio) → simple remote (SSE) → production remote
(streamable HTTP)**, each building on standard, well-understood web primitives
rather than inventing new protocols.
