# Agent-to-User Protocol in LangChain

## Question

> Do agents built with LangChain SDK use a standard protocol between users
> and clients? For example, I use the SDK to build a travel agent that can
> research travel plans, book flights and hotels, and create Google Calendar
> events. I then host this agent to run in the cloud and expose it as a webapp
> to other users. Other users can use it for their own travels. What will the
> protocol between the agent and the users using it?

## Answer

**LangChain itself (including `langchain-community`) does not define a standard
protocol for serving agents to end users.** LangChain is a *framework for
building* agents — it handles the orchestration of LLM calls, tool usage,
memory, and chains — but it stops short of prescribing how you expose that agent
as a service.

## What LangChain Provides

- **Agent construction**: Tools, chains, memory, retrieval, etc.
- **Runnable interface** (`langchain-core`): A universal `.invoke()` /
  `.stream()` / `.batch()` API for composable components. This is an
  *in-process* Python API, not a network protocol.

## Options for Serving an Agent to Users

### 1. LangServe (for simpler, stateless use cases)

- Wraps any LangChain `Runnable` as a REST API (FastAPI-based).
- Provides endpoints like `/invoke`, `/stream`, `/batch`.
- Uses JSON over HTTP — but this is LangChain-specific, not an industry
  standard protocol.

### 2. LangGraph Platform (recommended for production)

- Designed specifically for stateful, multi-step agents.
- Provides a **REST API** with endpoints for:
  - Creating and managing **threads** (conversations)
  - Sending **runs** (agent invocations) to threads
  - **Streaming** responses (SSE-based)
  - Managing **assistants** and **cron jobs**
- Has a Python/JS SDK (`langgraph-sdk`) for clients.
- This is the closest thing to a "standard protocol" in the LangChain
  ecosystem, but it is **LangChain-specific**, not an open industry standard.

### 3. Agent Protocol (open standard)

- An emerging open standard called the
  [Agent Protocol](https://agentprotocol.ai/) (originally from the AI Engineer
  Foundation) aims to define a universal REST API for interacting with AI
  agents.
- LangChain/LangGraph has shown interest in compatibility, but it is not the
  default.

### 4. Model Context Protocol (MCP)

- Anthropic's MCP standardizes how agents **call tools** (agent → tool
  direction), not how users talk to agents.
- `langchain-community` has MCP tool integrations, but MCP is not a user-facing
  protocol.

### 5. Roll Your Own

- Many teams simply wrap their LangChain agent in a **FastAPI / Flask /
  Django** app with their own REST or WebSocket API, using whatever
  request/response schema fits their app.

## Summary for the Travel Agent Scenario

| Layer | What Handles It |
| --- | --- |
| Agent logic (research, book, calendar) | LangChain + tools |
| Hosting & API protocol | **You choose**: LangGraph Platform, LangServe, custom FastAPI, etc. |
| User ↔ Agent communication | Typically **JSON over HTTP/REST** or **SSE for streaming**; no single mandated standard |
| Client SDK | `langgraph-sdk` if using LangGraph Platform, or your own HTTP client |

**Bottom line:** There is no single mandated protocol. The LangChain
ecosystem's closest answer is the **LangGraph Platform API** (REST + SSE), but
you're free to wrap your agent in any web framework and define your own API
contract. If cross-ecosystem interoperability matters to you, look at the
emerging **Agent Protocol** standard.
