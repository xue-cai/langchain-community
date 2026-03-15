# LangChain Community Tools — Technical Analysis

This document provides a technical analysis of the tools in the `langchain-community` repository, covering architecture, execution flow, authentication, security mechanisms, and key design patterns.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [How Tools Work](#how-tools-work)
- [Tool Categories](#tool-categories)
- [Authentication Patterns](#authentication-patterns)
- [Security Mechanisms](#security-mechanisms)
- [MCP Servers and Skills](#mcp-servers-and-skills)
- [Key Design Patterns](#key-design-patterns)
- [LangChain vs MCP: Side-by-Side Comparison](#langchain-vs-mcp-side-by-side-comparison)

---

## Overview

The `langchain-community` package contains **60+ tool integrations** located in
`libs/community/langchain_community/tools/`. Each tool is a Python class that an
LLM-powered Agent can invoke to interact with external services, APIs, or the
local operating system.

**Tools do NOT call MCP servers or use "skills."** They are plain Python classes
that inherit from `BaseTool` (defined in `langchain-core`) and directly wrap
external APIs, SDKs, or system operations. The `.mcp.json` at the repository
root is an IDE helper for documentation lookups — it is not used by any tool at
runtime.

---

## Architecture

### Class Hierarchy

```
langchain_core.tools.BaseTool        # abstract base (defined in langchain-core)
    ├── <Name>Tool                   # e.g., TavilySearchResults, ShellTool
    ├── BaseBrowserTool              # shared base for Playwright browser tools
    │       └── ClickTool, NavigateTool, ...
    ├── GmailBaseTool                # shared base for Gmail tools
    │       └── GmailSearch, GmailSendMessage, ...
    └── BaseRequestsTool + BaseTool  # mixin for HTTP request tools
            └── RequestsGetTool, RequestsPostTool, ...
```

### File Organization

Each tool lives under `libs/community/langchain_community/tools/<tool_name>/` with a consistent structure:

```
tools/
├── tavily_search/
│   ├── __init__.py          # re-exports tool classes
│   └── tool.py              # TavilySearchResults, TavilyAnswer classes
├── gmail/
│   ├── __init__.py
│   ├── base.py              # GmailBaseTool (shared base class)
│   ├── utils.py             # OAuth credential helpers
│   ├── search.py            # GmailSearch tool
│   ├── get_message.py       # GmailGetMessage tool
│   └── send_message.py      # GmailSendMessage tool
├── requests/
│   ├── __init__.py
│   └── tool.py              # RequestsGetTool, RequestsPostTool, etc.
└── ...
```

### Lazy Loading

The top-level `tools/__init__.py` uses **lazy importing** to avoid loading all
60+ integrations and their dependencies at import time:

```python
# tools/__init__.py (simplified)
_module_lookup = {
    "TavilySearchResults": "langchain_community.tools.tavily_search.tool",
    "BraveSearch":         "langchain_community.tools.brave_search.tool",
    "ShellTool":           "langchain_community.tools.shell.tool",
    # ... 150+ entries
}

def __getattr__(name: str) -> Any:
    if name in _module_lookup:
        module = importlib.import_module(_module_lookup[name])
        return getattr(module, name)
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

This means `from langchain_community.tools import TavilySearchResults` only
loads the Tavily module — not every tool in the package.

---

## How Tools Work

### Execution Flow

```
User / Agent
    │
    ▼
Tool.invoke({"query": "..."})          # public entry point (from BaseTool)
    │
    ▼
BaseTool._run() / _arun()             # abstract method implemented by each tool
    │
    ▼
API Wrapper (utility class)            # e.g., TavilySearchAPIWrapper
    │
    ▼
External API / SDK / OS               # HTTP call, SDK method, subprocess, etc.
    │
    ▼
Formatted response                     # returned to the Agent as string or dict
```

### The `BaseTool` Contract

Every tool must define:

| Attribute / Method | Purpose |
|---|---|
| `name: str` | Unique identifier the Agent uses to select the tool |
| `description: str` | Natural-language description the LLM reads to decide when to use the tool |
| `args_schema: Type[BaseModel]` | Pydantic model that validates input arguments |
| `_run(...)` | Synchronous execution logic |
| `_arun(...)` | *(Optional)* Asynchronous execution logic |

### Concrete Example — Tavily Search

The `TavilySearchResults` tool demonstrates the standard pattern:

```python
# libs/community/langchain_community/tools/tavily_search/tool.py

class TavilyInput(BaseModel):
    """Pydantic schema — validates the input the Agent sends."""
    query: str = Field(description="search query to look up")


class TavilySearchResults(BaseTool):
    # 1. Identity — the Agent reads these to decide when to use the tool
    name: str = "tavily_search_results_json"
    description: str = (
        "A search engine optimized for comprehensive, accurate, and trusted results. "
        "Useful for when you need to answer questions about current events. "
        "Input should be a search query."
    )
    args_schema: Type[BaseModel] = TavilyInput

    # 2. Configuration
    max_results: int = 5
    search_depth: str = "advanced"
    include_domains: List[str] = []
    # ... more options

    # 3. API wrapper — handles HTTP calls and auth
    api_wrapper: TavilySearchAPIWrapper = Field(
        default_factory=TavilySearchAPIWrapper
    )
    response_format: Literal["content_and_artifact"] = "content_and_artifact"

    # 4. Synchronous execution
    def _run(
        self,
        query: str,
        run_manager: Optional[CallbackManagerForToolRun] = None,
    ) -> Tuple[Union[List[Dict[str, str]], str], Dict]:
        try:
            raw_results = self.api_wrapper.raw_results(
                query,
                self.max_results,
                self.search_depth,
                self.include_domains,
                self.exclude_domains,
                self.include_answer,
                self.include_raw_content,
                self.include_images,
            )
        except Exception as e:
            return repr(e), {}
        return self.api_wrapper.clean_results(raw_results["results"]), raw_results

    # 5. Asynchronous execution
    async def _arun(
        self,
        query: str,
        run_manager: Optional[AsyncCallbackManagerForToolRun] = None,
    ) -> Tuple[Union[List[Dict[str, str]], str], Dict]:
        try:
            raw_results = await self.api_wrapper.raw_results_async(
                query, self.max_results, ...
            )
        except Exception as e:
            return repr(e), {}
        return self.api_wrapper.clean_results(raw_results["results"]), raw_results
```

**Key takeaways:**

- `_run()` delegates all HTTP work to `self.api_wrapper`.
- The method returns a `(content, artifact)` tuple so the Agent gets a
  human-readable summary *and* the full raw API response.
- Error handling catches exceptions and returns them as strings rather than
  crashing the Agent loop.

---

## Tool Categories

| Category | Examples | Mechanism |
|---|---|---|
| **Search engines** | Tavily, Brave, Bing, DuckDuckGo, Google, Mojeek, SearX | REST API via utility wrapper |
| **Productivity / SaaS** | Gmail, Slack, Office365, Jira, GitHub, GitLab, Clickup | OAuth or API key via SDK |
| **Database** | SQL Database, Spark SQL, Cassandra | Direct DB connection via driver |
| **HTTP / API** | Requests (GET/POST/PUT/PATCH/DELETE), OpenAPI, GraphQL | Raw HTTP via `requests` / `aiohttp` |
| **Browser automation** | Playwright (Click, Navigate, GetElements, Screenshot) | Playwright SDK (browser instance) |
| **System / Shell** | ShellTool, File Management (read/write/copy/move/delete) | `subprocess` / filesystem ops |
| **AI / ML services** | Azure AI, EdenAI, OpenAI DALL-E, Eleven Labs | REST API with API key |
| **Knowledge bases** | Arxiv, PubMed, Wikipedia, Wikidata, Wolfram Alpha | Public APIs |
| **Financial** | Polygon, Yahoo Finance, Alpha Vantage, Google Finance | REST API with API key |

---

## Authentication Patterns

Tools use **four distinct authentication strategies**, depending on the external
service they integrate with.

### 1. API Key via Environment Variable (most common)

The API key is read from an environment variable at instantiation time using
the `get_from_dict_or_env` helper. The key is stored as a Pydantic `SecretStr`
to prevent accidental logging.

```python
# libs/community/langchain_community/utilities/tavily_search.py

class TavilySearchAPIWrapper(BaseModel):
    tavily_api_key: SecretStr    # stored securely

    @model_validator(mode="before")
    @classmethod
    def validate_environment(cls, values: Dict) -> Any:
        # Reads from constructor arg OR falls back to TAVILY_API_KEY env var
        tavily_api_key = get_from_dict_or_env(
            values, "tavily_api_key", "TAVILY_API_KEY"
        )
        values["tavily_api_key"] = tavily_api_key
        return values

    def raw_results(self, query: str, ...) -> Dict:
        params = {
            "api_key": self.tavily_api_key.get_secret_value(),  # unwrap only when needed
            ...
        }
        response = requests.post(f"{TAVILY_API_URL}/search", json=params)
        ...
```

**Used by:** Tavily, Brave Search, Bing, DataForSEO, Wolfram Alpha, and many
others.

### 2. API Key via Constructor Parameter

Some tools accept the key directly and wrap it in `SecretStr`:

```python
# libs/community/langchain_community/tools/brave_search/tool.py

class BraveSearch(BaseTool):
    search_wrapper: BraveSearchWrapper = Field(default_factory=BraveSearchWrapper)

    @classmethod
    def from_api_key(cls, api_key: str, search_kwargs: Optional[dict] = None, **kwargs):
        wrapper = BraveSearchWrapper(
            api_key=SecretStr(api_key),    # explicitly wrapped
            search_kwargs=search_kwargs or {}
        )
        return cls(search_wrapper=wrapper, **kwargs)
```

### 3. OAuth 2.0 Flow (Google services)

Gmail and Google Cloud tools use a full OAuth flow with token persistence:

```python
# libs/community/langchain_community/tools/gmail/utils.py

DEFAULT_SCOPES = ["https://mail.google.com/"]
DEFAULT_CREDS_TOKEN_FILE = "token.json"
DEFAULT_CLIENT_SECRETS_FILE = "credentials.json"

def get_gmail_credentials(
    token_file: Optional[str] = None,
    client_secrets_file: Optional[str] = None,
    scopes: Optional[List[str]] = None,
) -> Credentials:
    Request, Credentials = import_google()
    InstalledAppFlow = import_installed_app_flow()
    creds = None
    scopes = scopes or DEFAULT_SCOPES
    token_file = token_file or DEFAULT_CREDS_TOKEN_FILE
    client_secrets_file = client_secrets_file or DEFAULT_CLIENT_SECRETS_FILE
    # Loads saved token.json if available, otherwise runs browser-based OAuth flow
    ...
```

The tool base class then injects the authenticated API resource:

```python
# libs/community/langchain_community/tools/gmail/base.py

class GmailBaseTool(BaseTool):
    api_resource: Resource = Field(default_factory=build_resource_service)
```

### 4. No Authentication (public APIs / local operations)

Tools like Wikipedia, Arxiv, Shell, and File Management do not require any
authentication — they access public APIs or local system resources directly.

---

## Security Mechanisms

### Dangerous Request Gate

HTTP request tools require an explicit opt-in flag to prevent accidental misuse
(e.g., Server-Side Request Forgery):

```python
# libs/community/langchain_community/tools/requests/tool.py

class BaseRequestsTool(BaseModel):
    requests_wrapper: GenericRequestsWrapper
    allow_dangerous_requests: bool = False

    def __init__(self, **kwargs: Any):
        if not kwargs.get("allow_dangerous_requests", False):
            raise ValueError(
                "You must set allow_dangerous_requests to True to use this tool. "
                "Requests can be dangerous and can lead to security vulnerabilities. "
                "For example, users can ask a server to make a request to an internal "
                "server. It's recommended to use requests through a proxy server "
                "and avoid accepting inputs from untrusted sources without proper "
                "sandboxing. "
                "Please see: https://python.langchain.com/docs/security"
            )
        super().__init__(**kwargs)
```

### Shell Command Warnings

The `ShellTool` emits a runtime warning before executing any command:

```python
# libs/community/langchain_community/tools/shell/tool.py

class ShellInput(BaseModel):
    commands: Union[str, List[str]]

    @model_validator(mode="before")
    @classmethod
    def _validate_commands(cls, values: dict) -> Any:
        warnings.warn(
            "The shell tool has no safeguards by default. Use at your own risk."
        )
        return values
```

It also supports an interactive confirmation mode:

```python
class ShellTool(BaseTool):
    ask_human_input: bool = False

    def _run(self, commands, ...):
        if self.ask_human_input:
            user_input = input("Proceed with command execution? (y/n): ").lower()
            if user_input == "y":
                return self.process.run(commands)
            ...
```

### Browser Tool Validation

Playwright tools require an explicit browser instance — they cannot create one
silently:

```python
# libs/community/langchain_community/tools/playwright/base.py

class BaseBrowserTool(BaseTool):
    sync_browser: Optional[SyncBrowser] = None
    async_browser: Optional[AsyncBrowser] = None

    @model_validator(mode="before")
    @classmethod
    def validate_browser_provided(cls, values: dict) -> Any:
        if values.get("async_browser") is None and values.get("sync_browser") is None:
            raise ValueError("Either async_browser or sync_browser must be specified.")
        return values
```

### SecretStr for API Keys

Utility wrappers store secrets as Pydantic `SecretStr` objects, which prevent
the key from appearing in `repr()`, `str()`, or serialized output:

```python
class TavilySearchAPIWrapper(BaseModel):
    tavily_api_key: SecretStr    # never accidentally printed

    def raw_results(self, ...):
        params = {
            "api_key": self.tavily_api_key.get_secret_value(),  # explicit unwrap
            ...
        }
```

---

## MCP Servers and Skills

**The tools in this repository do NOT use MCP (Model Context Protocol) servers
or "skills" at runtime.**

The repository root contains a `.mcp.json` file:

```json
{
  "mcpServers": {
    "docs-langchain": {
      "type": "http",
      "url": "https://docs.langchain.com/mcp"
    }
  }
}
```

This file is an **IDE/editor configuration** for tools like VS Code Copilot
that support MCP-based documentation lookup. It is not referenced by any Python
code in the repository and has no effect on tool behavior.

Each tool is a self-contained Python class that directly wraps an external API
or SDK — there is no intermediate MCP protocol, skill registry, or plugin
system involved.

---

## Key Design Patterns

### 1. Wrapper Utilities (Separation of Concerns)

Tools delegate API interaction to a dedicated utility wrapper class in
`langchain_community/utilities/`. This separates *tool identity* (name,
description, schema) from *API logic* (HTTP calls, auth, response parsing).

```
Tool class                          Utility wrapper
────────────                        ────────────────
TavilySearchResults  ──────────►    TavilySearchAPIWrapper
BraveSearch          ──────────►    BraveSearchWrapper
RequestsGetTool      ──────────►    GenericRequestsWrapper
GmailSearch          ──────────►    Google API Resource (via build_resource_service)
```

### 2. Factory Defaults with `Field(default_factory=...)`

Wrappers are lazily instantiated so that environment variables are read at tool
creation time, not at module import time:

```python
api_wrapper: TavilySearchAPIWrapper = Field(default_factory=TavilySearchAPIWrapper)
```

### 3. Input Validation with Pydantic `args_schema`

Every tool defines a Pydantic model that validates and documents the expected
input. This schema is also used by LLMs to generate correctly-formatted tool
calls:

```python
class TavilyInput(BaseModel):
    query: str = Field(description="search query to look up")

class TavilySearchResults(BaseTool):
    args_schema: Type[BaseModel] = TavilyInput
```

### 4. Content and Artifact Response Format

Several tools return a `(content, artifact)` tuple, allowing the Agent to
receive both a summary and the full raw response:

```python
response_format: Literal["content_and_artifact"] = "content_and_artifact"

def _run(self, query: str, ...) -> Tuple[Union[List[Dict], str], Dict]:
    ...
    return self.api_wrapper.clean_results(raw["results"]), raw
```

### 5. Graceful Error Handling

Tools catch exceptions internally and return error strings instead of raising,
so the Agent loop can continue:

```python
def _run(self, query: str, ...):
    try:
        raw_results = self.api_wrapper.raw_results(query, ...)
    except Exception as e:
        return repr(e), {}
    return self.api_wrapper.clean_results(raw_results["results"]), raw_results
```

### 6. Async Support

Most tools provide both `_run()` and `_arun()` methods for synchronous and
asynchronous execution:

```python
def _run(self, query: str, ...):
    return self.search_wrapper.run(query)

async def _arun(self, query: str, ...):
    return await self.search_wrapper.arun(query)
```

### 7. Class-Based (Not Decorator-Based) Tools

All community tools use the class-based `BaseTool` pattern. The `@tool`
function decorator (from `langchain_core`) is available but not used in this
package — the class approach provides richer configuration, validation, and
inheritance.

---

## LangChain vs MCP: Side-by-Side Comparison

This section compares how **LangChain tools** and **MCP (Model Context Protocol)
tools** define their shape, expose it to an LLM, and participate in a
multi-turn conversation. Both approaches solve the same problem — *telling the
LLM what tools are available and how to call them* — but they do it in
fundamentally different ways.

### 1. Defining a Tool

#### LangChain — Python class with Pydantic schema

The tool shape is defined **statically at code-writing time** as class
attributes on a `BaseTool` subclass:

```python
# langchain_community/tools/tavily_search/tool.py

from pydantic import BaseModel, Field
from langchain_core.tools import BaseTool

class TavilyInput(BaseModel):
    """Pydantic model — defines the tool's input parameters."""
    query: str = Field(description="search query to look up")

class TavilySearchResults(BaseTool):
    name: str = "tavily_search_results_json"
    description: str = (
        "A search engine optimized for comprehensive, accurate, and trusted results. "
        "Useful for when you need to answer questions about current events. "
        "Input should be a search query."
    )
    args_schema: Type[BaseModel] = TavilyInput

    def _run(self, query: str, **kwargs) -> str:
        # Delegates to the API wrapper (HTTP calls, auth, etc.)
        return self.api_wrapper.raw_results(query, self.max_results)
```

The key pieces — `name`, `description`, and `args_schema` — are baked into the
Python class definition.

#### MCP — JSON-RPC `tools/list` response from a remote server

In MCP, the tool shape is defined on a **remote MCP server** and discovered
at runtime via a `tools/list` JSON-RPC call:

```json
// Client sends:
{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}

// MCP server responds:
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "tavily_search",
        "description": "A search engine optimized for comprehensive, accurate results.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "search query to look up"
            }
          },
          "required": ["query"]
        }
      }
    ]
  }
}
```

The tool name, description, and input schema are returned as **JSON Schema**
by the server, not defined in the client's code.

### 2. Telling the LLM About the Tools

Both approaches ultimately produce the same thing: a JSON object that the LLM
API understands. The difference is *where* that JSON comes from.

#### LangChain — `bind_tools()` converts Pydantic to JSON at call time

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools import TavilySearchResults

# 1. Instantiate the tool (shape is in the Python class)
tool = TavilySearchResults()

# 2. Bind to LLM — internally converts Pydantic → JSON Schema → OpenAI format
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([tool])
```

