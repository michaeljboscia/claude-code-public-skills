---
name: n8n-troubleshooting
description: Diagnose and fix n8n MCP server connection issues, Node.js problems, and API connectivity
shareable: true
---

# n8n MCP Troubleshooting

## Quick Diagnosis

When n8n MCP servers fail to connect, follow this diagnostic path:

### 1. Test npx First
```bash
npx --version
```

**If this fails with library errors** (dyld, simdjson, etc.):
- Root cause: Node.js library dependency mismatch
- Fix: `brew reinstall node`
- Then restart Claude Desktop or run `/mcp` in CLI

### 2. Test n8n API Connectivity
```bash
# Test local n8n
curl -s -o /dev/null -w "%{http_code}" http://YOUR-INTERNAL-IP:5678/api/v1/workflows \
  --header "X-N8N-API-KEY: YOUR_KEY_HERE"

# Test cloud n8n
curl -s -o /dev/null -w "%{http_code}" https://your-n8n-cloud.app.n8n.cloud/api/v1/workflows \
  --header "X-N8N-API-KEY: YOUR_KEY_HERE"
```

**Expected:** HTTP 200
**If 401:** API key is invalid or expired
**If timeout:** Network/firewall issue or wrong URL

### 3. Test MCP Package Directly
```bash
# Should launch and show initialization logs
N8N_API_URL="http://YOUR-INTERNAL-IP:5678" \
N8N_API_KEY="YOUR_KEY" \
MCP_MODE="stdio" \
timeout 3 npx n8n-mcp 2>&1 | head -20
```

**Expected:** Logs showing "MCP server initialized" and "Database health check passed"

### 4. Check MCP Logs
```bash
# View recent errors
tail -50 ~/Library/Logs/Claude/mcp-server-n8n-cloud.log
tail -50 ~/Library/Logs/Claude/mcp-server-n8n-local.log
tail -50 ~/Library/Logs/Claude/mcp.log
```

## Common Issues & Fixes

### Issue: "Library not loaded: libsimdjson.XX.dylib"

**Symptoms:**
- npx crashes with dyld errors
- All MCP servers fail to connect
- Error mentions missing .dylib files

**Root Cause:**
Node.js was compiled against an old version of the simdjson library that no longer exists (typically after Homebrew upgrades).

**Fix:**
```bash
# Reinstall Node.js to rebuild against current libraries
brew reinstall node

# Verify fix
npx --version

# Reconnect MCP servers
# In Claude Code CLI: run /mcp
# In Claude Desktop: fully quit and restart the app
```

**Why This Works:**
Homebrew's `reinstall` rebuilds Node.js from scratch, linking it against the current versions of all dependencies (including simdjson).

### Issue: "Failed to reconnect to n8n-cloud/n8n-local"

**Symptoms:**
- `/mcp` command reports connection failure
- n8n tools missing from available tools
- Other MCP servers work fine

**Diagnostic Steps:**

1. **Check if it's a Node.js problem:**
   ```bash
   npx --version
   ```
   If this fails → See "Library not loaded" issue above

2. **Verify n8n API is accessible:**
   ```bash
   curl -I http://YOUR-INTERNAL-IP:5678/healthz
   ```
   Expected: HTTP 200

3. **Check API key validity:**
   Look in config: `~/Library/Application Support/Claude/claude_desktop_config.json`
   Test the key with curl (see "Test n8n API Connectivity" above)

4. **Check logs for specific error:**
   ```bash
   tail -100 ~/Library/Logs/Claude/mcp-server-n8n-cloud.log
   ```

**Common Causes:**
- **Old Node.js process cached:** Restart Claude Desktop completely
- **Invalid API key:** Regenerate in n8n UI (Settings → API)
- **Network issue:** Firewall blocking connection, VPN required
- **n8n server down:** Verify server is running

### Issue: MCP Config Not Loading

**Check current MCP configuration:**
```bash
cat ~/Library/Application\ Support/Claude/claude_desktop_config.json | jq .mcpServers
```

**Expected structure:**
```json
{
  "n8n-local": {
    "command": "npx",
    "args": ["n8n-mcp"],
    "env": {
      "MCP_MODE": "stdio",
      "LOG_LEVEL": "error",
      "N8N_API_URL": "http://YOUR-INTERNAL-IP:5678",
      "N8N_API_KEY": "eyJ..."
    }
  }
}
```

