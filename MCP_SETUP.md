# MCP Server Configuration

## What is MCP?
MCP (Model Context Protocol) servers allow AI assistants like Cursor to interact with external tools and services.

## Configuration

The MCP server configuration has been created in `mcp.json` in the project root.

## Setup Instructions

### For Cursor IDE:

1. **Global Configuration (Recommended):**
   - Open Cursor Settings
   - Go to "Features" → "MCP Servers"
   - Add the configuration from `mcp.json`
   - Or manually add to: `~/.cursor/mcp.json` (or `%USERPROFILE%\.cursor\mcp.json` on Windows)

2. **Project-Specific Configuration:**
   - Copy `mcp.json` to `.cursor/mcp.json` in your project root
   - Cursor will automatically detect it

### Configuration Details:

```json
{
  "mcpServers": {
    "opsera-devops-agent": {
      "type": "streamable-http",
      "url": "https://mcpserver-dev.agents.opsera-labs.com/mcp",
      "headers": {
        "Authorization": "Bearer opsera_1f2c7b2573de5d47af2d3144c3b053e914390d057bcdbcdf"
      }
    }
  }
}
```

## Security Note

⚠️ **Important:** The MCP configuration contains an authentication token. 
- The `mcp.json` file is already added to `.gitignore` to prevent committing sensitive tokens
- Never commit MCP configuration files with tokens to public repositories
- If you need to share the configuration, remove the token first

## Restart Cursor

After adding the MCP server configuration:
1. Restart Cursor IDE
2. The MCP server should be available in your AI assistant features

## Verify Setup

1. Open Cursor
2. Check if the MCP server is connected
3. You should see "opsera-devops-agent" in your MCP servers list

## Troubleshooting

- **Server not connecting:** Check your internet connection and verify the URL is accessible
- **Authentication error:** Verify the Bearer token is correct and not expired
- **Not showing up:** Make sure you've restarted Cursor after adding the configuration