Under the hood, `bind_tools` calls
`langchain_core.utils.function_calling.convert_to_openai_tool()` which:

1. Reads `tool.name` and `tool.description`
2. Calls `tool.tool_call_schema` → gets the Pydantic model
3. Calls `.model_json_schema()` on the Pydantic model → produces JSON Schema
4. Wraps it in the OpenAI function-calling format

The resulting JSON sent to the LLM API looks like:

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "tavily_search_results_json",
        "description": "A search engine optimized for comprehensive...",
        "parameters": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "search query to look up"
            }
          },
          "required": ["query"]
        }
      }
    }
  ]
}
```

#### MCP — Client forwards `tools/list` result to LLM

An MCP client first discovers available tools, then passes them to the LLM:

```python
# 1. Connect to MCP server and discover tools at runtime
tools_response = await mcp_client.request("tools/list")
mcp_tools = tools_response["tools"]

# 2. Convert MCP tool schema to OpenAI format
openai_tools = [
    {
        "type": "function",
        "function": {
            "name": t["name"],
            "description": t["description"],
            "parameters": t["inputSchema"],
        },
    }
    for t in mcp_tools
]

# 3. Send to LLM
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=openai_tools,
)
```

The JSON sent to the LLM is **structurally identical** to the LangChain case —
the difference is that the schema came from a remote server, not from a local
Python class.

### 3. Full Conversation Example

The following example shows a user asking "What is the latest news on AI?" and
the system using a search tool to answer. Both approaches follow the same
conceptual loop:

```
User message → LLM decides to call tool → Tool executes → Result returned → LLM answers
```

#### LangChain Conversation

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools import TavilySearchResults
from langchain_core.messages import HumanMessage, ToolMessage

# Setup
tool = TavilySearchResults()
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([tool])

# Step 1: User asks a question
messages = [HumanMessage(content="What is the latest news on AI?")]

# Step 2: LLM responds with a tool call (not a text answer)
ai_message = llm_with_tools.invoke(messages)
# ai_message.tool_calls == [
#     {"name": "tavily_search_results_json",
#      "args": {"query": "latest news on AI"},
#      "id": "call_abc123"}
# ]

# Step 3: Execute the tool
tool_result = tool.invoke(ai_message.tool_calls[0]["args"])

# Step 4: Append tool call + result to messages
messages.append(ai_message)
messages.append(
    ToolMessage(content=str(tool_result), tool_call_id="call_abc123")
)

# Step 5: LLM generates final answer using the tool result
final_response = llm_with_tools.invoke(messages)
print(final_response.content)
# "Here are the latest developments in AI: ..."
```

