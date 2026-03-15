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
multi-turn conversation. The examples use two real tools from this repository —
**Bing Search** (a simple single-input tool) and **Gmail Send Message** (a
multi-input tool with optional parameters) — to illustrate the patterns.

Both approaches solve the same problem — *telling the LLM what tools are
available and how to call them* — but they do it in fundamentally different
ways.

### 1. Defining a Tool

#### LangChain — Python class with Pydantic schema

Tool shapes are defined **statically at code-writing time** as class attributes
on a `BaseTool` subclass.

**Example A — Bing Search** (simple, single input):

```python
# langchain_community/tools/bing_search/tool.py

class BingSearchResults(BaseTool):
    name: str = "bing_search_results_json"
    description: str = (
        "A wrapper around Bing Search. "
        "Useful for when you need to answer questions about current events. "
        "Input should be a search query. Output is an array of the query results."
    )
    num_results: int = 4
    api_wrapper: BingSearchAPIWrapper
    response_format: Literal["content_and_artifact"] = "content_and_artifact"

    def _run(
        self,
        query: str,
        run_manager: Optional[CallbackManagerForToolRun] = None,
    ) -> Tuple[str, List[Dict]]:
        """Use the tool."""
        try:
            results = self.api_wrapper.results(query, self.num_results)
            return str(results), results
        except Exception as e:
            return repr(e), []
```

`BingSearchResults` has no explicit `args_schema` — `BaseTool` infers a
schema with a single `query: str` parameter from the `_run` signature.

**Example B — Gmail Send Message** (multi-input, with optional fields):

```python
# langchain_community/tools/gmail/send_message.py

class SendMessageSchema(BaseModel):
    """Input for SendMessageTool."""
    message: str = Field(..., description="The message to send.")
    to: Union[str, List[str]] = Field(..., description="The list of recipients.")
    subject: str = Field(..., description="The subject of the message.")
    cc: Optional[Union[str, List[str]]] = Field(
        None, description="The list of CC recipients."
    )
    bcc: Optional[Union[str, List[str]]] = Field(
        None, description="The list of BCC recipients."
    )


class GmailSendMessage(GmailBaseTool):
    name: str = "send_gmail_message"
    description: str = (
        "Use this tool to send email messages. The input is the message, recipients"
    )
    args_schema: Type[SendMessageSchema] = SendMessageSchema

    def _run(
        self,
        message: str,
        to: Union[str, List[str]],
        subject: str,
        cc: Optional[Union[str, List[str]]] = None,
        bcc: Optional[Union[str, List[str]]] = None,
        run_manager: Optional[CallbackManagerForToolRun] = None,
    ) -> str:
        """Run the tool."""
        try:
            create_message = self._prepare_message(message, to, subject, cc=cc, bcc=bcc)
            send_message = (
                self.api_resource.users()
                .messages()
                .send(userId="me", body=create_message)
            )
            sent_message = send_message.execute()
            return f"Message sent. Message Id: {sent_message['id']}"
        except Exception as error:
            raise Exception(f"An error occurred: {error}")
```

`GmailSendMessage` defines an explicit `args_schema` with required *and*
optional fields — the LLM sees all five parameters and their descriptions.

#### MCP — JSON-RPC `tools/list` response from a remote server

In MCP, tool shapes are defined on a **remote MCP server** and discovered at
runtime via a `tools/list` JSON-RPC call. Below is the equivalent of the two
tools above as an MCP server would describe them:

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
        "name": "bing_search",
        "description": "Search the web using Bing. Input should be a search query.",
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
      },
      {
        "name": "send_gmail_message",
        "description": "Send an email message via Gmail.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "message": { "type": "string", "description": "The message to send." },
            "to":      { "type": "string", "description": "Recipient email address." },
            "subject": { "type": "string", "description": "The subject of the message." },
            "cc":      { "type": "string", "description": "CC recipients (optional)." },
            "bcc":     { "type": "string", "description": "BCC recipients (optional)." }
          },
          "required": ["message", "to", "subject"]
        }
      }
    ]
  }
}
```

The tool names, descriptions, and input schemas are returned as **JSON Schema**
by the server — not defined in the client's code.

### 2. Telling the LLM About the Tools

Both approaches ultimately produce the same thing: a JSON `tools` array that
the LLM API understands. The difference is *where* that JSON comes from.

#### LangChain — `bind_tools()` converts Pydantic to JSON at call time

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools.bing_search import BingSearchResults
from langchain_community.tools.gmail import GmailSendMessage
from langchain_community.utilities import BingSearchAPIWrapper

# 1. Instantiate the tools (shapes are in the Python classes)
search = BingSearchResults(api_wrapper=BingSearchAPIWrapper())
gmail = GmailSendMessage()

# 2. Bind to LLM — internally converts Pydantic → JSON Schema → OpenAI format
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([search, gmail])
```

