---
name: n8n-api-mcp-server
description: Complete reference for n8n REST API endpoints and MCP server tools
shareable: true
---

# n8n REST API & MCP Server - Claude Code Skill

**Last Updated:** 2026-01-31
**API Version:** v1.1.1 | **MCP Server Version:** v2.33.x
**Purpose:** Exhaustive reference for n8n REST API endpoints and all 20 MCP server tools

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

**Base URL:** `https://your-instance.com/api/v1`
**OpenAPI:** 3.0.0

## Authentication

⚠️ **CRITICAL:** Uses `X-N8N-API-KEY` header—NOT Bearer tokens.

| Header | Value | Required |
|--------|-------|----------|
| `X-N8N-API-KEY` | Your API key from Settings → n8n API | Yes |
| `Content-Type` | `application/json` (for POST/PUT/PATCH) | Yes |

```bash
curl -X GET "https://your-n8n.com/api/v1/workflows" \
  -H "X-N8N-API-KEY: your-api-key"
```

---

## HTTP Status Codes

| Code | Name | Description |
|------|------|-------------|
| 200 | OK | Operation successful |
| 201 | Created | Resource created |
| 204 | No Content | Success, no body |
| 400 | Bad Request | Invalid parameters/malformed JSON |
| 401 | Unauthorized | Invalid/missing API key |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate or version conflict |
| 415 | Unsupported Media Type | Wrong content type |

---

## Pagination (All List Endpoints)

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `limit` | number | 100 | 250 | Maximum items per page |
| `cursor` | string | - | - | Pagination cursor from `nextCursor` |

---

## Workflows Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/workflows` | List all workflows |
| `POST` | `/workflows` | Create new workflow (INACTIVE) |
| `GET` | `/workflows/{id}` | Get specific workflow |
| `PUT` | `/workflows/{id}` | Update workflow (FULL REPLACEMENT) |
| `DELETE` | `/workflows/{id}` | Delete workflow |
| `POST` | `/workflows/{id}/activate` | Activate workflow |
| `POST` | `/workflows/{id}/deactivate` | Deactivate workflow |
| `PUT` | `/workflows/{id}/transfer` | Transfer to project (Enterprise) |
| `GET` | `/workflows/{id}/tags` | Get workflow tags |
| `PUT` | `/workflows/{id}/tags` | Update workflow tags (REPLACES all) |
| `GET` | `/workflows/{id}/{versionId}` | Get specific version |

### GET /workflows - Query Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `active` | boolean | No | Filter by active status |
| `tags` | string | No | Comma-separated tag names |
| `name` | string | No | Filter by workflow name (partial match) |
| `projectId` | string | No | Filter by project ID (Enterprise) |
| `excludePinnedData` | boolean | No | Avoid retrieving pinned data |

### POST /workflows - Complete Schema

```typescript
interface WorkflowNode {
  id: string;                    // Unique ID within workflow
  name: string;                  // Display name (must be unique)
  type: string;                  // Full type: "n8n-nodes-base.webhook"
  typeVersion: number;           // Node version (e.g., 1, 2)
  position: [number, number];    // [x, y] canvas coordinates
  parameters: Record<string, any>;  // Node-specific configuration
  credentials?: {                // Credential references
    [credentialType: string]: {
      id: string;
      name: string;
    }
  };
  webhookId?: string;            // For webhook nodes
  disabled?: boolean;            // Default: false
  notes?: string;                // Node notes
  notesInFlow?: boolean;         // Show notes in canvas
  executeOnce?: boolean;         // Execute only once
  alwaysOutputData?: boolean;    // Output even on error
  retryOnFail?: boolean;         // Enable retry
  maxTries?: number;             // Max retry attempts (default: 3)
  waitBetweenTries?: number;     // Retry delay in ms (default: 1000)
  onError?: 'stopWorkflow' | 'continueRegularOutput' | 'continueErrorOutput';
}

interface Connections {
  [sourceNodeName: string]: {
    main: Array<Array<{
      node: string;    // Target node name
      type: 'main';    // Connection type
      index: number;   // Input index (0 = first input)
    }>>;
    // For AI nodes:
    ai_languageModel?: Array<Array<ConnectionTarget>>;
    ai_tool?: Array<Array<ConnectionTarget>>;
    ai_memory?: Array<Array<ConnectionTarget>>;
    ai_outputParser?: Array<Array<ConnectionTarget>>;
  }
}

interface WorkflowSettings {
  saveExecutionProgress?: boolean;
  saveManualExecutions?: boolean;
  saveDataErrorExecution?: 'all' | 'none';
  saveDataSuccessExecution?: 'all' | 'none';
  executionTimeout?: number;             // Max: 3600 seconds
  errorWorkflow?: string;                // Workflow ID to run on error
  timezone?: string;                     // IANA timezone
  executionOrder?: 'v0' | 'v1';          // v1 recommended
  callerPolicy?: 'any' | 'none' | 'workflowsFromAList' | 'workflowsFromSameOwner';
  callerIds?: string;                    // Comma-separated workflow IDs
}
```