**Common mistakes:**
- Missing environment variables
- Wrong command path (should be `npx`, not full path)
- Invalid JSON syntax
- Typo in server name

### Issue: n8n Tools Not Available

**After connecting, verify tools are loaded:**

In Claude Code CLI:
```
List available n8n tools
```

Expected tools should include:
- `mcp__n8n-cloud__n8n_list_workflows`
- `mcp__n8n-cloud__n8n_get_workflow`
- `mcp__n8n-cloud__search_nodes`
- etc.

**If tools missing after successful connection:**
1. Check if auto-approval includes MCP tools
2. Restart Claude session completely
3. Check for multiple instances with naming conflicts

## Configuration Reference

### MCP Config Location
- **Claude Desktop:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Claude Code CLI:** Shares the same config file

### n8n MCP Server Configuration

**Local n8n (your-n8n-server):**
```json
{
  "n8n-local": {
    "command": "npx",
    "args": ["n8n-mcp"],
    "env": {
      "MCP_MODE": "stdio",
      "LOG_LEVEL": "error",
      "DISABLE_CONSOLE_OUTPUT": "true",
      "N8N_API_URL": "http://YOUR-INTERNAL-IP:5678",
      "N8N_API_KEY": "YOUR-API-KEY",
      "NODE_TLS_REJECT_UNAUTHORIZED": "0"
    }
  }
}
```

**Cloud n8n (YourCompany):**
```json
{
  "n8n-cloud": {
    "command": "npx",
    "args": ["n8n-mcp"],
    "env": {
      "MCP_MODE": "stdio",
      "LOG_LEVEL": "error",
      "DISABLE_CONSOLE_OUTPUT": "true",
      "N8N_API_URL": "https://your-n8n-cloud.app.n8n.cloud",
      "N8N_API_KEY": "YOUR-API-KEY"
    }
  }
}
```

## Emergency Recovery Workflow

If all n8n MCP connections are broken:

```bash
# 1. Test Node.js health
npx --version

# 2. If npx fails, reinstall Node.js
brew reinstall node

# 3. Verify npx works
npx --version

# 4. Test n8n API directly
curl -s http://YOUR-INTERNAL-IP:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: YOUR_KEY" | head -100

# 5. Test MCP package
N8N_API_URL="http://YOUR-INTERNAL-IP:5678" \
N8N_API_KEY="YOUR_KEY" \
timeout 3 npx n8n-mcp 2>&1 | grep -E "initialized|health check"

# 6. Restart Claude Desktop (full quit, not just close window)

# 7. In Claude Code CLI, reconnect
/mcp
```

## Preventive Maintenance

### After Homebrew Updates

When running `brew upgrade`, Node.js dependencies can get out of sync:

```bash
# After brew upgrade, check Node.js health
npx --version

# If any errors, reinstall Node.js
brew reinstall node
```

### Generating New API Keys

If you need to regenerate n8n API keys:

1. Log into n8n web UI
2. Go to Settings → API
3. Create new API key
4. Update `claude_desktop_config.json`
5. Restart Claude Desktop

### Monitoring MCP Health

Check MCP server logs periodically:
```bash
# Check for errors in last hour
find ~/Library/Logs/Claude -name "mcp*.log" -exec grep -l "error" {} \;

# View recent errors
tail -20 ~/Library/Logs/Claude/mcp.log
```

## Environment Details

**System Information:**
- OS: macOS
- Package Manager: Homebrew
- Node.js: Installed via Homebrew
- MCP Transport: stdio (not HTTP/SSE)

**n8n Instances:**
- Local: http://YOUR-INTERNAL-IP:5678 (your-n8n-server server)
- Cloud: https://your-n8n-cloud.app.n8n.cloud (production)

**Related Files:**
- MCP Config: `~/Library/Application Support/Claude/claude_desktop_config.json`
- MCP Logs: `~/Library/Logs/Claude/mcp*.log`
- Node.js: `/usr/local/Cellar/node/`
- npm cache: `~/.npm/`

## When to Use This Skill

- `/mcp` command reports n8n connection failures
- n8n MCP tools suddenly disappear
- After system updates or Homebrew upgrades
- When switching between local and cloud n8n instances
- After Node.js or npm updates
- When debugging "command not found: npx" errors
