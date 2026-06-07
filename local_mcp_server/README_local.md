# ExpenseTracker — Local MCP Server

Local MCP server built with [FastMCP](https://github.com/jlowin/fastmcp) that exposes expense tracking tools to Claude Desktop (or any MCP client).

## Files
| File | Purpose |
|---|---|
| `main.py` | FastMCP server — tools + resource |
| `categories.json` | Editable category/subcategory taxonomy |
| `expenses.db` | SQLite DB (auto-created on first run) |
| `pyproject.toml` | Project metadata + deps |

## Tools exposed
| Tool | Args | Description |
|---|---|---|
| `add_expense` | `date, amount, category, subcategory?, note?` | Insert a new expense row |
| `list_expenses` | `start_date, end_date` | Fetch rows in date range |
| `summarize` | `start_date, end_date, category?` | Grouped sum by category |

## Resource
`expense://categories` — serves `categories.json` as `application/json` (live-reloaded, no server restart needed).

## Setup
```bash
cd local_mcp_server
uv sync          # or: pip install fastmcp
uv run main.py   # test run
```

## Claude Desktop config
Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "ExpenseTracker": {
      "command": "uv",
      "args": ["run", "--directory", "/absolute/path/to/local_mcp_server", "main.py"]
    }
  }
}
```