---

## Executions Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/executions` | List executions |
| `GET` | `/executions/{id}` | Get specific execution |
| `DELETE` | `/executions/{id}` | Delete execution |
| `POST` | `/executions/{id}/retry` | Retry failed execution |

### GET /executions - Query Parameters

| Name | Type | Description |
|------|------|-------------|
| `includeData` | boolean | Include execution's detailed data |
| `status` | string | `canceled`, `error`, `success`, `waiting` |
| `workflowId` | string | Filter by workflow ID |

⚠️ **KNOWN ISSUE:** `running` status is NOT accepted despite documentation.

⚠️ **KNOWN ISSUE:** Executions with `status: waiting` may not be returned.

### Execution Response Schema

```typescript
interface Execution {
  id: number;                    // Execution ID (number, not string)
  workflowId: string;            // ⚠️ STRING not number (schema bug)
  finished: boolean;
  mode: 'cli' | 'error' | 'integrated' | 'internal' | 'manual' |
        'retry' | 'trigger' | 'webhook' | 'evaluation' | 'chat';
  retryOf: string | null;
  retrySuccessId: string | null;
  startedAt: string;             // ISO 8601
  stoppedAt: string | null;
  waitTill: string | null;
  status: 'canceled' | 'crashed' | 'error' | 'new' | 'running' |
          'success' | 'unknown' | 'waiting';
  data?: {                       // Only if includeData=true
    resultData: {
      runData: {
        [nodeName: string]: Array<{
          startTime: number;
          executionTime: number;
          data: { main: Array<Array<{ json: any; binary?: any }>> };
        }>;
      };
      lastNodeExecuted: string;
      error?: ExecutionError;
    };
  };
}
```

---

## Credentials Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/credentials` | List credentials (metadata only) |
| `POST` | `/credentials` | Create credential |
| `PATCH` | `/credentials/{id}` | Update credential (owner only) |
| `DELETE` | `/credentials/{id}` | Delete credential (owner only) |
| `GET` | `/credentials/schema/{typeName}` | Get credential JSON schema |

⚠️ **IMPORTANT:** Returns ONLY metadata (id, name, type). Credential data/secrets are NEVER returned.

### Credential Type Examples

```json
// GitHub API
{"type": "githubApi", "data": {"accessToken": "your-github-pat"}}

// Slack API
{"type": "slackApi", "data": {"accessToken": "xoxb-xxx"}}

// HTTP Header Auth
{"type": "httpHeaderAuth", "data": {"name": "X-API-Key", "value": "xxx"}}

// HTTP Basic Auth
{"type": "httpBasicAuth", "data": {"user": "username", "password": "xxx"}}
```

---

## Other Endpoints

### Tags
`POST/GET /tags`, `GET/PUT/DELETE /tags/{id}`

### Users (Instance Owner Only)
`GET/POST /users`, `GET/DELETE /users/{id}`, `PATCH /users/{id}/role`

**Roles:** `global:owner`, `global:admin`, `global:member`

### Variables (Pro/Enterprise)
`GET/POST /variables`, `PUT/DELETE /variables/{id}`

### Projects (Enterprise)
`GET/POST /projects`, `PUT/DELETE /projects/{projectId}`

