---
name: n8n-api-mcp-server
description: Complete reference for n8n REST API endpoints and MCP server tools
shareable: true
---

# n8n REST API & MCP Server - Claude Code Skill

**Last Updated:** 2026-01-31
**API Version:** v1.1.1 | **MCP Server Version:** v2.33.4
**Purpose:** Complete reference for n8n REST API endpoints and MCP server tools

---

## Overview

This skill covers two interfaces for programmatic n8n access:
1. **n8n REST API** - Direct HTTP endpoints for workflow/execution management
2. **n8n MCP Server** - 20 MCP tools for AI assistants (czlonkowski/n8n-mcp)

**Primary Use Cases:**
- Programmatic workflow creation/updates via API
- Using MCP tools to interact with n8n
- Understanding which tools/endpoints exist (anti-hallucination)
- Debugging API/MCP connection issues
- Template deployment and version management

**Related Skills:**
- `n8n-platform-reference.md` - Platform concepts, expressions, node behavior
- `n8n-node-implementation.md` - typeVersions, JSON structures for workflow generation
- `n8n-workflow-development.md` - Workflow building patterns
- `n8n-troubleshooting.md` - Connection issues, debugging

---

# PART 1: n8n REST API Reference

The n8n Public REST API (OpenAPI 3.0.0) provides programmatic access to workflows, executions, credentials, and administrative functions. All endpoints use the `/api/v1` prefix.

## Authentication

⚠️ **CRITICAL:** Uses `X-N8N-API-KEY` header—NOT Bearer tokens.

| Header | Value |
|--------|-------|
| `X-N8N-API-KEY` | Your API key (required) |
| `Content-Type` | `application/json` (for POST/PUT/PATCH) |

```bash
curl -X GET "https://your-n8n.com/api/v1/workflows" \
  -H "X-N8N-API-KEY: your-api-key"
```

Generate keys at: **Settings → n8n API → Create API Key**

---

## Pagination

All list endpoints use cursor-based pagination.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | integer | 100 | Results per page (max: 250) |
| `cursor` | string | — | Pagination cursor from previous response |

---

## Complete Endpoint Reference

### Workflows (`/workflows`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/workflows` | List all workflows |
| `POST` | `/workflows` | Create new workflow |
| `GET` | `/workflows/{id}` | Get specific workflow |
| `PUT` | `/workflows/{id}` | Update workflow (full replacement) |
| `DELETE` | `/workflows/{id}` | Delete workflow |
| `POST` | `/workflows/{id}/activate` | Activate workflow |
| `POST` | `/workflows/{id}/deactivate` | Deactivate workflow |
| `PUT` | `/workflows/{id}/transfer` | Transfer to another project |
| `GET` | `/workflows/{id}/tags` | Get workflow tags |
| `PUT` | `/workflows/{id}/tags` | Update workflow tags |

#### GET /workflows - Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `active` | boolean | No | Filter by active status |
| `tags` | string | No | Comma-separated tag names |
| `name` | string | No | Filter by workflow name |
| `projectId` | string | No | Filter by project (Enterprise) |
| `excludePinnedData` | boolean | No | Exclude pinned test data |

#### POST /workflows - Request Body

```json
{
  "name": "string (required)",
  "nodes": [
    {
      "id": "string (UUID)",
      "name": "string (unique within workflow)",
      "type": "string (e.g., n8n-nodes-base.webhook)",
      "typeVersion": "number (e.g., 1, 1.1, 2)",
      "position": "[x, y] coordinates",
      "parameters": "object (node-specific)",
      "credentials": "object (optional)",
      "disabled": "boolean (optional)"
    }
  ],
  "connections": {
    "NodeName": {
      "main": [[{"node": "TargetNode", "type": "main", "index": 0}]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": "boolean",
    "timezone": "string (e.g., America/New_York)"
  }
}
```

---

### Executions (`/executions`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/executions` | List executions |
| `GET` | `/executions/{id}` | Get specific execution |
| `DELETE` | `/executions/{id}` | Delete execution |
| `POST` | `/executions/{id}/retry` | Retry failed execution |

