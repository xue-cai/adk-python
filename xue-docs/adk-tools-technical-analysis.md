# ADK Python — Tools System Technical Analysis

A deep-dive into the Agent Development Kit (ADK) tools architecture: how tools
are defined, how they integrate with MCP servers, how skills work, how
authentication is handled, and how the entire execution pipeline fits together.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Abstractions](#2-core-abstractions)
3. [Tool Types at a Glance](#3-tool-types-at-a-glance)
4. [FunctionTool — Wrapping Python Functions](#4-functiontool--wrapping-python-functions)
5. [MCP Tool Integration](#5-mcp-tool-integration)
6. [OpenAPI / REST API Tools](#6-openapi--rest-api-tools)
7. [Google API Tools](#7-google-api-tools)
8. [Built-in Google Search & Grounding Tools](#8-built-in-google-search--grounding-tools)
9. [Database Tools (BigQuery, Bigtable, Spanner, Pub/Sub)](#9-database-tools-bigquery-bigtable-spanner-pubsub)
10. [Third-Party Adapters (LangChain, CrewAI)](#10-third-party-adapters-langchain-crewai)
11. [Skill Toolset](#11-skill-toolset)
12. [Retrieval Tools](#12-retrieval-tools)
13. [Utility Tools](#13-utility-tools)
14. [Authentication System](#14-authentication-system)
15. [Tool Execution Flow (Runner → Agent → Tool)](#15-tool-execution-flow-runner--agent--tool)
16. [ToolContext — What Tools Receive at Runtime](#16-toolcontext--what-tools-receive-at-runtime)
17. [Toolsets and Filtering](#17-toolsets-and-filtering)
18. [Summary & Key Takeaways](#18-summary--key-takeaways)

---

## 1. Architecture Overview

The tools subsystem lives under `src/google/adk/tools/` and contains **40+
modules** organized into:

```
tools/
├── base_tool.py                  # Abstract base class for ALL tools
├── base_toolset.py               # Abstract base class for tool collections
├── base_authenticated_tool.py    # Base class for auth-aware tools
├── function_tool.py              # Wraps plain Python callables
├── tool_context.py               # Runtime context passed to every tool
│
├── mcp_tool/                     # Model Context Protocol integration
├── openapi_tool/                 # REST API tool generation from OpenAPI specs
├── google_api_tool/              # Google API-specific toolset
│
├── bigquery/                     # BigQuery tools
├── bigtable/                     # Bigtable tools
├── spanner/                      # Spanner tools
├── pubsub/                       # Pub/Sub tools
│
├── retrieval/                    # RAG / document retrieval
├── computer_use/                 # Screen-control tool
├── agent_simulator/              # Mock tool execution for testing
├── skill_toolset.py              # Skills discovery and loading
├── application_integration_tool/ # Google Application Integration
├── apihub_tool/                  # Google API Hub
├── data_agent/                   # Data agent interaction
│
├── langchain_tool.py             # LangChain adapter
├── crewai_tool.py                # CrewAI adapter
├── google_search_tool.py         # Gemini built-in Google Search
├── google_maps_grounding_tool.py # Gemini built-in Google Maps
├── enterprise_search_tool.py     # Enterprise web search grounding
├── load_memory_tool.py           # Session memory search
├── load_artifacts_tool.py        # Artifact loading
├── long_running_tool.py          # Long-running operations
├── example_tool.py               # Few-shot example injection
├── exit_loop_tool.py             # Loop exit signal
├── load_web_page.py              # Web page content fetcher
└── get_user_choice_tool.py       # User choice prompt
```

**Key design principles:**

- **Every tool is a `BaseTool`** — a single abstract interface with `name`,
  `description`, `run_async()`, and optional `process_llm_request()`.
- **Toolsets (`BaseToolset`)** — group multiple tools together and provide
  filtering / prefixing.
- **ToolContext** — a rich runtime context object passed to every tool, giving
  access to session state, artifacts, credentials, and actions.
- **Authentication** is layered: `BaseAuthenticatedTool`, `ToolAuthHandler`,
  and credential exchangers handle OAuth2, API keys, service accounts, and
  OpenID Connect.

---

## 2. Core Abstractions

### `BaseTool` — The Root of All Tools

Every tool in ADK inherits from `BaseTool`:

```python
# src/google/adk/tools/base_tool.py

class BaseTool(ABC):
    """The base class for all tools."""

    name: str
    description: str
    is_long_running: bool = False
    custom_metadata: Optional[dict[str, Any]] = None

    def __init__(self, *, name, description, is_long_running=False, custom_metadata=None):
        self.name = name
        self.description = description
        self.is_long_running = is_long_running
        self.custom_metadata = custom_metadata

    def _get_declaration(self) -> Optional[types.FunctionDeclaration]:
        """Returns the Gemini FunctionDeclaration for this tool."""
        return None

    async def run_async(self, *, args: dict[str, Any], tool_context: ToolContext) -> Any:
        """Execute the tool. Override in subclasses."""
        ...

    async def process_llm_request(self, *, tool_context: ToolContext, llm_request: LlmRequest) -> None:
        """Hook to modify the LLM request before it's sent (e.g., inject system instructions)."""
        del tool_context, llm_request
```

**Two critical methods:**

| Method | Purpose |
|--------|---------|
| `run_async()` | Called by the agent when the LLM invokes this tool via function-calling. Receives `args` (from the LLM) and `tool_context`. |
| `process_llm_request()` | Called *before* the LLM request is sent. Lets tools inject system instructions, modify the request config, or add grounding tools. |

### `BaseToolset` — Grouping & Filtering

```python
# src/google/adk/tools/base_toolset.py

class BaseToolset(ABC):
    def __init__(self, tool_filter=None, tool_name_prefix=None):
        self._tool_filter = tool_filter       # List[str] or ToolPredicate callable
        self._tool_name_prefix = tool_name_prefix

    @abstractmethod
    async def get_tools(self, readonly_context=None) -> list[BaseTool]:
        """Return all tools in this toolset."""
        ...

    def _is_tool_selected(self, tool, readonly_context) -> bool:
        """Apply filter: either a name list or a predicate function."""
        if isinstance(self._tool_filter, list):
            return tool.name in self._tool_filter
        elif callable(self._tool_filter):
            return self._tool_filter(tool, readonly_context)
        return True
```

---

## 3. Tool Types at a Glance

| Category | Tool / Toolset | How It Works |
|----------|---------------|--------------|
| **Python Functions** | `FunctionTool` | Wraps any Python callable |
| **MCP Servers** | `McpToolset` / `McpTool` | Connects to MCP servers via stdio/SSE/HTTP |
| **REST APIs** | `OpenAPIToolset` / `RestApiTool` | Generates tools from OpenAPI specs |
| **Google APIs** | `GoogleApiToolset` / `GoogleApiTool` | Converts Google Discovery APIs → OpenAPI → tools |
| **Gemini Built-ins** | `GoogleSearchTool`, `GoogleMapsGroundingTool` | Injected into Gemini's request config (not function-called) |
| **Database** | `BigQueryToolset`, `BigtableToolset`, `SpannerToolset`, `PubSubToolset` | Direct SDK calls to Google Cloud databases |
| **Retrieval** | `BaseRetrievalTool`, `VertexAiRagRetrieval`, `LlamaIndexRetrieval` | Document retrieval for RAG |
| **Third-Party** | `LangchainTool`, `CrewaiTool` | Adapters for LangChain / CrewAI tools |
| **Skills** | `SkillToolset` | Discovers and loads skill instruction folders |
| **Utility** | `LoadMemoryTool`, `LoadArtifactsTool`, `ExampleTool`, `ExitLoopTool` | Session memory, artifacts, few-shot examples, loop control |

---

## 4. FunctionTool — Wrapping Python Functions

The most common tool type. Wraps any Python function — sync or async — as a
Gemini-compatible tool:

```python
# src/google/adk/tools/function_tool.py

class FunctionTool(BaseTool):
    def __init__(self, func: Callable[..., Any], *, require_confirmation=False):
        name = func.__name__
        description = func.__doc__ or ''
        super().__init__(name=name, description=description)
        self.func = func
        self._require_confirmation = require_confirmation

    def _get_declaration(self) -> types.FunctionDeclaration:
        """Auto-generates a FunctionDeclaration from the Python function signature."""
        return build_function_declaration(func=self.func, ignore_params=self._ignore_params)

    async def run_async(self, *, args, tool_context):
        # 1. Validate mandatory args
        # 2. Handle confirmation if required
        # 3. Preprocess args (convert JSON dicts to Pydantic models, etc.)
        # 4. Invoke the callable (handles both sync and async)
        return await self._invoke_callable(tool_context, **args)
```

**How declaration auto-generation works:**

`build_function_declaration()` inspects the Python function's type hints and
docstring, then builds an OpenAPI-style JSON schema that Gemini uses for
function-calling. Parameters like `tool_context` are automatically excluded.

**Usage:**

```python
from google.adk.tools import FunctionTool

def get_weather(city: str, unit: str = "celsius") -> str:
    """Get weather for a city."""
    return f"Weather in {city}: 22°{unit[0].upper()}"

weather_tool = FunctionTool(get_weather)
# Or simply pass the function directly to Agent(tools=[get_weather])
```

---

## 5. MCP Tool Integration

ADK has first-class support for the **Model Context Protocol (MCP)**. This is
one of the most sophisticated integrations in the tools system.

### How MCP Works in ADK

```
┌─────────────┐        ┌──────────────────┐        ┌─────────────────┐
│  ADK Agent   │───────▶│    McpToolset     │───────▶│   MCP Server    │
│              │        │  (BaseToolset)    │        │  (stdio/SSE/    │
│  tools=[     │        │                  │        │   HTTP)         │
│    McpToolset│        │  ┌──────────────┐│        │                 │
│  ]           │        │  │ McpTool #1   ││        │  tools/list     │
│              │        │  │ McpTool #2   ││◀──────▶│  tools/call     │
│              │        │  │ McpTool #N   ││        │  resources/read │
│              │        │  └──────────────┘│        │                 │
└─────────────┘        └──────────────────┘        └─────────────────┘
```

### Connection Types

ADK supports all three MCP transport protocols:

```python
# 1. Stdio — Spawns a child process
from mcp import StdioServerParameters
from google.adk.tools.mcp_tool import McpToolset, StdioConnectionParams

toolset = McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command='npx',
            args=["-y", "@modelcontextprotocol/server-filesystem"],
        ),
        timeout=5.0,
    )
)

# 2. SSE — Connects to HTTP Server-Sent Events endpoint
from google.adk.tools.mcp_tool import SseConnectionParams

toolset = McpToolset(
    connection_params=SseConnectionParams(
        url="http://localhost:8080/sse",
        headers={"Authorization": "Bearer ..."},
        timeout=5.0,
        sse_read_timeout=300.0,
    )
)

# 3. Streamable HTTP — Newer HTTP-based MCP transport
from google.adk.tools.mcp_tool import StreamableHTTPConnectionParams

toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="http://localhost:8080/mcp",
        headers={"Authorization": "Bearer ..."},
        timeout=5.0,
    )
)
```

### Session Management & Pooling

The `MCPSessionManager` manages MCP client sessions with connection pooling:

```python
# src/google/adk/tools/mcp_tool/mcp_session_manager.py

class MCPSessionManager:
    def __init__(self, connection_params, errlog=None):
        self._connection_params = connection_params
        self._session_pool: Dict[str, SessionContext] = {}
        self._lock = asyncio.Lock()

    def _generate_session_key(self, extra_headers=None) -> str:
        """Pool key based on headers hash — different auth = different session."""
        merged = self._merge_headers(extra_headers)
        return hashlib.md5(json.dumps(merged, sort_keys=True).encode()).hexdigest()

    async def create_session(self, extra_headers=None) -> ClientSession:
        """Returns a pooled session or creates a new one."""
        key = self._generate_session_key(extra_headers)
        async with self._lock:
            if key in self._session_pool:
                ctx = self._session_pool[key]
                if not self._is_session_disconnected(ctx.session):
                    return ctx.session
            # Create new session via _create_client()
            ...
```

Sessions are pooled by header hash — this means different auth tokens get
different sessions, while identical requests share connections.

### McpTool — Individual Tool Wrapper

Each tool discovered from an MCP server becomes an `McpTool`:

```python
# src/google/adk/tools/mcp_tool/mcp_tool.py

class McpTool(BaseAuthenticatedTool):
    """Wraps an individual MCP tool as an ADK BaseTool."""

    def __init__(self, mcp_tool, mcp_session_manager, auth_scheme=None,
                 auth_credential=None, require_confirmation=False,
                 progress_callback=None):
        self._mcp_tool = mcp_tool            # mcp.types.Tool from MCP server
        self._mcp_session_manager = mcp_session_manager
        self._progress_callback = progress_callback

    @retry_on_errors()
    async def _run_async_impl(self, *, args, tool_context, credential=None):
        """Execute the tool by calling the MCP server."""
        extra_headers = self._get_headers(credential)

        # Inject OpenTelemetry trace context into MCP headers
        carrier = {}
        propagate.inject(carrier)
        extra_headers.update(carrier)

        session = await self._mcp_session_manager.create_session(extra_headers)
        progress_callback = self._resolve_progress_callback(tool_context)

        result = await session.call_tool(
            self.name, args, read_timeout_seconds=None,
            progress_callback=progress_callback,
        )
        return self._process_result(result)
```

**Key features:**

- **Auto-retry**: The `@retry_on_errors()` decorator automatically retries on
  session errors (but not on cancellation).
- **OpenTelemetry**: Trace context is propagated to MCP servers via headers.
- **Progress callbacks**: Supports `ProgressCallbackFactory` for real-time
  progress reporting with access to session state.
- **Auth forwarding**: Credentials are converted to HTTP headers and passed to
  the MCP session.

### MCP Resource Loading

ADK also supports MCP resources via `LoadMcpResourceTool`:

```python
# src/google/adk/tools/load_mcp_resource_tool.py

class LoadMcpResourceTool(BaseTool):
    """Loads MCP resources into the agent's session."""

    async def _append_resources_to_llm_request(self, *, tool_context, llm_request):
        """Fetches resources from MCP server and converts to LLM-compatible parts."""
        for resource_name in self._requested_resource_names:
            resource = await self._mcp_toolset.read_resource(resource_name)
            for content in resource.contents:
                part = self._mcp_content_to_part(content)  # text or blob
                llm_request.contents.append(types.Content(parts=[part]))
```

---

## 6. OpenAPI / REST API Tools

ADK can auto-generate tools from any OpenAPI 3.x specification.

### How It Works

```
OpenAPI Spec (JSON/YAML)
        │
        ▼
┌──────────────────┐
│ OpenAPIToolset    │  Parses spec, generates one tool per operation
│                   │
│  ┌──────────────┐│
│  │ RestApiTool   ││  GET /users → rest_api_get_users
│  │ RestApiTool   ││  POST /users → rest_api_post_users
│  │ RestApiTool   ││  ...
│  └──────────────┘│
└──────────────────┘
```

```python
# src/google/adk/tools/openapi_tool/openapi_spec_parser/openapi_toolset.py

class OpenAPIToolset(BaseToolset):
    def __init__(self, spec_str=None, spec_str_type='json', spec_dict=None,
                 auth_scheme=None, auth_credential=None):
        # Parse spec → list of RestApiTool
        self._tools = OpenApiSpecParser.parse(spec_dict)
        # Apply global auth to all tools
        for tool in self._tools:
            tool.configure_auth(auth_scheme, auth_credential)
```

### RestApiTool — Making HTTP Calls

```python
# src/google/adk/tools/openapi_tool/openapi_spec_parser/rest_api_tool.py

class RestApiTool(BaseTool):
    async def call(self, *, args, tool_context):
        # 1. Prepare auth credentials
        auth_handler = ToolAuthHandler(auth_scheme=self._auth_scheme, ...)
        credential = await auth_handler.prepare_auth_credentials(tool_context)

        # 2. Build request parameters (path, query, headers, body)
        params = self._prepare_request_params(args)

        # 3. Attach auth to request
        params = self._prepare_auth_request_params(credential, params)

        # 4. Execute HTTP request
        async with httpx.AsyncClient(verify=self._should_verify_ssl) as client:
            response = await client.request(
                method=params.method,
                url=params.url,
                headers=params.headers,
                params=params.query_params,
                json=params.body,
            )
        return response.json()
```

---

## 7. Google API Tools

Google APIs are exposed through a two-step conversion pipeline:

```
Google Discovery API Spec
        │
        ▼
┌──────────────────────────────┐
│ GoogleApiToOpenApiConverter   │  Converts Discovery → OpenAPI 3.0
└──────────────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│ OpenAPIToolset                │  Standard OpenAPI parsing
└──────────────────────────────┘
        │
        ▼
┌──────────────────────────────┐
│ GoogleApiTool (wrapper)       │  Adds OAuth2 / Service Account auth
└──────────────────────────────┘
```

```python
# src/google/adk/tools/google_api_tool/google_api_toolset.py

class GoogleApiToolset(BaseToolset):
    def __init__(self, api_name: str, api_version: str,
                 client_id=None, client_secret=None, scopes=None):
        # Fetch Google Discovery spec and convert to OpenAPI
        self._converter = GoogleApiToOpenApiConverter(api_name, api_version)

    async def get_tools(self, readonly_context=None) -> List[GoogleApiTool]:
        openapi_spec = self._converter.convert()
        openapi_toolset = OpenAPIToolset(spec_dict=openapi_spec)
        return [
            GoogleApiTool(rest_api_tool=tool, client_id=..., client_secret=...)
            for tool in openapi_toolset.get_tools()
        ]
```

```python
# src/google/adk/tools/google_api_tool/google_api_tool.py

class GoogleApiTool(BaseTool):
    """Wraps a RestApiTool with Google-specific auth."""

    def configure_auth(self, client_id: str, client_secret: str):
        """Set up OAuth2 for this Google API tool."""
        self._rest_api_tool.configure_auth(
            auth_scheme=oauth2_scheme,
            auth_credential=AuthCredential(
                auth_type=AuthCredentialTypes.OAUTH2,
                oauth2=OAuth2Auth(client_id=client_id, client_secret=client_secret)
            )
        )

    async def run_async(self, *, args, tool_context):
        return await self._rest_api_tool.call(args=args, tool_context=tool_context)
```

---

## 8. Built-in Google Search & Grounding Tools

These tools work differently from function-calling tools. Instead of being
invoked via `run_async()`, they modify the LLM request configuration to enable
Gemini's built-in grounding capabilities.

### GoogleSearchTool

```python
# src/google/adk/tools/google_search_tool.py

class GoogleSearchTool(BaseTool):
    """Built-in tool automatically invoked by Gemini 2 for Google Search grounding."""

    async def process_llm_request(self, *, tool_context, llm_request):
        # Inject GoogleSearch into the LLM request config
        google_search_tool = types.Tool(google_search=types.GoogleSearch())
        llm_request.config.tools = llm_request.config.tools or []
        llm_request.config.tools.append(google_search_tool)
```

This **does not** result in a function call → response cycle. Instead, Gemini
natively performs the search and incorporates grounding information into its
response.

### GoogleMapsGroundingTool

```python
class GoogleMapsGroundingTool(BaseTool):
    async def process_llm_request(self, *, tool_context, llm_request):
        google_maps_tool = types.Tool(google_maps=types.GoogleMaps())
        llm_request.config.tools.append(google_maps_tool)
```

### EnterpriseWebSearchTool

```python
class EnterpriseWebSearchTool(BaseTool):
    async def process_llm_request(self, *, tool_context, llm_request):
        enterprise_search = types.Tool(
            enterprise_web_search=types.EnterpriseWebSearch()
        )
        llm_request.config.tools.append(enterprise_search)
```

**Important:** These grounding tools only work with Gemini 2+ models and modify
the LLM request via `process_llm_request()` rather than being called through
the function-calling flow.

---

## 9. Database Tools (BigQuery, Bigtable, Spanner, Pub/Sub)

Each database toolset provides pre-built tools for interacting with Google Cloud
databases. They all use `GoogleTool` as a base, which handles credential
injection.

### BigQuery Toolset

```python
# Available tools:
# - list_dataset_ids, list_table_ids, get_dataset_info, get_table_info
# - execute_sql, forecast, detect_anomalies, analyze_contribution
# - ask_data_insights, get_job_info

async def execute_sql(project_id, query, credentials, settings,
                      tool_context, dry_run=False):
    """Execute SQL with write protection modes (BLOCKED/PROTECTED/PERMISSIVE)."""
    client = bigquery.Client(project=project_id, credentials=credentials)
    job_config = bigquery.QueryJobConfig(dry_run=dry_run)
    query_job = client.query(query, job_config=job_config)
    return {"status": "SUCCESS", "rows": [...]}
```

### Spanner Toolset

```python
# Available tools:
# - list_table_names, get_table_schema, list_table_indexes
# - execute_sql, similarity_search, vector_store_similarity_search

async def execute_sql(project_id, instance_id, database_id, query,
                      credentials, settings, tool_context):
    """Read-only SQL queries on Spanner."""
    client = spanner.Client(project=project_id, credentials=credentials)
    instance = client.instance(instance_id)
    database = instance.database(database_id)
    # Execute as snapshot (read-only)
    ...
```

### Bigtable Toolset

```python
# Available tools:
# - list_instances, get_instance_info, list_tables, get_table_info
# - execute_sql (GoogleSQL)

async def execute_sql(project_id, instance_id, query, credentials,
                      settings, tool_context):
    """Execute GoogleSQL on Bigtable (50 row default limit)."""
    ...
```

### Pub/Sub Toolset

```python
# Available tools:
# - publish_message, pull_messages, acknowledge_messages

async def publish_message(topic_name, message, credentials, settings,
                          attributes=None, ordering_key=""):
    publisher = pubsub_v1.PublisherClient(credentials=credentials)
    future = publisher.publish(topic_name, message.encode('utf-8'), **attributes)
    return {"status": "SUCCESS", "message_id": future.result()}
```

**Auth pattern:** All database tools accept `credentials` and `settings`
parameters. The `GoogleTool` base class automatically injects Google Cloud
credentials via `GoogleCredentialsManager`.

---

## 10. Third-Party Adapters (LangChain, CrewAI)

### LangchainTool

Wraps any LangChain tool as an ADK `FunctionTool`:

```python
# src/google/adk/tools/langchain_tool.py

class LangchainTool(FunctionTool):
    def __init__(self, tool: Union[LangchainBaseTool, object],
                 name=None, description=None):
        # Extract the callable function
        if isinstance(tool, StructuredTool):
            func = tool.func  # or tool.coroutine for async
        elif hasattr(tool, '_run'):
            func = tool._run
        else:
            func = tool.run

        super().__init__(func)
        self._ignore_params.append('run_manager')  # LangChain internal param
```

**Usage:**

```python
from langchain_community.tools import DuckDuckGoSearchTool
from google.adk.tools import LangchainTool

search = LangchainTool(DuckDuckGoSearchTool())
agent = Agent(tools=[search])
```

### CrewaiTool

Wraps CrewAI tools with special handling for their `**kwargs` pattern:

```python
# src/google/adk/tools/crewai_tool.py

class CrewaiTool(FunctionTool):
    def __init__(self, tool: CrewaiBaseTool, *, name, description):
        func = tool._run
        super().__init__(func)

    async def run_async(self, *, args, tool_context):
        # Filter args to only include params the function actually accepts
        sig = inspect.signature(self.func)
        filtered_args = {
            k: v for k, v in args.items()
            if k in sig.parameters and k not in ('self', 'kwargs')
        }
        return await super().run_async(args=filtered_args, tool_context=tool_context)
```

---

## 11. Skill Toolset

Skills are structured instruction folders that extend an agent's capabilities.
The `SkillToolset` provides three sub-tools for skill management:

```python
# src/google/adk/tools/skill_toolset.py

class SkillToolset(BaseToolset):
    """Manages skill discovery and loading."""

    def __init__(self, skills: list[models.Skill]):
        self._skills = skills

    async def get_tools(self, readonly_context=None) -> list[BaseTool]:
        return [
            ListSkillsTool(self),     # Lists all available skills
            LoadSkillTool(self),      # Loads a specific skill's instructions
            LoadSkillResourceTool(self),  # Loads files within a skill folder
        ]

    async def process_llm_request(self, *, tool_context, llm_request):
        """Inject skill system instructions into LLM request."""
        llm_request.config.system_instruction += DEFAULT_SKILL_SYSTEM_INSTRUCTION
```

**Skill folder structure:**

```
my_skill/
├── SKILL.md           # Main instructions with metadata
├── references/        # Documentation / examples
└── assets/            # Templates, scripts, resources
```

**How skills work at runtime:**

1. Agent receives the skill system instruction telling it skills are available.
2. Agent calls `list_skills` to see what's available.
3. Agent calls `load_skill(name="my_skill")` to read `SKILL.md`.
4. Agent follows the instructions from the skill.
5. If needed, agent calls `load_skill_resource(name="my_skill", path="assets/template.txt")`.

Skills are **not** MCP servers. They are local folders of markdown instructions
and resources that are dynamically loaded into the LLM context.

---

## 12. Retrieval Tools

ADK provides multiple retrieval backends for RAG (Retrieval-Augmented
Generation):

```python
# src/google/adk/tools/retrieval/base_retrieval_tool.py

class BaseRetrievalTool(BaseTool):
    """Abstract base for all retrieval tools."""

    def _get_declaration(self):
        return types.FunctionDeclaration(
            name=self.name,
            description=self.description,
            parameters=types.Schema(
                type="OBJECT",
                properties={"query": types.Schema(type="STRING")},
                required=["query"],
            ),
        )
```

**Available backends:**

| Backend | Class | Description |
|---------|-------|-------------|
| Vertex AI RAG | `VertexAiRagRetrieval` | Uses Vertex AI's RAG engine |
| LlamaIndex | `LlamaIndexRetrieval` | Integrates LlamaIndex retrievers |
| Files | `FilesRetrieval` | Local file-based retrieval |

---

## 13. Utility Tools

### LoadMemoryTool — Session Memory Search

```python
# src/google/adk/tools/load_memory_tool.py

class LoadMemoryTool(FunctionTool):
    async def process_llm_request(self, *, tool_context, llm_request):
        """Add memory instruction to system prompt."""
        llm_request.config.system_instruction += MEMORY_INSTRUCTION

async def load_memory(query: str, tool_context: ToolContext) -> LoadMemoryResponse:
    """Search session memory for relevant past interactions."""
    results = await tool_context.search_memory(query)
    return LoadMemoryResponse(memories=results)
```

### LoadArtifactsTool — Artifact Loading

```python
class LoadArtifactsTool(BaseTool):
    async def run_async(self, *, args, tool_context):
        """Load a specific artifact by filename and optional version."""
        artifact = await tool_context.load_artifact(filename, version)
        return artifact

    async def process_llm_request(self, *, tool_context, llm_request):
        """Append all session artifacts to the LLM request."""
        # Converts unsupported MIME types to text
        # Adds artifact content inline to request
```

### ExampleTool — Few-Shot Example Injection

```python
class ExampleTool(BaseTool):
    def __init__(self, examples: Union[list[Example], BaseExampleProvider]):
        self._examples = examples

    async def process_llm_request(self, *, tool_context, llm_request):
        """Inject few-shot examples into LLM system instruction."""
        examples = self._examples  # or call provider
        si = example_util.build_example_si(examples)
        llm_request.config.system_instruction += si
```

### ExitLoopTool — Loop Escape

```python
def exit_loop(tool_context: ToolContext):
    """Signal the agent to exit the current loop."""
    tool_context.actions.escalate = True
    tool_context.actions.skip_summarization = True
```

### LongRunningFunctionTool

```python
class LongRunningFunctionTool(FunctionTool):
    """Marks a tool as long-running (returns a resource ID, finishes later)."""
    def __init__(self, func):
        super().__init__(func)
        self.is_long_running = True
```

### ComputerUseTool

```python
class ComputerUseTool(FunctionTool):
    """Wraps screen-control functions with coordinate normalization."""
    # Normalizes coordinates from virtual space (1000x1000) to actual screen size
    # Allows LLMs to work with a consistent coordinate system
```

---

## 14. Authentication System

Authentication is one of the most sophisticated parts of the tools system. It
supports multiple auth mechanisms and handles the complete credential lifecycle.

### Auth Data Models

```python
# src/google/adk/auth/auth_credential.py

class AuthCredentialTypes(str, Enum):
    API_KEY = "apiKey"
    HTTP = "http"
    OAUTH2 = "oauth2"
    OPEN_ID_CONNECT = "openIdConnect"
    SERVICE_ACCOUNT = "serviceAccount"

class AuthCredential(BaseModel):
    auth_type: AuthCredentialTypes
    resource_ref: Optional[str] = None
    api_key: Optional[str] = None
    http: Optional[HttpAuth] = None           # Bearer tokens, Basic auth
    service_account: Optional[ServiceAccount] = None
    oauth2: Optional[OAuth2Auth] = None       # OAuth2 / OIDC tokens

class OAuth2Auth(BaseModel):
    client_id: Optional[str] = None
    client_secret: Optional[str] = None
    auth_uri: Optional[str] = None
    auth_code: Optional[str] = None
    access_token: Optional[str] = None
    refresh_token: Optional[str] = None
    expires_at: Optional[int] = None
```

### Auth Config

```python
# src/google/adk/auth/auth_tool.py

class AuthConfig(BaseModel):
    auth_scheme: AuthScheme           # Security scheme definition (OpenAPI-style)
    raw_auth_credential: Optional[AuthCredential] = None     # Input credential
    exchanged_auth_credential: Optional[AuthCredential] = None  # After exchange
```

### The Auth Flow

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│ Tool invoked │────▶│ ToolAuthHandler  │────▶│ CredentialStore      │
│              │     │                  │     │ (in ToolContext.state)│
└─────────────┘     └──────────────────┘     └──────────────────────┘
                           │                            │
                    ┌──────▼──────────┐         Has cached credential?
                    │ Check cache     │────────── Yes ──▶ Use it
                    │                 │
                    │ No cached cred  │
                    └──────┬──────────┘
                           │
                    ┌──────▼──────────────────────────┐
                    │ AutoAuthCredentialExchanger      │
                    │                                  │
                    │ Routes to specific exchanger:    │
                    │  ├─ OAuth2CredentialExchanger    │
                    │  ├─ ServiceAccountExchanger      │
                    │  └─ Custom exchangers            │
                    └──────┬──────────────────────────┘
                           │
                    ┌──────▼──────────┐
                    │ Exchange result  │
                    │                  │
                    │ Success? ───▶ Cache & use
                    │ Need user? ──▶ Request user auth (pause tool)
                    └─────────────────┘
```

### ToolAuthHandler — Core Orchestrator

```python
# src/google/adk/tools/openapi_tool/openapi_spec_parser/tool_auth_handler.py

class ToolAuthHandler:
    async def prepare_auth_credentials(self, tool_context):
        # 1. Check credential store for cached credential
        cached = credential_store.load(credential_key)
        if cached and not expired:
            return cached

        # 2. Try refresh (OAuth2 / OIDC)
        if has_refresh_token:
            refreshed = OAuth2CredentialRefresher().refresh(credential)
            credential_store.save(refreshed)
            return refreshed

        # 3. Try exchange via AutoAuthCredentialExchanger
        exchanged = AutoAuthCredentialExchanger().exchange(auth_scheme, raw_credential)
        if exchanged:
            credential_store.save(exchanged)
            return exchanged

        # 4. Request user authorization
        tool_context.request_credential(auth_config)
        return None  # Tool returns "Pending User Authorization"
```

### How Auth Credentials Become HTTP Params

```python
# src/google/adk/tools/openapi_tool/auth/auth_helpers.py

def credential_to_param(auth_credential, auth_scheme):
    """Convert an AuthCredential to request parameters."""
    if auth_credential.auth_type == AuthCredentialTypes.API_KEY:
        if auth_scheme.in_ == APIKeyIn.header:
            return {"headers": {auth_scheme.name: auth_credential.api_key}}
        elif auth_scheme.in_ == APIKeyIn.query:
            return {"query_params": {auth_scheme.name: auth_credential.api_key}}

    elif auth_credential.auth_type == AuthCredentialTypes.HTTP:
        if auth_credential.http.scheme == "bearer":
            return {"headers": {"Authorization": f"Bearer {auth_credential.http.credentials.token}"}}
        elif auth_credential.http.scheme == "basic":
            encoded = base64.b64encode(f"{username}:{password}".encode()).decode()
            return {"headers": {"Authorization": f"Basic {encoded}"}}
```

### MCP Tool Auth

MCP tools use `BaseAuthenticatedTool` and convert credentials to HTTP headers:

```python
# src/google/adk/tools/mcp_tool/mcp_tool.py

class McpTool(BaseAuthenticatedTool):
    def _get_headers(self, credential: AuthCredential) -> dict:
        """Convert AuthCredential to MCP session headers."""
        if credential.auth_type == AuthCredentialTypes.OAUTH2:
            return {"Authorization": f"Bearer {credential.oauth2.access_token}"}
        elif credential.auth_type == AuthCredentialTypes.HTTP:
            if credential.http.scheme == "bearer":
                return {"Authorization": f"Bearer {credential.http.credentials.token}"}
            elif credential.http.scheme == "basic":
                encoded = base64.b64encode(...)
                return {"Authorization": f"Basic {encoded}"}
        elif credential.auth_type == AuthCredentialTypes.API_KEY:
            return {"x-api-key": credential.api_key}
        elif credential.auth_type == AuthCredentialTypes.SERVICE_ACCOUNT:
            token = credential.service_account.get_access_token()
            return {"Authorization": f"Bearer {token}"}
```

---

## 15. Tool Execution Flow (Runner → Agent → Tool)

Here's the complete flow from user message to tool execution:

```
User Message
    │
    ▼
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│  Runner   │────▶│    Agent     │────▶│   LLM (Gemini)   │
│ run_async │     │  run_async   │     │                  │
└──────────┘     └─────────────┘     └──────────────────┘
                                              │
                                     Function call response
                                              │
                                              ▼
                                     ┌──────────────────┐
                                     │  functions.py     │
                                     │  handle_function_ │
                                     │  calls_async()    │
                                     └──────────────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                         Tool Call #1    Tool Call #2    Tool Call #N
                      (parallel via asyncio.gather)
                              │
                              ▼
                     ┌──────────────────┐
                     │ _execute_single_ │
                     │ function_call    │
                     └──────────────────┘
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
              before_tool  tool.run   after_tool
              callbacks    _async()   callbacks
                              │
                              ▼
                     ┌──────────────────┐
                     │ FunctionResponse │
                     │ + state deltas   │
                     │ + artifact deltas│
                     └──────────────────┘
```

### Key code from `functions.py`:

```python
# src/google/adk/flows/llm_flows/functions.py

async def handle_function_calls_async(invocation_context, event):
    """Process all function calls from LLM response in parallel."""
    tasks = [
        _execute_single_function_call_async(invocation_context, fc, tool)
        for fc, tool in function_calls
    ]
    results = await asyncio.gather(*tasks)
    return merge_results_into_event(results)

async def _execute_single_function_call_async(invocation_context, fc, tool):
    # 1. Create ToolContext
    tool_context = ToolContext(
        invocation_context=invocation_context,
        function_call_id=fc.id,
    )

    # 2. Run before_tool_callbacks
    for callback in callbacks:
        result = await callback(tool=tool, args=fc.args, tool_context=tool_context)
        if result is not None:
            return result  # Callback intercepted

    # 3. Execute the tool
    result = await tool.run_async(args=fc.args, tool_context=tool_context)

    # 4. Run after_tool_callbacks
    for callback in callbacks:
        modified = await callback(tool=tool, args=fc.args, tool_context=tool_context, result=result)
        if modified is not None:
            result = modified

    # 5. Build FunctionResponse with state/artifact deltas
    return FunctionResponse(name=tool.name, response=result, actions=tool_context.actions)
```

**Important:** All function calls from a single LLM response execute
**concurrently** via `asyncio.gather()`.

---

## 16. ToolContext — What Tools Receive at Runtime

`ToolContext` is the runtime context passed to every `tool.run_async()` call. It
provides access to the entire invocation environment:

```python
# Key properties and methods of ToolContext:

# Session & Identity
tool_context.session          # Current Session object
tool_context.user_id          # User identifier
tool_context.agent_name       # Running agent's name
tool_context.invocation_id    # Current invocation ID
tool_context.function_call_id # ID of this specific tool call

# State Management (read/write)
tool_context.state['key'] = 'value'   # Modify session state
value = tool_context.state['key']     # Read session state

# Artifacts
await tool_context.save_artifact('file.txt', artifact)
artifact = await tool_context.load_artifact('file.txt', version=1)

# Memory
results = await tool_context.search_memory('query about past conversations')

# Authentication
await tool_context.save_credential(auth_config)
credential = await tool_context.load_credential(auth_config)
await tool_context.request_credential(auth_config)  # Ask user for auth

# User Interaction
await tool_context.request_confirmation(message="Are you sure?",
    tool_confirmation=ToolConfirmation(decision='accept'))

# Actions (collected by the framework)
tool_context.actions.escalate = True           # Exit loop / transfer
tool_context.actions.skip_summarization = True  # Skip summarization
tool_context.actions.transfer_to_agent = 'other_agent'  # Transfer
```

---

## 17. Toolsets and Filtering

Toolsets group multiple tools and support two filtering mechanisms:

### Name-based filtering

```python
toolset = McpToolset(
    connection_params=...,
    tool_filter=['read_file', 'list_directory']  # Only expose these tools
)
```

### Predicate-based filtering (dynamic)

```python
def my_filter(tool: BaseTool, readonly_context: ReadonlyContext = None) -> bool:
    """Only allow tools if user has premium access."""
    if readonly_context and readonly_context.state.get('is_premium'):
        return True
    return tool.name in ['basic_tool_1', 'basic_tool_2']

toolset = McpToolset(
    connection_params=...,
    tool_filter=my_filter
)
```

### Name prefixing

```python
toolset = McpToolset(
    connection_params=...,
    tool_name_prefix='fs_'  # All tools become fs_read_file, fs_list_directory, etc.
)
```

---

## 18. Summary & Key Takeaways

### Are tools calling MCP servers?

**Yes** — `McpToolset` / `McpTool` connect to MCP servers via three transport
protocols (stdio, SSE, streamable HTTP). Each MCP server tool is wrapped as an
individual `McpTool` that calls the MCP server's `tools/call` endpoint. Session
pooling, auto-retry, and OpenTelemetry propagation are built in.

### Are tools using skills?

**Yes, but separately** — `SkillToolset` provides skill management via three
sub-tools (`list_skills`, `load_skill`, `load_skill_resource`). Skills are
local markdown instruction folders, not MCP servers. The skill content is
injected into the LLM's system instructions at runtime.

### How is auth handled?

Authentication is a layered, pluggable system:

1. **`AuthConfig`** defines what auth is needed (scheme + credentials).
2. **`ToolAuthHandler`** orchestrates the credential lifecycle (cache → refresh
   → exchange → request user input).
3. **`AutoAuthCredentialExchanger`** routes to type-specific exchangers (OAuth2,
   service account, etc.).
4. **Credential caching** is handled in `ToolContext.state` via
   `ToolContextCredentialStore`.
5. **For MCP tools**, credentials are converted to HTTP headers.
6. **For REST API tools**, credentials become headers or query parameters per the
   OpenAPI security scheme.
7. **For database tools**, Google Cloud credentials are injected via
   `GoogleCredentialsManager`.

### Other important aspects

- **Parallel execution**: All tool calls from a single LLM response run
  concurrently via `asyncio.gather()`.
- **process_llm_request()**: Some tools (search, grounding, memory, examples)
  modify the LLM request *before* it's sent rather than being function-called.
- **Third-party adapters**: LangChain and CrewAI tools are wrapped as
  `FunctionTool` subclasses.
- **Lazy loading**: The tools `__init__.py` uses lazy imports to avoid pulling
  in heavy dependencies until needed.
- **Confirmation flow**: Tools can require user confirmation before executing via
  `require_confirmation` or `tool_context.request_confirmation()`.
- **OpenTelemetry**: MCP tools propagate trace context to MCP servers for
  distributed tracing.