### Source Control
`POST /source-control/pull` - Pull from connected Git repository

### Audit
`POST /audit` - Generate security audit report

---

## Known Issues and Gotchas

### Issue #1: workflowId Type Mismatch (CRITICAL)
**Problem:** OpenAPI defines `execution.workflowId` as `type: number` but n8n returns STRING.
**Workaround:** Always treat `workflowId` as string.

### Issue #2: Waiting Executions Not Returned
**Problem:** `GET /executions` does not include `status: waiting`.
**Status:** Closed as "not planned".

### Issue #3: Webhook Registration Fails via API
**Problem:** Workflows activated via API may not register webhooks.
**Error:** `404 - The requested webhook is not registered`
**Workaround:** Save workflow via UI after API deployment.

### Issue #4: "running" Status Filter Rejected
**Problem:** Documentation shows "running" as valid but API rejects it.
**Workaround:** Use only: `canceled`, `error`, `success`, `waiting`.

### Issue #5: No Built-in Rate Limiting
**Problem:** n8n Public API has NO rate limiting.
**Workaround:** Implement via reverse proxy.

---

# PART 2: n8n MCP Server Reference

**Repository:** github.com/czlonkowski/n8n-mcp
**npm Package:** `n8n-mcp`
**Coverage:** 1,084 nodes (537 core + 547 community) | 2,709+ templates

## All 20 Tools Quick Reference

| # | Tool | Category | Requires API |
|---|------|----------|--------------|
| 1 | `tools_documentation` | Documentation | No |
| 2 | `search_nodes` | Documentation | No |
| 3 | `get_node` | Documentation | No |
| 4 | `validate_node` | Documentation | No |
| 5 | `validate_workflow` | Documentation | No |
| 6 | `search_templates` | Documentation | No |
| 7 | `get_template` | Documentation | No |
| 8 | `n8n_create_workflow` | Management | Yes |
| 9 | `n8n_get_workflow` | Management | Yes |
| 10 | `n8n_update_full_workflow` | Management | Yes |
| 11 | `n8n_update_partial_workflow` | Management | Yes |
| 12 | `n8n_delete_workflow` | Management | Yes |
| 13 | `n8n_list_workflows` | Management | Yes |
| 14 | `n8n_validate_workflow` | Management | Yes |
| 15 | `n8n_autofix_workflow` | Management | Yes |
| 16 | `n8n_test_workflow` | Management | Yes |
| 17 | `n8n_executions` | Management | Yes |
| 18 | `n8n_workflow_versions` | Management | Yes |
| 19 | `n8n_deploy_template` | Management | Yes |
| 20 | `n8n_health_check` | Management | Yes |

---

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MCP_MODE` | Server mode: `stdio` or `http` | `stdio` | No |
| `N8N_API_URL` | n8n instance URL | - | For management tools |
| `N8N_API_KEY` | n8n API key | - | For management tools |
| `AUTH_TOKEN` | Bearer token for HTTP mode | - | HTTP mode only |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error` | `info` | No |
| `DISABLE_CONSOLE_OUTPUT` | Suppress stdout for stdio | `false` | Recommended |
| `WEBHOOK_SECURITY_MODE` | `strict`, `moderate`, `disabled` | `strict` | No |

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

## Documentation Tools (1-7)

### 1. `tools_documentation`
Get documentation for MCP tools. **ALWAYS call first.**

```typescript
interface Params {
  toolName?: string;
  depth?: 'summary' | 'standard' | 'full';
}
```

### 2. `search_nodes`
Full-text search across 1,084 n8n nodes.

```typescript
interface Params {
  query: string;           // REQUIRED
  source?: 'all' | 'core' | 'community' | 'verified';
  includeExamples?: boolean;
  limit?: number;          // Default: 20
}
```

| Source | Nodes |
|--------|-------|
| `all` | 1,084 |
| `core` | 537 |
| `community` | 547 |
| `verified` | 301 |

### 3. `get_node`
Unified node information with multiple modes.