**What happened under the hood:**

| Step | HTTP Request to OpenAI |
|------|----------------------|
| 2 | `POST /chat/completions` with `tools` parameter (JSON schema from Pydantic) and user message |
| 5 | `POST /chat/completions` with user message + assistant tool_call + tool result |

#### MCP Conversation

```python
import json
import openai

# Setup — connect to an MCP server
mcp_tools = await mcp_client.request("tools/list")  # runtime discovery

openai_tools = [
    {
        "type": "function",
        "function": {
            "name": t["name"],
            "description": t["description"],
            "parameters": t["inputSchema"],
        },
    }
    for t in mcp_tools["tools"]
]

client = openai.OpenAI()

# Step 1: User asks a question
messages = [{"role": "user", "content": "What is the latest news on AI?"}]

# Step 2: LLM responds with a tool call
response = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=openai_tools
)
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name == "tavily_search"
# tool_call.function.arguments == '{"query": "latest news on AI"}'

# Step 3: Execute the tool via MCP server (JSON-RPC call)
tool_result = await mcp_client.request(
    "tools/call",
    {"name": tool_call.function.name, "arguments": json.loads(tool_call.function.arguments)},
)

# Step 4: Append tool call + result to messages
messages.append(response.choices[0].message)
messages.append(
    {"role": "tool", "tool_call_id": tool_call.id, "content": str(tool_result)}
)

# Step 5: LLM generates final answer
final = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=openai_tools
)
print(final.choices[0].message.content)
# "Here are the latest developments in AI: ..."
```