Under the hood, `bind_tools` calls
`langchain_core.utils.function_calling.convert_to_openai_tool()` for each
tool, which:

1. Reads `tool.name` and `tool.description`
2. Calls `tool.tool_call_schema` → gets the Pydantic model
3. Calls `.model_json_schema()` → produces JSON Schema
4. Wraps it in the OpenAI function-calling format

The resulting JSON sent to the LLM API looks like:

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "bing_search_results_json",
        "description": "A wrapper around Bing Search. Useful for when you need to answer questions about current events...",
        "parameters": {
          "type": "object",
          "properties": {
            "query": { "type": "string" }
          },
          "required": ["query"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "send_gmail_message",
        "description": "Use this tool to send email messages...",
        "parameters": {
          "type": "object",
          "properties": {
            "message": { "type": "string", "description": "The message to send." },
            "to":      { "type": "string", "description": "The list of recipients." },
            "subject": { "type": "string", "description": "The subject of the message." },
            "cc":      { "type": "string", "description": "The list of CC recipients." },
            "bcc":     { "type": "string", "description": "The list of BCC recipients." }
          },
          "required": ["message", "to", "subject"]
        }
      }
    }
  ]
}
```

#### MCP — Client forwards `tools/list` result to LLM

An MCP client first discovers available tools, then maps them to the LLM's
format:

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

The following example shows a **multi-tool conversation** where the user asks
the agent to search for a topic and email the results to a colleague. The LLM
must decide to call *two* tools in sequence. Both approaches follow the same
conceptual loop:

```
User message → LLM calls bing_search → Result returned
            → LLM calls send_gmail_message → Confirmation returned
            → LLM answers user
```

#### LangChain Conversation

```python
from langchain_openai import ChatOpenAI
from langchain_community.tools.bing_search import BingSearchResults
from langchain_community.tools.gmail import GmailSendMessage
from langchain_community.utilities import BingSearchAPIWrapper
from langchain_core.messages import HumanMessage, ToolMessage

# Setup
search = BingSearchResults(api_wrapper=BingSearchAPIWrapper())
gmail = GmailSendMessage()
tools = {t.name: t for t in [search, gmail]}

llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([search, gmail])

# Step 1: User asks a question that requires both tools
messages = [HumanMessage(
    content="Search Bing for the latest AI news and email a summary to alice@example.com"
)]

# Step 2: LLM responds with a tool call — bing_search first
ai_msg = llm_with_tools.invoke(messages)
# ai_msg.tool_calls == [
#     {"name": "bing_search_results_json",
#      "args": {"query": "latest AI news 2026"},
#      "id": "call_abc123"}
# ]

# Step 3: Execute the search tool
tool_call = ai_msg.tool_calls[0]
result = tools[tool_call["name"]].invoke(tool_call["args"])

# Step 4: Append tool call + result to messages
messages.append(ai_msg)
messages.append(ToolMessage(content=str(result), tool_call_id=tool_call["id"]))

# Step 5: LLM now calls the gmail tool with the search results
ai_msg2 = llm_with_tools.invoke(messages)
# ai_msg2.tool_calls == [
#     {"name": "send_gmail_message",
#      "args": {"to": "alice@example.com",
#               "subject": "Latest AI News Summary",
#               "message": "<html>Here are the latest AI developments..."},
#      "id": "call_def456"}
# ]

# Step 6: Execute the gmail tool
tool_call2 = ai_msg2.tool_calls[0]
result2 = tools[tool_call2["name"]].invoke(tool_call2["args"])
# "Message sent. Message Id: 18f3a..."

# Step 7: Append and get final answer
messages.append(ai_msg2)
messages.append(ToolMessage(content=str(result2), tool_call_id=tool_call2["id"]))

final = llm_with_tools.invoke(messages)
print(final.content)
# "Done! I searched Bing for the latest AI news and emailed a summary to alice@example.com."
```

**What happened under the hood:**

| Step | HTTP Request to OpenAI |
|------|----------------------|
| 2 | `POST /chat/completions` with `tools` (both schemas) + user message → LLM returns `bing_search` call |
| 5 | `POST /chat/completions` with messages + search result → LLM returns `send_gmail_message` call |
| 7 | `POST /chat/completions` with all messages + gmail result → LLM returns text answer |

#### MCP Conversation

```python
import json
import openai