```typescript
interface Params {
  nodeType: string;        // REQUIRED: "n8n-nodes-base.httpRequest"
  mode?: 'info' | 'docs' | 'search_properties' | 'versions' | 'compare' | 'breaking' | 'migrations';
  detail?: 'minimal' | 'standard' | 'full';
  includeExamples?: boolean;
  propertyQuery?: string;  // For search_properties mode
}
```

**Detail Levels:**
| Level | Tokens | Content |
|-------|--------|---------|
| `minimal` | ~200 | Basic metadata only |
| `standard` | ~500-1000 | Essential properties (10-20) |
| `full` | ~3000-8000 | Complete with all properties |

### 4. `validate_node`
Validate node configuration against schema.

```typescript
interface Params {
  nodeType: string;        // REQUIRED
  config: Record<string, any>; // REQUIRED
  mode?: 'minimal' | 'full';
  profile?: 'minimal' | 'runtime' | 'ai-friendly' | 'strict';
}

interface Response {
  valid: boolean;
  errors: ValidationIssue[];
  warnings: ValidationIssue[];
  suggestions: ValidationIssue[];
  fixes?: AutoFix[];
}
```

### 5. `validate_workflow`
Complete workflow validation.

```typescript
interface Params {
  workflow: {
    name: string;
    nodes: WorkflowNode[];
    connections: ConnectionsObject;
    settings?: WorkflowSettings;
  };
}
```

**Common Error Codes:**
| Code | Description |
|------|-------------|
| `EMPTY_CONNECTIONS` | Multi-node workflow has no connections |
| `MISSING_TRIGGER` | No trigger node found |
| `AI_AGENT_MISSING_LLM` | AI Agent without language model |
| `INVALID_NODE_REFERENCE` | Expression references non-existent node |
| `CIRCULAR_DEPENDENCY` | Detected cycle in connections |

### 6. `search_templates`
Search 2,709+ workflow templates.

```typescript
interface Params {
  searchMode?: 'keyword' | 'by_nodes' | 'by_task' | 'by_metadata';
  query?: string;
  nodeTypes?: string[];
  task?: string;           // webhook_processing, slack_integration, ai_agent, etc.
  complexity?: 'simple' | 'medium' | 'complex';
  requiredService?: string;
  targetAudience?: 'developers' | 'marketers' | 'analysts' | 'operations';
  limit?: number;
}
```

### 7. `get_template`
Retrieve complete workflow template.

```typescript
interface Params {
  templateId: number;      // REQUIRED
  mode?: 'nodes_only' | 'structure' | 'full';
}
```

| Mode | Tokens |
|------|--------|
| `nodes_only` | ~200 |
| `structure` | ~500-1000 |
| `full` | ~2000-5000 |

---

## Management Tools (8-20)

### 8. `n8n_create_workflow`
Create new workflow (always INACTIVE).

```typescript
interface Params {
  name: string;            // REQUIRED
  nodes: WorkflowNode[];   // REQUIRED
  connections: ConnectionsObject; // REQUIRED
  settings?: WorkflowSettings;
}
```

### 9. `n8n_get_workflow`
Retrieve workflow with detail control.

```typescript
interface Params {
  id: string;              // REQUIRED
  mode?: 'full' | 'details' | 'structure' | 'minimal';
}
```

| Mode | Tokens |
|------|--------|
| `minimal` | ~100-200 |
| `structure` | ~500-1500 |
| `details` | ~2000-5000 |
| `full` | ~3000-8000 |

### 10. `n8n_update_full_workflow`
Complete workflow REPLACEMENT.

```typescript
interface Params {
  id: string;              // REQUIRED
  name?: string;
  nodes?: WorkflowNode[];
  connections?: ConnectionsObject;
  settings?: WorkflowSettings;
  intent?: string;
}
```

⚠️ **WARNING:** Omitted nodes/connections are DELETED.

### 11. `n8n_update_partial_workflow` ⭐ CRITICAL

