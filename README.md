# Rius MCP server

[![Rius MCP connector](https://glama.ai/mcp/connectors/ai.glassflow/rius/badges/score.svg)](https://glama.ai/mcp/connectors/ai.glassflow/rius)

[Rius](https://www.glassflow.ai/rius) is an observability platform for AI agents, built by GlassFlow. The Rius MCP server lets your coding assistant query your agents' traces, errors, cost and alerts in plain language, from VS Code (GitHub Copilot), Claude Code, Cursor, Codex or any other MCP client.

This repository holds the public README for the hosted server, plus a reference copy of its `server.json`. The live registry entry is published by the server's release pipeline, so its `version` may be newer than the copy here. The server itself runs as a managed service; there is nothing to install or self-host.

- **Endpoint:** `https://mcp.eu.console.rius-glassflow.com/mcp`
- **Transport:** Streamable HTTP
- **Auth:** OAuth 2.1 sign-in in the browser, or a Rius API key for headless use
- **MCP registry name:** `ai.glassflow/rius`
- **Docs:** https://docs.glassflow.ai/rius/mcp

## What you can ask

Once connected, ask questions about your agents in the chat:

- "Which of my agents had the most errors in the last 24 hours, and what are the top error groups?"
- "Show me today's failed runs for `support-agent` and walk me through the worst one."
- "Compare cost and latency between versions 1.4.0 and 1.5.0 of `support-agent`."
- "Which end users cost the most this week?"
- "List open alerts and explain what triggered each one."
- "Create an alert that fires when `checkout-agent` error rate goes above 5%."

## Prerequisites

A Rius workspace with agent traces in it. [Sign up](https://console.rius-glassflow.com/signup) and follow the [quickstart](https://docs.glassflow.ai/rius/quickstart) to send your first traces with the Rius SDK or any OpenTelemetry exporter.

## Connect

Every client below uses the server name `rius`. On first use the client signs you in to Rius through the browser (some clients need a command to start it; see the per-client notes). The server supports dynamic client registration and client ID metadata documents, so there is no client ID to configure.

### VS Code (GitHub Copilot)

Add the server to `.vscode/mcp.json` in your workspace (or your user `mcp.json`). Note the top-level key is `servers`:

```json
{
  "servers": {
    "rius": {
      "type": "http",
      "url": "https://mcp.eu.console.rius-glassflow.com/mcp"
    }
  }
}
```

Start the server from the **Start** action above its entry in `mcp.json` (or run **MCP: List Servers**) and allow the sign-in prompt. Then open Copilot Chat, switch to Agent mode and select the `rius` tools.

### Claude Code

```bash
claude mcp add --transport http rius https://mcp.eu.console.rius-glassflow.com/mcp
```

Run `/mcp` inside Claude Code to sign in.

### Cursor

Add to `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

```json
{
  "mcpServers": {
    "rius": {
      "url": "https://mcp.eu.console.rius-glassflow.com/mcp"
    }
  }
}
```

Then approve the server under **Settings → MCP**.

### Codex

```bash
codex mcp add rius --url https://mcp.eu.console.rius-glassflow.com/mcp
```

### Other clients

Most clients accept a JSON block like this one:

```json
{
  "mcpServers": {
    "rius": {
      "type": "http",
      "url": "https://mcp.eu.console.rius-glassflow.com/mcp"
    }
  }
}
```

Clients that only support stdio can bridge through `mcp-remote`:

```bash
npx mcp-remote https://mcp.eu.console.rius-glassflow.com/mcp
```

Per-client notes (Claude Desktop, Windsurf, Cline, Zed, Gemini CLI, opencode) are in the [MCP server docs](https://docs.glassflow.ai/rius/mcp).

## Headless use with an API key

For CI jobs, scripts or remote agents that cannot open a browser, send a Rius API key as a bearer token instead of signing in. Create a key with the **Query data** capability from the console's MCP page ("Connect an agent") or the Create API key dialog.

```bash
export RIUS_API_KEY=...
claude mcp add --transport http rius https://mcp.eu.console.rius-glassflow.com/mcp \
  --header "Authorization: Bearer ${RIUS_API_KEY}"
```

In VS Code, let it prompt for the key once and store it securely:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "rius-key",
      "description": "Rius API key",
      "password": true
    }
  ],
  "servers": {
    "rius": {
      "type": "http",
      "url": "https://mcp.eu.console.rius-glassflow.com/mcp",
      "headers": {
        "Authorization": "Bearer ${input:rius-key}"
      }
    }
  }
}
```

In other JSON configs, add a `headers` block. Not every client expands environment variables, so use your client's variable syntax or paste the key:

```json
{
  "mcpServers": {
    "rius": {
      "type": "http",
      "url": "https://mcp.eu.console.rius-glassflow.com/mcp",
      "headers": {
        "Authorization": "Bearer ${RIUS_API_KEY}"
      }
    }
  }
}
```

In Codex, set `bearer_token_env_var = "RIUS_API_KEY"` on the server entry in `~/.codex/config.toml`.

## Tools

The main tools are below. The full, current reference with parameters is at https://docs.glassflow.ai/rius/mcp/tools.

| Area | Tools |
| --- | --- |
| Workspace and identity | `get_me`, `list_workspaces`, `list_users`, `user_activity_summary` |
| Agents | `list_agents`, `agent_versions`, `compare_agent_versions`, `agent_lifecycle_events` |
| Traces and spans | `agent_traces_summary`, `list_agent_traces`, `get_agent_trace`, `list_spans`, `describe_search_catalog` |
| Errors and metrics | `agent_error_groups`, `workspace_metrics_overview`, `list_metrics_catalog` |
| Alerts (read) | `list_alerts`, `get_alert`, `list_alert_rules`, `list_alert_channels`, `describe_alert_catalog` |
| Alerts (write) | `create_alert_rule`, `update_alert_rule`, `delete_alert_rule`, `create_alert_channel`, `update_alert_channel`, `delete_alert_channel`, `ack_alert` |
| Keys (write) | `create_api_key` |

The tools marked "write" change data. When a call is not allowed, the server tells the assistant why instead of failing silently:

- **Signed in:** read tools and `ack_alert` work for every member. Alert rule and channel tools, `list_alert_channels` and `create_api_key` need an org admin.
- **With an API key:** a key is read-only unless it was created with `alerts:manage` (alert rule tools) or `channels:manage` (alert channel tools, including `list_alert_channels`). `ack_alert` and `create_api_key` never work with a key.

## Links

- Rius: https://www.glassflow.ai/rius
- Console: https://console.rius-glassflow.com
- MCP server docs: https://docs.glassflow.ai/rius/mcp
- Claude Code plugin (traces your own Claude Code sessions into Rius): https://github.com/glassflow/rius-coding-agents

## License

The contents of this repository are released under the [MIT License](LICENSE). The Rius service is governed by its own terms.
