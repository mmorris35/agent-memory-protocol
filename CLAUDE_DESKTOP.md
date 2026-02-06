# Using AMP with Claude Desktop

Claude Desktop supports MCP (Model Context Protocol). AMP nodes like Nellie speak MCP. Connect them and Claude gains persistent memory.

## Quick Setup

### 1. Find Your Config File

| OS | Path |
|----|------|
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **Linux** | `~/.config/claude/claude_desktop_config.json` |

Create the file if it doesn't exist.

### 2. Add Your AMP Node

**Local Nellie (same machine):**

```json
{
  "mcpServers": {
    "nellie": {
      "command": "curl",
      "args": ["-N", "http://localhost:8765/sse"]
    }
  }
}
```

**Nellie on Tailscale:**

```json
{
  "mcpServers": {
    "nellie": {
      "command": "curl", 
      "args": ["-N", "http://100.87.147.89:8765/sse"]
    }
  }
}
```

**Nellie via Cloudflare Tunnel:**

```json
{
  "mcpServers": {
    "nellie": {
      "command": "curl",
      "args": ["-N", "https://nellie.yourdomain.com/sse"]
    }
  }
}
```

### 3. Restart Claude Desktop

Quit and reopen Claude Desktop. Your AMP node's tools will appear.

## Available Tools

Once connected, Claude Desktop can use:

| Tool | Description |
|------|-------------|
| `amp_store` | Store a lesson, checkpoint, or snippet |
| `amp_search` | Semantic search across your memories |
| `amp_get` | Retrieve a specific record by ID |
| `amp_delete` | Remove a record |
| `amp_list` | List records with filters |
| `amp_status` | Check node health and stats |

## Example Conversation

**You:** "Remember that PostgreSQL needs connection pooling in production"

**Claude:** *Uses amp_store to save a lesson*

"Got it. I've stored that as a lesson about PostgreSQL connection pooling."

---

**You:** "What do I know about database performance?"

**Claude:** *Uses amp_search with query "database performance"*

"Based on your memories, you've noted:
- PostgreSQL needs connection pooling in production
- Always use indexes on frequently queried columns
- ..."

## Multiple AMP Nodes

You can connect multiple nodes:

```json
{
  "mcpServers": {
    "personal": {
      "command": "curl",
      "args": ["-N", "http://localhost:8765/sse"]
    },
    "work": {
      "command": "curl",
      "args": ["-N", "https://work-nellie.company.com/sse"]
    },
    "iot": {
      "command": "curl",
      "args": ["-N", "https://sensors.home.arpa/sse"]
    }
  }
}
```

Claude sees tools from all connected nodes.

## Troubleshooting

**Tools not appearing?**
- Check that the config file is valid JSON
- Verify your AMP node is running (`curl http://localhost:8765/amp/status`)
- Restart Claude Desktop completely (not just close window)

**Connection errors?**
- For Tailscale: ensure Tailscale is connected
- For Cloudflare Tunnel: ensure `cloudflared` is running
- Check firewall isn't blocking local connections

**SSE endpoint not responding?**
- Verify your AMP node supports SSE transport
- Try the URL directly in browser—should show "event stream" content type

## Security Notes

- Claude Desktop runs MCP locally—your data stays on your machine
- For remote nodes, use HTTPS (Cloudflare Tunnel provides this)
- Consider adding authentication for publicly-exposed nodes

## Claude Code

Claude Code (CLI) also supports MCP. Add to your config:

```bash
claude mcp add nellie --transport sse http://localhost:8765/sse
```

Or for remote:

```bash
claude mcp add nellie --transport sse https://nellie.yourdomain.com/sse
```

---

*Your memories, your machine, your Claude.*