# Setup — connect to an MCP server that exposes bing_search + gmail tools
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

# Step 1: User asks a question that requires both tools
messages = [{"role": "user",
             "content": "Search Bing for the latest AI news and email a summary to alice@example.com"}]

# Step 2: LLM responds with bing_search tool call
response = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=openai_tools
)
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name == "bing_search"
# tool_call.function.arguments == '{"query": "latest AI news 2026"}'

# Step 3: Execute via MCP server (JSON-RPC call)
search_result = await mcp_client.request(
    "tools/call",
    {"name": tool_call.function.name,
     "arguments": json.loads(tool_call.function.arguments)},
)

# Step 4: Append and continue
messages.append(response.choices[0].message)
messages.append({"role": "tool", "tool_call_id": tool_call.id,
                 "content": str(search_result)})

# Step 5: LLM now calls send_gmail_message
response2 = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=openai_tools
)
tool_call2 = response2.choices[0].message.tool_calls[0]
# tool_call2.function.name == "send_gmail_message"
# tool_call2.function.arguments == '{"to": "alice@example.com", "subject": "...", "message": "..."}'

# Step 6: Execute gmail via MCP server
gmail_result = await mcp_client.request(
    "tools/call",
    {"name": tool_call2.function.name,
     "arguments": json.loads(tool_call2.function.arguments)},
)

# Step 7: Append and get final answer
messages.append(response2.choices[0].message)
messages.append({"role": "tool", "tool_call_id": tool_call2.id,
                 "content": str(gmail_result)})

final = client.chat.completions.create(
    model="gpt-4o", messages=messages, tools=openai_tools
)
print(final.choices[0].message.content)
# "Done! I searched Bing for the latest AI news and emailed a summary to alice@example.com."
```

**What happened under the hood:**

| Step | What Happens |
|------|-------------|
| Setup | JSON-RPC `tools/list` to MCP server → get both tool schemas |
| 2 | `POST /chat/completions` to OpenAI → LLM returns `bing_search` call |
| 3 | JSON-RPC `tools/call` to MCP server → execute search remotely |
| 5 | `POST /chat/completions` to OpenAI → LLM returns `send_gmail_message` call |
| 6 | JSON-RPC `tools/call` to MCP server → execute email send remotely |
| 7 | `POST /chat/completions` to OpenAI → LLM returns text answer |

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

### 6. Code Interpreter Tools

The repository also contains tools for **executing Python code** in sandboxed
environments. These are the LangChain equivalent of a "code interpreter" tool.

| Tool | Class | How it works | Status |
|------|-------|-------------|--------|
| **E2B Data Analysis** | `E2BDataAnalysisTool` | Runs Python in a cloud sandbox via the [E2B](https://e2b.dev) API. Supports file uploads and artifact downloads. | **Active** |
| **Bearly Interpreter** | `BearlyInterpreterTool` | Runs Python in a sandbox via the [Bearly](https://bearly.ai) API. Accepts uploaded files and returns stdout/stderr/file links. | **Active** |
| **Python REPL** | `PythonREPLTool` | Executes code in the local Python process (no sandbox). | **Deprecated** — removed from exports |
| **Python AST REPL** | `PythonAstREPLTool` | Same as above but parsed via AST first. | **Deprecated** — removed from exports |

#### E2B Data Analysis — Example

```python
# langchain_community/tools/e2b_data_analysis/tool.py

class E2BDataAnalysisTool(BaseTool):
    name: str = "e2b_data_analysis"
    description: str = (
        "Evaluates python code in a sandbox environment. "
        "The environment is long running and exists across multiple executions. "
        "You must send the whole script every time and print your outputs."
    )
    # ...
    def _run(self, python_code: str, ...) -> str:
        # Sends code to the E2B cloud sandbox for execution
        ...
```

#### Bearly Interpreter — Example

```python
# langchain_community/tools/bearly/tool.py

class BearlyInterpreterToolArguments(BaseModel):
    python_code: str = Field(
        ...,
        description=(
            "The pure python script to be evaluated. "
            "The contents will be in main.py. "
            "It should not be in markdown format."
        ),
    )

class BearlyInterpreterTool:
    name: str = "bearly_interpreter"
    args_schema: Type[BaseModel] = BearlyInterpreterToolArguments

    def _run(self, python_code: str) -> dict:
        # Sends code to Bearly cloud sandbox, returns stdout/stderr/file links
        ...
```

Both tools follow the same `BaseTool` pattern as Bing Search and Gmail — they
define `name`, `description`, `args_schema`, and `_run()`, so they can be
bound to an LLM with `bind_tools()` in exactly the same way.