#### GET /executions - Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `includeData` | boolean | Include full execution data |
| `status` | enum | `canceled`, `error`, `running`, `success`, `waiting` |
| `workflowId` | string | Filter by workflow |

⚠️ **Known Issue:** `waiting` status may not return results in some versions.

---

### Credentials (`/credentials`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/credentials` | List credentials (metadata only) |
| `POST` | `/credentials` | Create credential |
| `DELETE` | `/credentials/{id}` | Delete credential (owner only) |
| `GET` | `/credentials/schema/{credentialTypeName}` | Get credential JSON schema |

⚠️ **Note:** Credentials cannot be updated via API (`PUT /credentials/{id}` does NOT exist).

---

### Tags, Users, Variables, Projects

| Resource | Endpoints |
|----------|-----------|
| **Tags** | `GET/POST/PUT/DELETE /tags`, `GET/PUT/DELETE /tags/{id}` |
| **Users** | `GET/POST /users`, `GET/DELETE /users/{id}`, `PATCH /users/{id}/role` (Admin only) |
| **Variables** | `GET/POST /variables`, `PUT/DELETE /variables/{id}` (Pro/Enterprise) |
| **Projects** | `GET/POST /projects`, `PUT/DELETE /projects/{id}` (Enterprise) |

---

## HTTP Response Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 200 | Success | GET, PUT, POST completed |
| 201 | Created | Resource successfully created |
| 204 | No Content | DELETE completed |
| 400 | Bad Request | Invalid parameters, malformed JSON |
| 401 | Unauthorized | Missing/invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate name, version conflict |
| 429 | Too Many Requests | Rate limited |

---

## ⛔ Anti-Hallucination: API Endpoints

### Endpoints That DO NOT Exist

| Hallucinated Endpoint | Reality |
|-----------------------|---------|
| `POST /workflows/{id}/execute` | ❌ Does not exist—use webhook triggers |
| `POST /workflows/{id}/run` | ❌ Does not exist |
| `GET /workflows/{id}/status` | ❌ Does not exist—check executions |
| `GET /nodes` | ❌ No API to list available node types |
| `/api/v2/*` | ❌ Only v1 exists |
| `PUT /credentials/{id}` | ❌ Credentials cannot be updated via API |

### Correct vs Incorrect Patterns

| ❌ WRONG | ✅ CORRECT |
|----------|-----------|
| `Authorization: Bearer KEY` | `X-N8N-API-KEY: KEY` |
| `/rest/workflows` | `/api/v1/workflows` |
| `"active": true` in POST body | Use `/activate` endpoint |
| `nodeId` in connections | `name` (nodes use name) |

---

## Critical Known Issues

1. **Webhook registration bug**: Workflows activated via API may not properly register webhooks. Save via UI after API activation.

2. **workflowId type mismatch**: OpenAPI spec defines `execution.workflowId` as number but API returns alphanumeric strings.

3. **Waiting executions**: `GET /executions` may not return `status: "waiting"` executions.

4. **Encryption key dependency**: Changing `N8N_ENCRYPTION_KEY` invalidates ALL stored credentials.

---

# PART 2: n8n MCP Server Reference

**Primary Server:** `czlonkowski/n8n-mcp` (MIT license, 12,500+ stars)
**npm Package:** `n8n-mcp`
**Coverage:** 1,084 nodes (537 core + 547 community), 2,709 templates

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `MCP_MODE` | No | `stdio` (Claude Desktop) or `http` |
| `LOG_LEVEL` | No | `error`, `warn`, `info`, `debug` |
| `DISABLE_CONSOLE_OUTPUT` | No | `true` for clean stdio |
| `N8N_API_URL` | For management | n8n instance URL |
| `N8N_API_KEY` | For management | n8n API key |

### Claude Desktop Configuration

```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "npx",
      "args": ["n8n-mcp"],
      "env": {
        "MCP_MODE": "stdio",
        "LOG_LEVEL": "error",
        "DISABLE_CONSOLE_OUTPUT": "true",
        "N8N_API_URL": "https://your-n8n.com",
        "N8N_API_KEY": "your-api-key"
      }
    }
  }
}
```

---

