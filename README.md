<p align="center"><img src="icon.png" width="96" alt="Crevio"></p>

# Crevio MCP Server

Crevio's official MCP server. Connect Claude, ChatGPT, Cursor, VS Code or any MCP client to your [Crevio](https://crevio.co) account and:

- **Delegate work** to the Crevio agent, the AI that runs your small business: "refund the last order from jane@…", "write and schedule this week's posts". Long jobs run in the background as runs that you can follow, continue, and approve.
- **Run scripts** against the whole Crevio API (products, customers, orders, and more) and the 3,000+ apps connected to your account, without an agent turn.

This repository holds the connection configs and listing metadata. The server is hosted by Crevio; there is nothing to install or run.

| | |
|---|---|
| **URL** | `https://mcp.crevio.co/mcp` |
| **Transport** | Streamable HTTP |
| **Auth** | OAuth 2.1 with dynamic client registration (any plan, including Free), or an API token as `Authorization: Bearer` / `X-Api-Key` (Business plan) |
| **Registry** | [`co.crevio/crevio`](https://registry.modelcontextprotocol.io/v0.1/servers?search=co.crevio/crevio) |
| **Docs** | [crevio.co/docs/developer/mcp](https://crevio.co/docs/developer/mcp) |

You need a Crevio account. [Sign up free](https://crevio.co/users/signup), no card required.

## Connect

Most clients sign you in with OAuth the first time a tool is used; you only need the URL.

**Claude Code**: install the plugin from this repository's marketplace

```
/plugin marketplace add creviolabs/crevio-mcp
/plugin install crevio@crevio
```

or add the server directly:

```bash
claude mcp add --transport http crevio https://mcp.crevio.co/mcp
```

**Codex**

```bash
codex plugin marketplace add creviolabs/crevio-mcp
codex plugin add crevio@crevio
```

Start a new Codex session after installing.

**Claude.ai and Claude Desktop**: Settings → Connectors → Add custom connector → `https://mcp.crevio.co/mcp`.

**Cursor** (`~/.cursor/mcp.json`), **VS Code** and other clients that take a URL:

```json
{
  "mcpServers": {
    "crevio": {
      "url": "https://mcp.crevio.co/mcp"
    }
  }
}
```

**Gemini CLI**

```bash
gemini extensions install https://github.com/creviolabs/crevio-mcp
```

**Clients that only run local (stdio) servers**:

```json
{
  "mcpServers": {
    "crevio": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.crevio.co/mcp"]
    }
  }
}
```

**With an API token** (Business plan; create one under Settings → Developers → API tokens):

```bash
claude mcp add --transport http crevio https://mcp.crevio.co/mcp \
  --header "Authorization: Bearer $CREVIO_API_TOKEN"
```

Then ask your agent to call `whoami`. If it answers with your account, you're connected.

## Tools

| Tool | What it does |
|---|---|
| `ask_crevio` | Delegate a job to the Crevio agent and wait for the result |
| `start_chat`, `send_message` | Start a background job, or continue one with follow-up instructions |
| `list_chats`, `get_chat`, `list_messages` | Read chats and their transcripts |
| `wait_for_run`, `get_run`, `list_runs`, `cancel_run` | Follow, list and stop agent runs |
| `resolve_approvals` | Approve or deny actions a run is paused on |
| `list_integrations`, `list_integration_tools` | See what a script can call: the Crevio API and every connected app |
| `run_script` | Run JavaScript against the Crevio API and your integrations |
| `wait_for_script_run`, `get_script_run`, `cancel_script_run` | Follow and stop script runs |
| `get_file_download_url` | Download files a run or script produced |
| `list_audit_events` | Read the account's audit log |
| `whoami` | Check the account, plan, credits and limits behind the connection |

Every tool carries read-only and destructive hints. Actions on connected apps that need approval pause the run until you approve them.

## Support

- Docs: [crevio.co/docs/developer/mcp](https://crevio.co/docs/developer/mcp)
- Email: [hello@crevio.co](mailto:hello@crevio.co)
- Privacy: [crevio.co/legal/privacy-policy](https://crevio.co/legal/privacy-policy) · Terms: [crevio.co/legal/terms-of-service](https://crevio.co/legal/terms-of-service)

The configs in this repository are MIT licensed. The Crevio service is covered by its [terms](https://crevio.co/legal/terms-of-service).