> ⚠️ **IRON LAW: The n8n API only supports PUT (full replacement), not PATCH (partial merge).**
>
> **The name "partial" is MISLEADING.** This tool does NOT merge your changes with existing data.
>
> When using `updateNode` with `changes: { parameters: {...} }`:
> - ❌ WRONG assumption: "I'll send just the fields I want to change"
> - ✅ REALITY: The entire `parameters` object gets REPLACED
> - ⚠️ Parameters you don't include will be DELETED
>
> **Safe Pattern:**
> 1. `n8n_get_workflow()` - Get complete workflow with all node parameters
> 2. Extract the node's current `parameters` object
> 3. Modify only the specific field(s) you need
> 4. Send the COMPLETE parameters object back
> 5. Verify critical fields survived
>
> **Incident 2026-02-01:** Attempt to update just `jsonBody` on an HTTP node wiped `method`, `url`, `headers`, and `timeout`. Only workflow backup prevented production breakage.

Incremental workflow updates with **17 operation types**.

```typescript
interface Params {
  id: string;              // REQUIRED
  operations: Operation[]; // REQUIRED
  validateOnly?: boolean;  // Default: false (preview only)
  continueOnError?: boolean;
  intent?: string;
}
```

#### All 17 Operation Types

**Node Operations:**
```typescript
// 1. addNode
{ type: "addNode", node: { name, type, position?, parameters?, typeVersion?, credentials?, disabled? } }

// 2. removeNode
{ type: "removeNode", nodeName: string }

// 3. updateNode (dot notation for changes)
{ type: "updateNode", nodeName: string, changes: { "parameters.url": "https://...", "parameters.method": "POST" } }

// 4. moveNode
{ type: "moveNode", nodeName: string, position: [x, y] }

// 5. enableNode
{ type: "enableNode", nodeName: string }

// 6. disableNode
{ type: "disableNode", nodeName: string }
```

**Connection Operations:**
```typescript
// 7. addConnection
{ type: "addConnection", source: string, target: string, sourceOutput?: string, targetInput?: string, branch?: "true" | "false" }

// 8. removeConnection
{ type: "removeConnection", source: string, target: string, sourceOutput?: string, targetInput?: string }

// 9. rewireConnection
{ type: "rewireConnection", source: string, oldTarget: string, newTarget: string }

// 10. cleanStaleConnections
{ type: "cleanStaleConnections" }

// 11. replaceConnections
{ type: "replaceConnections", connections: ConnectionsObject }
```

**Metadata Operations:**
```typescript
// 12. updateSettings
{ type: "updateSettings", settings: Partial<WorkflowSettings> }

// 13. updateName
{ type: "updateName", name: string }

// 14. addTag
{ type: "addTag", tag: string }

// 15. removeTag
{ type: "removeTag", tag: string }
```

**State Operations:**
```typescript
// 16. activateWorkflow
{ type: "activateWorkflow" }

// 17. deactivateWorkflow
{ type: "deactivateWorkflow" }
```

**IF Node Routing:**
```typescript
// TRUE branch
{ type: "addConnection", source: "IF", target: "Success", branch: "true" }
// FALSE branch
{ type: "addConnection", source: "IF", target: "Failure", branch: "false" }
```

**AI Node Connections:**
```typescript
// Language Model
{ type: "addConnection", source: "OpenAI Chat", target: "AI Agent", sourceOutput: "ai_languageModel" }
// Tool
{ type: "addConnection", source: "HTTP Tool", target: "AI Agent", sourceOutput: "ai_tool" }
// Memory
{ type: "addConnection", source: "Memory", target: "AI Agent", sourceOutput: "ai_memory" }
```

### 12. `n8n_delete_workflow`
⚠️ PERMANENTLY deletes workflow and ALL execution history.

```typescript
interface Params { id: string }
```

### 13. `n8n_list_workflows`

```typescript
interface Params {
  limit?: number;          // Default: 100, Max: 100
  cursor?: string;
  active?: boolean;
  tags?: string[];         // AND logic
  projectId?: string;
}
```

### 14. `n8n_validate_workflow`
Validate deployed workflow by ID.

```typescript
interface Params {
  id: string;              // REQUIRED
  options?: {
    profile?: 'minimal' | 'runtime' | 'ai-friendly' | 'strict';
    checkCredentials?: boolean;
    checkExpressions?: boolean;
    checkConnections?: boolean;
  };
}
```