## Complete Tool Reference (20 Tools)

### Documentation Tools (No API key required)

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `tools_documentation` | Get help for any MCP tool | `toolName`, `detail` (brief/full) |
| `search_nodes` | Full-text search across 1,084 nodes | `query` (required), `source`, `limit` |
| `get_node` | Unified node information | `nodeType` (required), `mode`, `detail` |
| `validate_node` | Validate node config against schema | `nodeType`, `config` (required) |
| `validate_workflow` | Complete workflow validation | `workflow` (required), `profile` |
| `search_templates` | Search 2,709 workflow templates | `searchMode`, `query`, `complexity` |
| `get_template` | Get complete template JSON | `templateId` (required), `mode` |

### Management Tools (Require N8N_API_KEY)

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `n8n_create_workflow` | Create new workflow (inactive) | `name`, `nodes`, `connections` (all required) |
| `n8n_get_workflow` | Retrieve workflow | `id` (required), `mode` |
| `n8n_update_full_workflow` | Complete workflow replacement | `id` (required), `nodes`, `connections` |
| `n8n_update_partial_workflow` | Diff-based incremental updates | `id`, `operations` (required) |
| `n8n_delete_workflow` | Permanently delete | `id` (required) |
| `n8n_list_workflows` | List with filtering | `limit`, `active`, `tags` |
| `n8n_validate_workflow` | Validate by ID | `id` (required) |
| `n8n_autofix_workflow` | Auto-fix validation errors | `id` (required), `applyFixes` |
| `n8n_test_workflow` | Trigger execution | `workflowId` (required), `triggerType`, `data` |
| `n8n_executions` | Manage executions | `action` (get/list/delete), `id` |
| `n8n_workflow_versions` | Version history/rollback | `mode` (list/get/rollback), `workflowId` |
| `n8n_deploy_template` | Deploy from n8n.io | `templateId` (required), `autoFix` |
| `n8n_health_check` | Check API connectivity | (none) |

---

## Key Tool Details

### `get_node` Modes

| Mode | Token Cost | Description |
|------|------------|-------------|
| `info` + `minimal` | ~200 | Basic metadata |
| `info` + `standard` | ~500 | Essential 10-20 properties |
| `info` + `full` | ~3000-8000 | Complete schema |
| `docs` | Varies | Human-readable markdown |
| `search_properties` | Low | Find specific properties |
| `versions` | Low | List all node versions |

### `n8n_update_partial_workflow` Operation Types

**Node Operations:** `addNode`, `removeNode`, `updateNode`, `moveNode`, `enableNode`, `disableNode`

**Connection Operations:** `addConnection`, `removeConnection`, `rewireConnection`, `cleanStaleConnections`, `replaceConnections`

**Metadata Operations:** `updateSettings`, `updateName`, `addTag`, `removeTag`

**State Operations:** `activateWorkflow`, `deactivateWorkflow`

### `n8n_autofix_workflow` Fix Types

`expression-format`, `typeversion-correction`, `error-output-config`, `node-type-correction`, `webhook-missing-path`, `typeversion-upgrade`, `version-migration`

---

## Quick Reference Table

| Tool | Category | API Key | Primary Use |
|------|----------|---------|-------------|
| `tools_documentation` | Docs | No | Get tool help |
| `search_nodes` | Docs | No | Find nodes by keyword |
| `get_node` | Docs | No | Get node details/schema |
| `validate_node` | Validation | No | Validate node config |
| `validate_workflow` | Validation | No | Validate workflow JSON |
| `search_templates` | Docs | No | Find workflow templates |
| `get_template` | Docs | No | Get template JSON |
| `n8n_create_workflow` | Mgmt | **Yes** | Create workflow |
| `n8n_get_workflow` | Mgmt | **Yes** | Fetch workflow |
| `n8n_update_full_workflow` | Mgmt | **Yes** | Replace workflow |
| `n8n_update_partial_workflow` | Mgmt | **Yes** | Diff-based updates |
| `n8n_delete_workflow` | Mgmt | **Yes** | Delete workflow |
| `n8n_list_workflows` | Mgmt | **Yes** | List workflows |
| `n8n_validate_workflow` | Mgmt | **Yes** | Validate by ID |
| `n8n_autofix_workflow` | Mgmt | **Yes** | Auto-fix errors |
| `n8n_test_workflow` | Exec | **Yes** | Trigger execution |
| `n8n_executions` | Exec | **Yes** | Manage executions |
| `n8n_workflow_versions` | Mgmt | **Yes** | Version control |
| `n8n_deploy_template` | Mgmt | **Yes** | Deploy templates |
| `n8n_health_check` | System | **Yes** | Check connectivity |

