---
title: MCP Integration
---

# MCP Integration

WSO2 Integrator can be both an **MCP server** (exposing your integrations as standardised tools for AI assistants) and an **MCP client** (using tools published by other MCP servers from inside your own AI agents).

This section is the feature reference. For the protocol concepts see [What is MCP?](/docs/genai/key-concepts/what-is-mcp).

## Features at a Glance

| Feature | What it is | Where you find it in BI |
|---|---|---|
| [Exposing a Service as MCP](exposing-as-mcp.md) | Build an MCP service in BI — pick a listener, add tools, configure each tool's name, description, and parameters. | **Artifacts** → **MCP Service**. |
| [Consuming MCP from an Agent](consuming-mcp-from-agent.md) | Add an MCP server as a tool source for an AI Agent, optionally filtered to specific tools. | Agent canvas → **+ Add Tool** → **Use MCP Server**. |

The two sides of MCP map cleanly to the two pages — the wizard builds the server, the agent's Add Tool dialog consumes it.

## When to Use MCP

| Use MCP when… | Look elsewhere when… |
|---|---|
| You want your integration's capabilities reusable by Claude Desktop, Copilot, or any other MCP client. | Only your own agents will consume the tools — local function tools work fine. |
| You want to consume tools published by community or vendor MCP servers. | The tools live behind a normal HTTP API and you can use the connector — just wrap the connector. |
| You need a standard, governed boundary between the AI surface and your tools (auth, rate limits, audit). | Tooling is internal-only and ephemeral. |

## The Two Sides Side-by-Side

```
┌────────────────────────────┐                ┌────────────────────────────┐
│  AI Assistant / Agent       │                │   AI Assistant / Agent      │
│  (Claude Desktop, Copilot,  │                │   (your AI agent)           │
│   your own AI agent)        │                │                             │
└──────────────┬──────────────┘                └──────────────┬──────────────┘
               │   MCP                                        │   MCP
               ▼                                              ▼
┌────────────────────────────┐                ┌────────────────────────────┐
│   Your WSO2 Integrator       │                │   Someone else's MCP server │
│   MCP Service                │                │   (Slack, GitHub, vendor)   │
│                              │                │                             │
│   tools = remote functions   │                │   tools = whatever they      │
│           in BI              │                │           expose             │
└────────────────────────────┘                └────────────────────────────┘
   (Exposing as MCP)                              (Consuming MCP from an agent)
```

You can do both in the same project. A single integration might expose a few tools (your CRM lookups) over MCP for outside assistants while also consuming Slack and GitHub MCP servers from its own AI agents.

## Where MCP Lives in BI

The MCP feature surface is split between the artifact catalogue and the agent's Add Tool dialog:

| Surface | What it does | Created where |
|---|---|---|
| **MCP Service** artifact | A new top-level artifact that publishes tools over MCP. Listed under **AI Integration** alongside AI Chat Agent. | Artifacts → AI Integration → MCP Service |
| **mcp:Listener** | The listener that an MCP Service runs on. Created automatically when you make the first MCP Service. | Listeners → mcpListener |
| **`mcp:Service` / `mcp:AdvancedService`** | The actual service body. `mcp:Service` derives tools from `remote function`s; `mcp:AdvancedService` lets you implement `onListTools`/`onCallTool` for dynamic tool sets. | Inside the MCP Service editor |
| **Add MCP Server** panel | The tool-source picker on the agent canvas. Adds an `ai:McpToolKit` connection that pulls tools from a remote MCP server. | Agent canvas → + Add Tool → Use MCP Server |
| **Tool Configuration** panel | Per-tool editor inside an MCP Service — Name, Description, Parameters, Return Type. | MCP Service editor → + Add Tool |

## Transports

MCP runs over two transports; BI defaults to Streamable HTTP because it's the modern, web-friendly variant:

| Transport | When to use | BI default for new MCP services? |
|---|---|---|
| **Streamable HTTP** | Remote assistants, web-based tooling, anything that talks HTTP. The default and recommended transport. | Yes |
| **stdio** | Local clients like Claude Desktop that launch your service as a subprocess. | No (configure manually) |

For Streamable HTTP, the listener exposes a single endpoint (e.g. `http://localhost:9090/mcp`); clients connect to it and the protocol multiplexes tool listing and tool calls over the same connection.

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| MCP client connects but sees no tools. | The `mcp:Service` has no `remote function`s yet, or tools haven't been added via the editor. | Click **+ Add Tool** in the MCP Service editor. |
| Client gets HTTP 404. | Wrong URL — base path or listener port mismatch. | Check the listener port (`mcpListener` in the project sidebar) and the service base path (`/mcp` by default). |
| Tool returns "Invalid arguments" for what looks like the right input. | The tool's parameter type is too strict, or the agent passed the wrong shape. | Look at the `onCallTool` traceback in `bal run` output; loosen the parameter type or add a clearer description. |
| Agent picks the wrong MCP tool. | Tool descriptions are too generic or overlap. | Tighten each tool's first sentence — name *what* and *when*. |
| Agent prompt becomes huge after adding an MCP server. | Server advertises many tools and `Tools to Include` is `All`. | Filter `Tools to Include` to just the names you need. |
| MCP Service can't write to its database from a tool. | The service's connection pool isn't configured for writes. | Edit the connector connection's Advanced Configurations or expose a write-capable client variant. |
| MCP calls timeout from a remote agent. | Default 30s timeout is too short for slow tools. | Raise **Timeout** on the agent's Add MCP Server panel. |

## What's Next

- **[Exposing a Service as MCP](exposing-as-mcp.md)** — turn your integration into an MCP server.
- **[Consuming MCP from an Agent](consuming-mcp-from-agent.md)** — let your agent use tools from any MCP server.
- **[What is MCP?](/docs/genai/key-concepts/what-is-mcp)** — protocol concepts.