**What happened under the hood:**

| Step | What Happens |
|------|-------------|
| Setup | JSON-RPC `tools/list` to MCP server → get tool schemas |
| 2 | `POST /chat/completions` to OpenAI with `tools` (from MCP) and user message |
| 3 | JSON-RPC `tools/call` to MCP server → execute tool remotely |
| 5 | `POST /chat/completions` to OpenAI with full conversation |

### 4. Summary of Key Differences

| Aspect | LangChain | MCP |
|--------|-----------|-----|
| **Where tool shape is defined** | Python class attributes (`name`, `description`, `args_schema`) | Remote MCP server, returned by `tools/list` JSON-RPC |
| **When shape is known** | Definition time (class/module load) | Runtime (server discovery) |
| **Schema format** | Pydantic `BaseModel` → converted to JSON Schema | JSON Schema directly from server |
| **How LLM sees tools** | `bind_tools()` → `convert_to_openai_tool()` → JSON | Client maps `tools/list` response → JSON |
| **Tool execution** | Local Python: `tool.invoke(args)` → `_run()` → API wrapper | Remote JSON-RPC: `tools/call` to MCP server |
| **Transport** | In-process function call | JSON-RPC over stdio / HTTP / SSE |
| **Adding a new tool** | Write a Python class, install the package | Deploy/configure an MCP server |
| **What the LLM API receives** | Identical JSON format | Identical JSON format |

### 5. Key Insight

Despite the different architectures, both approaches converge to the **same
JSON payload** sent to the LLM API. The LLM doesn't know or care whether the
tool schema came from a Pydantic class or an MCP server — it just sees a
`tools` array of JSON Schema objects.

The fundamental difference is:

- **LangChain:** Tool shape is *code* (Python classes). Discovery is implicit —
  the developer passes `tools=[...]` to the agent.
- **MCP:** Tool shape is *data* (JSON). Discovery is explicit — the client asks
  the server "what tools do you have?" at runtime.

This means LangChain tools are tightly coupled to the Python process (you must
install the package that contains the tool), while MCP tools can live on a
separate server and be discovered dynamically by any client that speaks the
MCP protocol.