---

## ⛔ Anti-Hallucination: MCP Tools

### Tools That DO NOT Exist

| Hallucinated Tool | Use Instead |
|-------------------|-------------|
| `execute_workflow` | `n8n_test_workflow` |
| `get_nodes` | `search_nodes` |
| `get_credentials` | Limited operations only |
| `create_node` | Create workflow with nodes array |
| `update_node` | `n8n_update_partial_workflow` |
| `list_nodes` | `search_nodes` with broad query |
| `run_workflow` | `n8n_test_workflow` |

### Common Parameter Mistakes

| ❌ Wrong | ✅ Correct | Tool |
|----------|-----------|------|
| `name` | `nodeType` | get_node |
| `workflow_id` | `workflowId` | n8n_test_workflow |
| `workflow` | `id` | n8n_get_workflow |
| `"Slack"` | `"n8n-nodes-base.slack"` | get_node nodeType |
| `mode: "run"` | `action: "get"` | n8n_executions |

### Node Type Format

```json
// ❌ WRONG
{"tool": "get_node", "nodeType": "Slack"}
{"tool": "get_node", "nodeType": "slack"}

// ✅ CORRECT - Always include package prefix
{"tool": "get_node", "nodeType": "n8n-nodes-base.slack"}
{"tool": "get_node", "nodeType": "@n8n/n8n-nodes-langchain.agent"}
```

---

## Common Error Codes

### MCP Connection Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `AUTHENTICATION_ERROR` | Invalid API key | Verify `N8N_API_KEY` |
| `Error 1001 (Status: 403)` | Permission denied | Check key scopes |
| `Cannot connect to MCP server` | Protocol mismatch | Verify `MCP_MODE` setting |
| `Failed to initialize` | Missing dependencies | Run `npm install` |

### API-Related Errors

| Error | Cause | Solution |
|-------|-------|----------|
| 401 Unauthorized | Wrong header | Use `X-N8N-API-KEY` |
| 404 Not Found | Wrong URL | Include `/api/v1` prefix |
| Webhook 404 | Registration bug | Save workflow in UI |
| Empty execution data | Waiting status | Known limitation |

---

## Alternative MCP Servers

### leonardsellem/n8n-mcp-server

Simpler alternative (12 tools):

| Tool | Description |
|------|-------------|
| `workflow_list` | List workflows |
| `workflow_get` | Get workflow |
| `workflow_create` | Create workflow |
| `workflow_update` | Update workflow |
| `workflow_delete` | Delete workflow |
| `workflow_activate` | Activate workflow |
| `workflow_deactivate` | Deactivate workflow |
| `execution_run` | Run execution |
| `run_webhook` | Trigger webhook |
| `execution_get` | Get execution |
| `execution_list` | List executions |
| `execution_stop` | Stop execution |

---

## Key Gotchas

1. **API auth is `X-N8N-API-KEY`** - NOT Bearer tokens
2. **No `/api/v2`** - Only v1 exists
3. **No `PUT /credentials/{id}`** - Credentials cannot be updated via API
4. **No `POST /workflows/{id}/execute`** - Use webhook triggers or MCP `n8n_test_workflow`
5. **Workflows created via API are inactive** - Must activate separately
6. **Webhook registration bug** - Save workflow in UI after API activation
7. **nodeType must include package prefix** - `n8n-nodes-base.slack`, not `Slack`
8. **`waiting` status may not return** - Known API limitation

---

**Source Document:** `/Users/mikeboscia/My Drive/GTM Machine content/Research & Strategy/Claude Code Skill Reference- n8n API & MCP Server.md`