### 15. `n8n_autofix_workflow`
Automatically fix common validation errors.

```typescript
interface Params {
  id: string;              // REQUIRED
  applyFixes?: boolean;    // Default: false (preview only)
  fixTypes?: FixType[];
  confidenceThreshold?: 'high' | 'medium' | 'low';
  maxFixes?: number;       // Default: 50
}

type FixType =
  | 'expression-format'
  | 'typeversion-correction'
  | 'error-output-config'
  | 'node-type-correction'
  | 'webhook-missing-path'
  | 'typeversion-upgrade'
  | 'version-migration';
```

**Confidence Levels:**
| Level | Certainty | Use Case |
|-------|-----------|----------|
| `high` | 95%+ | Production (safest) |
| `medium` | 80%+ | Balanced (default) |
| `low` | All | Aggressive (review carefully) |

### 16. `n8n_test_workflow`
Trigger workflow execution.

```typescript
interface Params {
  workflowId: string;      // REQUIRED
  triggerType?: 'webhook' | 'form' | 'chat' | 'auto';
  httpMethod?: 'GET' | 'POST' | 'PUT' | 'DELETE';
  webhookPath?: string;
  message?: string;        // For chat triggers
  sessionId?: string;
  data?: Record<string, any>;
  headers?: Record<string, string>;
  timeout?: number;        // Default: 120000 (2 min)
  waitForResponse?: boolean;
}
```

⚠️ Workflow MUST be ACTIVE to be triggered.

### 17. `n8n_executions`
Unified execution management.

```typescript
interface Params {
  action: 'get' | 'list' | 'delete';
  id?: string;             // For get/delete
  mode?: 'preview' | 'summary' | 'filtered' | 'full';
  nodeNames?: string[];    // For filtered mode
  itemsLimit?: number;     // For filtered (default: 2)
  workflowId?: string;     // For list
  status?: 'success' | 'error' | 'waiting';
  limit?: number;
  cursor?: string;
}
```

### 18. `n8n_workflow_versions`
Version history and rollback.

```typescript
interface Params {
  mode: 'list' | 'get' | 'rollback' | 'delete' | 'prune' | 'truncate';
  workflowId?: string;
  versionId?: number;
  limit?: number;
  validateBefore?: boolean;  // For rollback (default: true)
  deleteAll?: boolean;       // For delete
  maxVersions?: number;      // For prune (default: 10)
  confirmTruncate?: boolean; // For truncate (REQUIRED: true)
}
```

### 19. `n8n_deploy_template`
Deploy template from n8n.io with auto-fixing.

```typescript
interface Params {
  templateId: number;      // REQUIRED
  name?: string;
  autoUpgradeVersions?: boolean; // Default: true
  autoFix?: boolean;       // Default: true
  stripCredentials?: boolean; // Default: true
}

interface Response {
  success: boolean;
  workflowId: string;
  active: false;           // Always inactive
  requiredCredentials: Array<{ nodeType, credentialType, nodeName }>;
  fixes: Array<{ type, description, applied }>;
  upgrades: Array<{ nodeName, oldVersion, newVersion }>;
}
```

### 20. `n8n_health_check`
Check n8n connectivity.

```typescript
interface Response {
  status: 'healthy' | 'unhealthy' | 'degraded';
  n8n: { connected, apiVersion, baseUrl, responseTime };
  mcp: { version, mode, features: { workflowManagement, templateLibrary, nodeValidation } };
  checks: { apiKey, apiAccess, workflows, executions };
}
```

---

## Error Handling

### MCP Protocol Error Codes (JSON-RPC 2.0)
| Code | Meaning |
|------|---------|
| `-32700` | Parse error |
| `-32600` | Invalid Request |
| `-32601` | Method not found |
| `-32602` | Invalid params |
| `-32603` | Internal error |

### Common Errors
| Error | Cause | Solution |
|-------|-------|----------|
| `Could not connect to MCP server` | Connection failure | Check URL, auth, network |
| `Unexpected token...` | JSON parsing in stdio | Set `DISABLE_CONSOLE_OUTPUT=true` |
| `N8N_API_TOKEN not configured` | Missing API key | Set `N8N_API_KEY` env var |

