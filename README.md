# mcp-agent-python

> **AI Agent backed by an MCP (Model Context Protocol) server — Python edition.**  
> Lightweight, async-first, built with Anthropic SDK + FastMCP. Easy to extend with new MCP tool servers.

[![Python](https://img.shields.io/badge/Python-3.11+-blue)](https://python.org/)
[![Anthropic](https://img.shields.io/badge/Anthropic-SDK-orange)](https://docs.anthropic.com/)
[![MCP](https://img.shields.io/badge/MCP-2025--03--26-purple)](https://modelcontextprotocol.io/)

---

## What is MCP?

The **Model Context Protocol** (MCP) is an open standard that lets AI models discover and call tools exposed by external servers — databases, APIs, file systems, internal services — in a structured, typed, and secure way.

```
  ┌────────────────────┐       MCP Protocol        ┌──────────────────────┐
  │     AI Agent       │ ◄───────────────────────► │    MCP Server        │
  │  (Anthropic SDK)   │   list_tools()            │  (lazarus-mcp)       │
  │                    │   call_tool(name, args)   │                      │
  │  Async-first       │                           │  - list_events()     │
  │  FastMCP-powered   │                           │  - retry_event(id)   │
  │  LangChain-ready   │                           │  - failure_summary() │
  └────────────────────┘                           └──────────────────────┘
```

---

## Option A: Anthropic SDK + FastMCP (Recommended)

The most direct approach — the Anthropic Python SDK handles the agent loop natively, and `mcp` (the official client library) connects to any MCP server.

### Installation

```bash
pip install anthropic mcp fastmcp
```

### Agent Implementation

```python
import anthropic
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

SYSTEM_PROMPT = """You are a failure analysis assistant.
Use the available tools to inspect and help resolve healing events."""

async def run_agent(query: str):
    # Connect to MCP server
    server_params = StdioServerParameters(
        command="python",
        args=["-m", "lazarus_mcp.server"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Discover tools from MCP server at runtime
            tools_result = await session.list_tools()
            tools = [
                {
                    "name": t.name,
                    "description": t.description,
                    "input_schema": t.inputSchema,
                }
                for t in tools_result.tools
            ]

            client = anthropic.Anthropic()
            messages = [{"role": "user", "content": query}]

            # Agent loop
            while True:
                response = client.messages.create(
                    model="claude-sonnet-4-5",
                    max_tokens=4096,
                    system=SYSTEM_PROMPT,
                    tools=tools,
                    messages=messages,
                )

                if response.stop_reason == "end_turn":
                    return next(b.text for b in response.content if hasattr(b, "text"))

                # Handle tool calls
                messages.append({"role": "assistant", "content": response.content})
                tool_results = []

                for block in response.content:
                    if block.type == "tool_use":
                        result = await session.call_tool(block.name, block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": str(result.content),
                        })

                messages.append({"role": "user", "content": tool_results})
```

### FastAPI Wrapper

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.post("/ask")
async def ask(question: str):
    result = await run_agent(question)
    return {"answer": result}
```

---

## Option B: LangChain + MCP Adapters (Best for complex multi-agent chains)

```bash
pip install langchain-anthropic langchain-mcp-adapters langgraph
```

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-5")

async def build_agent():
    async with MultiServerMCPClient(
        {
            "lazarus": {
                "url": "http://localhost:8081/sse",
                "transport": "sse",
            }
        }
    ) as client:
        tools = client.get_tools()
        agent = create_react_agent(model, tools)

        result = await agent.ainvoke({
            "messages": [("user", "Show all PENDING events from the last 24 hours")]
        })
        return result
```

---

## Architecture

```
  ┌────────────────────────────────────────────────────────┐
  │              Python Agent                               │
  │                                                         │
  │  FastAPI /ask endpoint  (or CLI)                        │
  │        │                                                │
  │        ▼                                                │
  │  ┌─────────────────┐  list_tools / call_tool            │
  │  │  Anthropic SDK  │ ◄───────────────────────────────┐  │
  │  │  (claude model) │                                  │  │
  │  └────────┬────────┘                                  │  │
  │           │                          ┌────────────────┘  │
  │           │ tool_use blocks          │  MCP Client        │
  │           └─────────────────────────┘  (mcp library)     │
  └────────────────────────────────────────────────────────┘
                                │ stdio / HTTP+SSE
                                ▼
                   ┌──────────────────────────┐
                   │  lazarus-mcp             │
                   │  (FastMCP or Spring Boot) │
                   └──────────────────────────┘
```

---

## Choosing Option A vs Option B

| Consideration | Anthropic SDK + FastMCP | LangChain + MCP Adapters |
|---|---|---|
| **Complexity** | Simple, explicit agent loop | Higher-level abstraction |
| **Flexibility** | Full control over each step | ReAct patterns built-in |
| **Dependencies** | Minimal | Larger ecosystem |
| **Best for** | Single-purpose agents, learning | Multi-agent chains, complex flows |
| **Streaming** | asyncio native | LangGraph streaming |

---

## Example Queries

```python
import asyncio

async def main():
    # Failure analysis
    print(await run_agent(
        "How many events failed in the last hour and which methods are most affected?"
    ))

    # Bulk retry
    print(await run_agent(
        "Retry all pending ORDER events that failed with TransientDataException"
    ))

    # Deep inspection
    print(await run_agent(
        "Show me the payload of event abc-123 — it looks like a data issue"
    ))

asyncio.run(main())
```

---

## Related Projects

- **[lazarus-lib](https://github.com/byte-by-k/lazarus-lib)** — The Spring library that captures failures into the Healer DB
- **[lazarus-mcp](https://github.com/byte-by-k/lazarus-mcp)** — The MCP server this agent talks to
- **[mcp-agent-java](https://github.com/byte-by-k/mcp-agent-java)** — The same agent pattern in Java

---

## License

MIT © Kamlesh
