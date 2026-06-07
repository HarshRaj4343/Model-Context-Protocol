# Model Context Protocol — Learning Repo

Personal exploration of the **Model Context Protocol (MCP)** ecosystem — local servers, remote servers, and Claude Desktop integration.

---

## What is MCP?

MCP (Model Context Protocol) is an open standard by Anthropic that lets LLMs like Claude connect to external tools, data sources, and services in a standardised way. Think of it as a plugin system for AI agents.

```
Claude Desktop / Claude API
        │
        │  MCP Protocol (JSON-RPC over stdio / SSE / HTTP)
        │
  ┌─────▼──────┐
  │ MCP Server │  ← exposes: Tools, Resources, Prompts
  │  (local or │
  │   remote)  │
  └─────┬──────┘
        │
   DB / API / FS / ...
```

Three primitives:
- **Tools** — callable functions (like `add_expense`, `search_web`)
- **Resources** — readable data endpoints (like `expense://categories`)
- **Prompts** — reusable prompt templates

---

## Repo Structure

```
Model-Context-Protocol/
├── local_mcp_server/        ← ExpenseTracker local MCP server
│   ├── main.py              # FastMCP server (tools + resource)
│   ├── categories.json      # Expense category taxonomy
│   ├── pyproject.toml       # uv project config
│   └── README_local.md      # Local server docs
├── .python-version          # Python 3.13+
├── .gitignore
└── README.md                ← this file
```

---

## Local MCP Server — ExpenseTracker

Built with [FastMCP](https://github.com/jlowin/fastmcp). Tracks personal expenses via SQLite, exposed as MCP tools to Claude.

### Tools

| Tool | Args | Description |
|---|---|---|
| `add_expense` | `date, amount, category, subcategory?, note?` | Insert expense into SQLite |
| `list_expenses` | `start_date, end_date` | Fetch all rows in date range |
| `summarize` | `start_date, end_date, category?` | Sum by category |

### Resource

`expense://categories` → serves `categories.json` live (no restart needed after edits)

### Quick Start

```bash
cd local_mcp_server
uv sync
uv run main.py
```

### Claude Desktop Integration

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ExpenseTracker": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/Users/harshraj/path/to/local_mcp_server",
        "main.py"
      ]
    }
  }
}
```

Restart Claude Desktop. The 3 tools will appear automatically.

---

## Remote MCP Servers

Remote servers run over HTTP (SSE or Streamable HTTP transport) instead of stdio, making them accessible from anywhere — not just your local machine.

### Transport comparison

| | Local (stdio) | Remote (SSE / HTTP) |
|---|---|---|
| Where it runs | Same machine as Claude | Any server / cloud |
| Auth | None needed | OAuth 2.1 / API keys |
| Latency | ~0ms | Network latency |
| Use case | Personal tools, local FS | SaaS integrations, shared teams |

### Connecting a remote server in Claude Desktop

```json
{
  "mcpServers": {
    "MyRemoteServer": {
      "type": "sse",
      "url": "https://your-server.com/sse",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

For **Streamable HTTP** (newer, preferred):
```json
{
  "mcpServers": {
    "MyRemoteServer": {
      "type": "http",
      "url": "https://your-server.com/mcp"
    }
  }
}
```

### Hosting a FastMCP server remotely

```python
# main.py
from fastmcp import FastMCP

mcp = FastMCP("MyServer")

@mcp.tool()
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

Deploy on any VPS / Railway / Render:
```bash
pip install fastmcp
python main.py
# Server listens at http://0.0.0.0:8000/sse
```

### Popular remote MCP servers (public)

| Server | URL | What it does |
|---|---|---|
| GitHub | `https://api.githubcopilot.com/mcp/` | Issues, PRs, code search |
| Asana | `https://mcp.asana.com/sse` | Tasks, projects |
| Linear | `https://mcp.linear.app/sse` | Issues, cycles |
| Cloudflare | `https://mcp.cloudflare.com/sse` | DNS, Workers, KV |
| Stripe | `https://mcp.stripe.com/sse` | Payments, customers |
| Brave Search | `https://search.brave.com/mcp/sse` | Web search |

### Auth — OAuth 2.1

MCP remote servers use OAuth 2.1 PKCE for user-facing auth. Claude Desktop handles the browser flow automatically when you add a remote server — you just get redirected to login.

For server-to-server (API key based), just pass the key as a header (see config above).

---

## Key Concepts Cheatsheet

```python
from fastmcp import FastMCP

mcp = FastMCP("ServerName")

# Tool — callable by Claude
@mcp.tool()
def my_tool(arg: str) -> dict:
    return {"result": arg}

# Resource — readable data
@mcp.resource("data://my-resource", mime_type="application/json")
def my_resource():
    return '{"key": "value"}'

# Prompt — reusable template
@mcp.prompt()
def my_prompt(context: str) -> str:
    return f"Analyze this: {context}"

# Run local
mcp.run()  # stdio

# Run remote
mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

---

## References

- [MCP Spec](https://spec.modelcontextprotocol.io)
- [FastMCP GitHub](https://github.com/jlowin/fastmcp)
- [Anthropic MCP Docs](https://docs.anthropic.com/en/docs/agents-and-tools/mcp)
- [Claude Desktop MCP Guide](https://modelcontextprotocol.io/quickstart/user)
- [MCP Server Directory](https://glama.ai/mcp/servers)