---

## Token Cost Estimates

| Operation | Tokens |
|-----------|--------|
| `get_node({ detail: 'minimal' })` | ~200 |
| `get_node({ detail: 'standard' })` | ~500-1000 |
| `get_node({ detail: 'full' })` | ~3000-8000 |
| `search_nodes()` (20 results) | ~2000 |
| `get_template({ mode: 'nodes_only' })` | ~200 |
| `get_template({ mode: 'full' })` | ~2000-5000 |
| `n8n_get_workflow({ mode: 'minimal' })` | ~100-200 |
| `n8n_get_workflow({ mode: 'full' })` | ~3000-8000 |

---

## Best Practices Workflow

1. `tools_documentation()` - Understand capabilities
2. `search_templates()` - Check 2,709 templates first
3. `search_nodes({ includeExamples: true })` - If no template fits
4. `get_node({ detail: 'standard' })` - Get essential properties
5. `validate_node({ mode: 'minimal' })` - Quick check (<100ms)
6. `validate_node({ mode: 'full', profile: 'runtime' })` - Full validation
7. Build workflow with validated configs
8. `validate_workflow()` - Complete workflow validation
9. `n8n_create_workflow()` - Create in n8n
10. `n8n_validate_workflow()` - Post-deployment validation
11. `n8n_autofix_workflow()` - Auto-fix issues
12. Activate: `n8n_update_partial_workflow({ operations: [{ type: "activateWorkflow" }] })`
13. `n8n_test_workflow()` - Test execution

### Batch Operations (Efficient)
```typescript
// ✅ GOOD - Single call with multiple operations
n8n_update_partial_workflow({
  id: "wf-123",
  operations: [
    { type: "updateNode", nodeName: "A", changes: {...} },
    { type: "updateNode", nodeName: "B", changes: {...} },
    { type: "addConnection", source: "A", target: "B" },
    { type: "cleanStaleConnections" }
  ]
})

// ❌ BAD - Multiple separate calls
n8n_update_partial_workflow({ id: "wf-123", operations: [...] })
n8n_update_partial_workflow({ id: "wf-123", operations: [...] })
```

---

## Anti-Hallucination Section

### API - DO NOT assume:
- ❌ Rate limiting exists - it does NOT
- ❌ workflowId is a number - it's a STRING
- ❌ "running" status filter works - it does NOT
- ❌ API automatically registers webhooks - it may NOT
- ❌ Credential data is returned - only metadata is returned
- ❌ Delete operations can be undone - they are PERMANENT

### MCP - DO NOT assume:
- ❌ Management tools work without `N8N_API_URL` and `N8N_API_KEY`
- ❌ Created workflows are active - they are ALWAYS inactive
- ❌ `n8n_update_full_workflow` merges changes - it REPLACES everything
- ❌ `n8n_update_partial_workflow` merges `parameters` - it REPLACES them (name is misleading!)
- ❌ `updateNode` with `changes: {parameters: {...}}` is smart - it will DELETE unlisted params
- ❌ Workflows can be triggered when inactive - they MUST be active
- ❌ All 20 tools require API config - 7 documentation tools work standalone

### Node Type Format:
- Search tools: `nodes-base.httpRequest` or `n8n-nodes-base.httpRequest`
- Workflow tools: ALWAYS `n8n-nodes-base.httpRequest`
- LangChain: `@n8n/n8n-nodes-langchain.lmChatOpenAi`

### ALWAYS verify:
- ✅ Use `n8n_health_check` to verify connectivity first
- ✅ Use `validateOnly: true` to preview partial updates
- ✅ Activate workflow before `n8n_test_workflow`
- ✅ Use full node type names with package prefix
- ✅ Include `typeVersion` when creating nodes
- ✅ Use `branch: "true"/"false"` for IF node connections

---

**Source Document:** `/Users/mikeboscia/My Drive/GTM Machine content/Research & Strategy/n8n REST API and MCP Server - Exhaustive Skill Reference Documentation.md`
