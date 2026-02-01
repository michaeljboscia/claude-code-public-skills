---
name: n8n-workflow-development
description: Comprehensive reference for building n8n workflows - typeVersions, expression syntax, node patterns, anti-patterns
shareable: true
---

# n8n Workflow Development - Comprehensive Reference

**OBJECTIVE:** Build n8n workflows with ZERO hallucinations. Every node type, typeVersion, and parameter structure must be verified against documentation.

**CRITICAL:** When in doubt, CHECK THE DOCS. Never guess webhook URLs, node names, or parameter structures.

---

## Documentation Hierarchy

When building or troubleshooting n8n workflows, follow this lookup order:

1. **THIS SKILL FILE** - Quick reference for common patterns
2. **LOCAL DOCUMENTATION** - Comprehensive technical references:
   - `/Users/mikeboscia/My Drive/GTM Machine content/n8n docs/n8n-node-implementation-reference-v2.md`
   - `/Users/mikeboscia/My Drive/GTM Machine content/n8n docs/n8n Workflow Automation Platform- Complete Technical Reference.md`
3. **LIVE n8n DOCS** - Official documentation:
   - https://docs.n8n.io/integrations/builtin/core-nodes/
   - https://docs.n8n.io/code/
   - https://docs.n8n.io/workflows/

**Never proceed without verification if you're uncertain.**

---

## ⛔ CRITICAL: Workflow Creation Anti-Pattern ⛔

**NEVER GENERATE n8n WORKFLOW JSON FROM SCRATCH. ALWAYS USE THE GOLDEN TEMPLATE APPROACH.**

### The Problem: API vs UI Validation Gap

The n8n API performs **looser validation** than the UI. The API will accept workflow JSON that the UI **cannot render** due to:
- Missing parameter schemas
- Invalid typeVersion combinations
- Incomplete node configurations
- Malformed connection structures

**This means:** Workflows created via MCP tools (`n8n_create_workflow`, `n8n_update_partial_workflow`) will often **fail to render** in the n8n UI even though the API accepts them.

### The Solution: Golden Template Approach

**ALWAYS use this method:**

1. **Find a working template workflow** that contains similar node types
2. **Fetch the complete workflow** using `n8n_get_workflow` with `mode: "full"`
3. **Modify ONLY VALUES** in parameters (URLs, field names, expressions)
4. **NEVER modify STRUCTURE** (nesting, parameter keys, connection patterns)
5. **Preserve all typeVersions** exactly as they appear in the template

**Example - Correct Value Modification:**
```json
// Template has:
{
  "parameters": {
    "url": "https://api.example.com/users",
    "method": "GET"
  }
}

// Modify to:
{
  "parameters": {
    "url": "https://api.instantly.ai/campaign/create",  // ✅ Value changed
    "method": "POST"                                     // ✅ Value changed
  }
}

// ❌ WRONG - Changing structure:
{
  "parameters": {
    "endpoint": "https://api.instantly.ai/campaign/create",  // ❌ Key renamed
    "httpMethod": "POST"                                     // ❌ Key renamed
  }
}
```

### When Creating Workflows

**DO:**
- Use existing workflow as template
- Copy entire node structures
- Modify only leaf values (strings, numbers, booleans)
- Preserve all nesting and keys
- Test by importing manually if unsure

**DON'T:**
- Generate workflow JSON from scratch
- Use MCP create/update tools without template
- Modify parameter keys or structure
- Change typeVersion numbers
- Guess at node configurations

### Reference Documentation

See: `/Users/mikeboscia/My Drive/GTM Machine content/n8n/n8n-workflow-api-antipattern-spec.md` for complete technical analysis.

---

## ⛔ CRITICAL: typeVersion Reference ⛔

**STOP. Every node MUST use these exact typeVersion values. Using outdated versions causes silent failures.**

| Node Type | typeVersion | ⛔ DO NOT USE |
|-----------|-------------|---------------|
| `n8n-nodes-base.if` | **2.3** | ~~1, 2, 2.1, 2.2~~ |
| `n8n-nodes-base.switch` | **3.4** | ~~1, 2, 3, 3.1, 3.2, 3.3~~ |
| `n8n-nodes-base.merge` | **3.2** | ~~1, 2, 2.1, 3, 3.1~~ |
| `n8n-nodes-base.splitInBatches` | **3** | ~~1, 2~~ |
| `n8n-nodes-base.splitOut` | **1** | (only version) |
| `n8n-nodes-base.set` | **3.4** | ~~1, 2, 3, 3.1, 3.2, 3.3~~ |
| `n8n-nodes-base.code` | **2** | ~~1~~ |
| `n8n-nodes-base.filter` | **2.3** | ~~1, 2, 2.1, 2.2~~ |
| `n8n-nodes-base.wait` | **1.1** | ~~1~~ |
| `n8n-nodes-base.executeWorkflow` | **1.3** | ~~1, 1.1, 1.2~~ |
| `n8n-nodes-base.executeWorkflowTrigger` | **1.1** | ~~1~~ |
| `n8n-nodes-base.errorTrigger` | **1** | (only version) |
| `n8n-nodes-base.stopAndError` | **1** | (only version) |
| `n8n-nodes-base.httpRequest` | **4.3** | ~~1, 2, 3, 4, 4.1, 4.2~~ |
| `n8n-nodes-base.webhook` | **2.1** | ~~1, 1.1, 2~~ |
| `n8n-nodes-base.scheduleTrigger` | **1.3** | ~~1, 1.1, 1.2~~ |
| `n8n-nodes-base.manualTrigger` | **1** | (only version) |
| `n8n-nodes-base.respondToWebhook` | **1.5** | ~~1, 1.1, 1.2, 1.3, 1.4~~ |
| `n8n-nodes-base.noOp` | **1** | (only version) |
| `n8n-nodes-base.aggregate` | **1** | (only version) |

### Quick Copy Templates

```json
// IF Node - MUST be typeVersion 2.3
{"type": "n8n-nodes-base.if", "typeVersion": 2.3}

// Switch Node - MUST be typeVersion 3.4
{"type": "n8n-nodes-base.switch", "typeVersion": 3.4}

// Merge Node - MUST be typeVersion 3.2
{"type": "n8n-nodes-base.merge", "typeVersion": 3.2}

// HTTP Request - MUST be typeVersion 4.3
{"type": "n8n-nodes-base.httpRequest", "typeVersion": 4.3}

// Set/Edit Fields - MUST be typeVersion 3.4
{"type": "n8n-nodes-base.set", "typeVersion": 3.4}

// Code Node - MUST be typeVersion 2
{"type": "n8n-nodes-base.code", "typeVersion": 2}

// Filter - MUST be typeVersion 2.3
{"type": "n8n-nodes-base.filter", "typeVersion": 2.3}
```

---

## ⛔ Deprecated Parameter Structures - DO NOT USE ⛔

### IF Node: v1 vs v2.3 Structure

**❌ WRONG - v1 structure (DEPRECATED):**
```json
{
  "type": "n8n-nodes-base.if",
  "typeVersion": 1,
  "parameters": {
    "conditions": {
      "string": [
        {
          "value1": "={{$json.status}}",
          "operation": "equals",
          "value2": "active"
        }
      ]
    }
  }
}
```

**✅ CORRECT - v2.3 structure:**
```json
{
  "type": "n8n-nodes-base.if",
  "typeVersion": 2.3,
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "typeValidation": "strict"
      },
      "conditions": [
        {
          "id": "condition-uuid",
          "leftValue": "={{ $json.status }}",
          "rightValue": "active",
          "operator": {
            "type": "string",
            "operation": "equals"
          }
        }
      ],
      "combinator": "and"
    }
  }
}
```

### Switch Node: v1/v2 vs v3.4 Structure

**❌ WRONG - v1/v2 structure:**
```json
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 1,
  "parameters": {
    "dataType": "string",
    "value1": "={{$json.type}}",
    "rules": {
      "rules": [
        {"value2": "email", "output": 0},
        {"value2": "sms", "output": 1}
      ]
    }
  }
}
```

**✅ CORRECT - v3.4 structure:**
```json
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3.4,
  "parameters": {
    "mode": "rules",
    "rules": {
      "rules": [
        {
          "outputKey": "email",
          "conditions": {
            "options": {"caseSensitive": true, "typeValidation": "strict"},
            "conditions": [
              {
                "leftValue": "={{ $json.type }}",
                "rightValue": "email",
                "operator": {"type": "string", "operation": "equals"}
              }
            ],
            "combinator": "and"
          }
        }
      ]
    },
    "options": {}
  }
}
```

### Set/Edit Fields Node: v2 vs v3.4 Structure

**❌ WRONG - v2 structure:**
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 2,
  "parameters": {
    "values": {
      "string": [
        {"name": "fullName", "value": "={{$json.firstName}} {{$json.lastName}}"}
      ]
    }
  }
}
```

**✅ CORRECT - v3.4 structure:**
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "parameters": {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {
          "id": "uuid-1",
          "name": "fullName",
          "value": "={{ $json.firstName }} {{ $json.lastName }}",
          "type": "string"
        }
      ]
    },
    "includeOtherFields": false,
    "options": {}
  }
}
```

### HTTP Request Node: v3 vs v4.3 Structure

**❌ WRONG - v3 structure:**
```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 3,
  "parameters": {
    "requestMethod": "POST",
    "url": "https://api.example.com",
    "jsonParameters": true,
    "bodyParametersJson": "{\"key\": \"value\"}"
  }
}
```

**✅ CORRECT - v4.3 structure:**
```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3,
  "parameters": {
    "method": "POST",
    "url": "https://api.example.com",
    "authentication": "none",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ { \"key\": \"value\" } }}",
    "options": {}
  }
}
```

### Code Node: v1 vs v2 Structure

**❌ WRONG - v1 structure:**
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 1,
  "parameters": {
    "jsCode": "return items.map(item => { item.json.processed = true; return item; });"
  }
}
```

**✅ CORRECT - v2 structure:**
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "mode": "runOnceForAllItems",
    "jsCode": "return $input.all().map(item => ({\n  json: {\n    ...item.json,\n    processed: true\n  }\n}));"
  }
}
```

---

## Expression Syntax Rules

**All expressions MUST:**
1. Start with `=` followed by `{{ }}`
2. Use `$json.field` NOT `$json["field"]` for simple access
3. Have a space after `={{` and before `}}`

```javascript
// ✅ CORRECT
"={{ $json.status }}"
"={{ $json.user.email }}"
"={{ $json.items[0].name }}"
"={{ $now.toISO() }}"

// ❌ WRONG
"{{$json.status}}"           // Missing =
"=$json.status"              // Missing {{ }}
"={{$json.status}}"          // Missing spaces (works but not canonical)
```

---

## Data Structure Fundamentals

All data flows through n8n as **arrays of items**:

```javascript
[
  { "json": { "name": "John", "email": "john@example.com" }, "binary": {} },
  { "json": { "name": "Jane", "email": "jane@example.com" }, "binary": {} }
]
```

### Built-in Expression Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `$json` | Current item's JSON data | `{{ $json.email }}` |
| `$input.all()` | All input items as array | `{{ $input.all() }}` |
| `$input.first()` | First input item | `{{ $input.first().json.name }}` |
| `$input.last()` | Last input item | `{{ $input.last().json.id }}` |
| `$('NodeName')` | Access another node's output | `{{ $('Webhook').first().json.body }}` |
| `$execution.id` | Current execution ID | `{{ $execution.id }}` |
| `$execution.resumeUrl` | Resume URL for Wait node | `{{ $execution.resumeUrl }}` |
| `$workflow.name` | Workflow name | `{{ $workflow.name }}` |
| `$now` | Current Luxon DateTime | `{{ $now.toFormat('yyyy-MM-dd') }}` |
| `$today` | Today at midnight | `{{ $today }}` |
| `$itemIndex` | Current item's index | `{{ $itemIndex }}` |
| `$runIndex` | Current run index in loops | `{{ $runIndex }}` |

---

## Workflow JSON Structure

Every n8n workflow follows this structure:

```json
{
  "name": "Workflow Name",
  "nodes": [
    {
      "id": "unique-uuid",
      "name": "Node Display Name",
      "type": "n8n-nodes-base.nodeType",
      "typeVersion": 2,
      "position": [400, 300],
      "parameters": {}
    }
  ],
  "connections": {
    "Source Node": {
      "main": [
        [{"node": "Target Node", "type": "main", "index": 0}]
      ]
    }
  },
  "settings": {"executionOrder": "v1"},
  "active": false
}
```

**Key points:**
- `connections` uses **node names** as keys (NOT node IDs)
- Array-of-arrays pattern: outer array = outputs, inner arrays = multiple connections
- Data flows as arrays of objects with `json` and `binary` keys

---

## Core Flow Control Nodes

### IF Node - Binary Branching

**Use for:** True/false decisions (2 outputs)

**Outputs:**
- Output 0: True branch
- Output 1: False branch

**Connection Pattern:**
```json
"connections": {
  "Check Status": {
    "main": [
      [{"node": "Process Active", "type": "main", "index": 0}],
      [{"node": "Handle Inactive", "type": "main", "index": 0}]
    ]
  }
}
```

**Available Operators by Type:**
- **String**: equals, notEquals, contains, notContains, startsWith, endsWith, regex, isEmpty, isNotEmpty
- **Number**: equals, notEquals, gt, gte, lt, lte
- **Boolean**: true, false
- **Array**: contains, notContains, lengthEquals, lengthGt, isEmpty, isNotEmpty

### Switch Node - Multi-Branch Routing

**Use for:** Routing to 3+ destinations or when you need named output branches

**Modes:**
- **Rules Mode**: Condition-based routing
- **Expression Mode**: Programmatic output calculation

**Decision:** Use IF for binary decisions; use Switch for 3+ destinations.

### Merge Node - Combining Data Streams

**Critical Modes:**

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Append** | Concatenates all items into single list | Combining parallel branch results |
| **Combine by Fields** | Joins items matching field values | SQL-like JOIN operations |
| **Combine by Position** | Pairs items by array index | Matching ordered datasets |
| **Choose Branch** | Outputs from selected input only | Conditional branch selection |

**Connection Pattern (multiple inputs):**
```json
"connections": {
  "Get Users": {
    "main": [[{"node": "Merge Data", "type": "main", "index": 0}]]
  },
  "Get Orders": {
    "main": [[{"node": "Merge Data", "type": "main", "index": 1}]]
  }
}
```

### Loop Over Items (Split in Batches)

**Use for:**
- Processing items in batches
- Rate-limited APIs (set batchSize: 1)
- Nodes that only process first item

**Parameters:**
- `batchSize`: Items per iteration
- `options.reset`: Treat incoming data as new set

**Outputs:**
- Output 0: Loop (connect back to processing)
- Output 1: Done (continue workflow)

**Loop Connection Pattern:**
```json
"connections": {
  "Loop Over Items": {
    "main": [
      [{"node": "HTTP Request", "type": "main", "index": 0}],
      [{"node": "Final Output", "type": "main", "index": 0}]
    ]
  },
  "HTTP Request": {
    "main": [[{"node": "Loop Over Items", "type": "main", "index": 0}]]
  }
}
```

**Context Variables:**
```javascript
{{ $("Loop Over Items").context["noItemsLeft"] }}     // boolean
{{ $("Loop Over Items").context["currentRunIndex"] }} // number
```

⚠️ **Critical:** Always include termination condition to prevent infinite loops.

---

## Data Transformation Nodes

### Edit Fields (Set) Node

**Use for:** Creating, modifying, or removing fields without code

**Modes:**
- **Manual Mapping**: GUI-based field configuration
- **JSON Output**: Write JSON directly

**Key Options:**
- `includeOtherFields`: Keep (true) or discard (false) other input fields
- `duplicateItem`: Create copy or modify in place
- `options.dotNotation`: Enable `parent.child` syntax

### Code Node

**Execution Modes:**

**Run Once for All Items** (default):
```javascript
const items = $input.all();
const total = items.reduce((sum, item) => sum + item.json.amount, 0);
return [{ json: { total } }];
```

**Run Once for Each Item**:
```javascript
return {
  json: {
    ...($json),
    processed: true,
    fullName: `${$json.firstName} ${$json.lastName}`
  }
};
```

**Built-in Variables:**
- `$input.all()` - All items array
- `$input.first()` - First item
- `$json` - Current item's json (in "each item" mode)
- `$('NodeName').all()` - All items from specific node
- `$now` - Luxon DateTime
- `$workflow.name`, `$execution.id`

**Environment Limitations:**
- ❌ No file system access (use Read/Write Files nodes)
- ❌ No HTTP requests (use HTTP Request node)
- ❌ No DOM access (server-side only)
- ✅ Luxon available for dates
- ✅ console.log() outputs to browser console
- ✅ async/await supported

### Filter Node

**Use for:** Removing items that don't match conditions (single stream, no branching)

**Decision:** Use Filter to remove items from flow. Use IF when you need to process non-matching items differently.

---

## HTTP & Trigger Nodes

### HTTP Request Node

**Authentication Types:**
- None, Basic Auth, Digest Auth, Header Auth, Bearer Token
- OAuth1, OAuth2 (Authorization Code, Client Credentials, PKCE)
- Query Auth, Custom Auth

**Pagination Modes:**
- Response Contains Next URL
- Response Contains Page Number
- Update a Parameter (custom with expressions)

**Common Errors:**

| Error | Solution |
|-------|----------|
| `400 Bad Request` | Verify query param names; use Array Format option |
| `403 Forbidden` | Check credentials, permissions, rate limits |
| `429 Rate Limit` | Implement batching with Loop + Wait |
| `JSON parameter need to be valid JSON` | Ensure expressions return objects, not strings |

### PhantomBuster Integration Pattern (CRITICAL)

**⚠️ ALWAYS use `bonusArgument`, NEVER use `argument` when launching PhantomBuster agents.**

**The Problem:**
PhantomBuster agents store configuration (including LinkedIn `sessionCookie`) in the dashboard. When launching via API using `argument`, it **replaces the entire config**, wiping out the saved `sessionCookie`. This causes "Missing cookie" errors even when properly configured.

**The Solution:**
Use `bonusArgument` which **merges** with saved config instead of replacing it.

**Correct HTTP Request Configuration:**
```json
{
  "name": "Launch PhantomBuster",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3,
  "parameters": {
    "method": "POST",
    "url": "https://api.phantombuster.com/api/v2/agents/launch",
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "phantombusterApi",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {
          "name": "X-Phantombuster-Key-1",
          "value": "={{ $credentials.apiKey }}"
        }
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ {\n  \"id\": \"970262183893333\",\n  \"bonusArgument\": {\n    \"spreadsheetUrl\": $json.sheetUrl,\n    \"columnName\": \"profileUrl\",\n    \"message\": \"#customMessage#\"\n  }\n} }}"
  }
}
```

**Field Mapping Requirements:**
PhantomBuster expects specific field names in Google Sheets. Map your database fields in Code nodes:

| Your Field | PB Expected Field | Use Case |
|-----------|------------------|----------|
| `linkedin_url` | `profileUrl` | All LinkedIn actions |
| `message` | `customMessage` | LinkedIn messages |
| `connection_note` | `connectionNote` | Connection requests |

**Code Node Example:**
```javascript
// Map database fields to PhantomBuster format
const rows = [];
for (const item of $input.all()) {
  rows.push({
    profileUrl: item.json.action_data.linkedin_url,  // NOT linkedin_url
    customMessage: item.json.action_data.message      // NOT message
  });
}
return [{ json: { rows } }];
```

**Google Sheets Header Preservation:**
Configure Clear nodes with range `A2:Z1000` to preserve row 1 headers, preventing "column not found" errors.

**See:** `~/.claude/skills/phantombuster.md` for complete integration details.

### Webhook Node

**Creates HTTP endpoints** to trigger workflows or receive data mid-workflow.

**Key Configuration:**
- `httpMethod`: GET, POST, PUT, PATCH, DELETE, HEAD, or All
- `path`: URL path appended to webhook base
- `authentication`: None, Basic Auth, Header Auth, JWT Auth
- `responseMode`: Immediately, When Last Node Finishes, Using Respond to Webhook node

**URL Types:**
- **Test URL**: Development only, active during "Listen for Test Event"
- **Production URL**: Active only when workflow is published/active

⚠️ **Cloud Limit:** Webhook timeout is 100 seconds (Cloudflare limit)

### Schedule Trigger

**Trigger Intervals:** Seconds, Minutes, Hours, Days, Weeks, Months, Custom (Cron)

**Cron Examples:**
```
0 8 * * *        # 8:00 AM daily
30 8 * * 1-5     # 8:30 AM weekdays
0 0 1 * *        # Midnight on 1st of month
*/15 * * * *     # Every 15 minutes
```

⚠️ **Critical:** Workflow must be **saved AND published** for Schedule Trigger to execute.

---

## Error Handling

### Error Trigger Node

**Captures workflow failures** and triggers error handling workflow.

**Setup:**
1. Create error workflow with Error Trigger as first node
2. In monitored workflow: Settings → Error Workflow → Select error workflow

**Error Data Structure:**
```javascript
{
  "execution": {
    "id": "abc123",
    "url": "https://...",
    "error": { "message": "...", "stack": "..." },
    "lastNodeExecuted": "HTTP Request",
    "mode": "trigger"
  },
  "workflow": { "id": "xyz", "name": "My Workflow" }
}
```

⚠️ **Critical:** Error Trigger only fires for **automatic** workflow executions, not manual runs.

### Per-Node Error Settings

| Setting | Options |
|---------|---------|
| **On Error** | Stop Workflow, Continue, Continue (using error output) |
| **Retry On Fail** | Enable with Max Tries and Wait Between Tries |

### Stop And Error Node

**Deliberately fails workflow** with custom error message—useful for validation failures.

---

## Execute Sub-workflows

**Calls another workflow** as a sub-workflow for code reuse and memory optimization.

**Reference Methods:**
- **Database ID**: Workflow ID from URL
- **URL**: Full workflow URL
- **From Local File**: JSON file path
- **By Parameter**: Inline JSON

**Sub-workflow uses Execute Workflow Trigger node** with input options:
- Define using fields
- Define using JSON example
- Accept all data

⚠️ Sub-workflow executions **don't count** toward Cloud execution limits.

---

## Wait Node - Execution Pause

**Pauses execution** until resume condition is met.

**Resume Operations:**
- **After Time Interval**: Wait seconds/minutes/hours/days
- **At Specified Time**: Wait until specific datetime
- **On Webhook Call**: Wait for HTTP request to `$execution.resumeUrl`
- **On Form Submitted**: Wait for form completion

**Critical Webhook Resume Pattern:**

Send `$execution.resumeUrl` to external service **before** Wait node:

```json
{
  "name": "Register Callback",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3,
  "parameters": {
    "url": "https://approval-service.com/register",
    "method": "POST",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ { \"callbackUrl\": $execution.resumeUrl, \"executionId\": $execution.id } }}"
  }
}
```

**Common Mistakes:**
- Accessing `$execution.resumeUrl` after Wait node (URL only valid before pause)
- n8n Cloud 100-second timeout

---

## ❌ Anti-Patterns and Nodes That DON'T EXIST

### Nodes That DON'T EXIST (AI Hallucinations)

- ~~"Google Trends" node~~ → Use HTTP Request
- ~~"SerpApi" standalone node~~ → Only exists as AI Agent tool sub-node
- ~~"Cron Trigger"~~ → Use **Schedule Trigger**
- ~~"Start" node~~ → **Removed in v2.0**, use Manual Trigger

### Deprecated Syntax

| Deprecated | Current Replacement |
|------------|---------------------|
| Function node | **Code** node |
| Function Item node | **Code** node (Run Once for Each Item) |
| Item Lists node | **Split Out**, **Aggregate**, **Sort**, **Remove Duplicates** |
| `items` global variable | `$input.all()` |
| `$evaluateExpression()` | Direct expressions in Code node |

### Wrong Expression Syntax

```javascript
// ❌ WRONG: Missing braces
$json.name

// ✅ CORRECT
{{ $json.name }}

// ❌ WRONG: Single equals in condition
{{ $if($json.status = "active", "yes", "no") }}

// ✅ CORRECT: Double equals
{{ $if($json.status == "active", "yes", "no") }}
```

### Code Node Mistakes

```javascript
// ❌ WRONG: Old Function node syntax
items[0].json.value = "test";
return items;

// ✅ CORRECT: Code node syntax
const items = $input.all();
items[0].json.value = "test";
return items;
```

---

## Node Selection Decision Matrix

| Scenario | Recommended Node | Alternative |
|----------|------------------|-------------|
| Binary yes/no routing | IF | — |
| Route to 3+ destinations | Switch | Nested IF (less clean) |
| Combine parallel branches | Merge | — |
| Process one item at a time | Loop Over Items | — |
| Rate-limit API calls | Loop Over Items + Wait | HTTP batching option |
| Custom data transformation | Code | Edit Fields (for simple cases) |
| Remove items from stream | Filter | IF (if branches needed) |
| Pause for external callback | Wait (webhook mode) | — |
| Reusable workflow logic | Execute Sub-workflow | — |
| Centralized error notifications | Error Trigger | — |
| Validate and fail gracefully | Stop and Error | Code (throw) |
| Call external APIs | HTTP Request | — |
| Receive external events | Webhook | — |
| Time-based automation | Schedule Trigger | — |

---

## Complete Working Example: API Integration with Error Handling

```json
{
  "name": "Resilient API Sync",
  "nodes": [
    {
      "id": "sched-1",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.3,
      "position": [250, 300],
      "parameters": {
        "rule": {"interval": [{"field": "hours", "hoursInterval": 1}]}
      }
    },
    {
      "id": "http-1",
      "name": "Fetch Data",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [450, 300],
      "parameters": {
        "url": "https://api.example.com/data",
        "method": "GET",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {"timeout": 30000}
      },
      "onError": "continueErrorOutput",
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "id": "if-1",
      "name": "Check Success",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [650, 250],
      "parameters": {
        "conditions": {
          "options": {"typeValidation": "strict"},
          "conditions": [
            {
              "id": "cond-1",
              "leftValue": "={{ $json.status }}",
              "rightValue": "ok",
              "operator": {"type": "string", "operation": "equals"}
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "code-1",
      "name": "Process Data",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [850, 200],
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "return $input.all().map(item => ({json: {...item.json, processedAt: $now.toISO()}}));"
      }
    },
    {
      "id": "set-1",
      "name": "Log Error",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [850, 350],
      "parameters": {
        "mode": "manual",
        "assignments": {
          "assignments": [
            {"id": "a1", "name": "error", "value": "={{ $json.error || 'Unknown error' }}", "type": "string"}
          ]
        }
      }
    }
  ],
  "connections": {
    "Schedule Trigger": {"main": [[{"node": "Fetch Data", "type": "main", "index": 0}]]},
    "Fetch Data": {
      "main": [
        [{"node": "Check Success", "type": "main", "index": 0}],
        [{"node": "Log Error", "type": "main", "index": 0}]
      ]
    },
    "Check Success": {
      "main": [
        [{"node": "Process Data", "type": "main", "index": 0}],
        [{"node": "Log Error", "type": "main", "index": 0}]
      ]
    }
  },
  "settings": {"executionOrder": "v1"}
}
```

---

## Verification Protocol - When to Check Documentation

**ALWAYS check documentation before:**
1. Using a node type you haven't used in this session
2. Creating a webhook (NEVER guess webhook URLs)
3. Using a typeVersion you're uncertain about
4. Writing complex Code node logic
5. Troubleshooting unexpected behavior

**Verification Steps:**
1. Check THIS skill file for the pattern
2. If uncertain, read the LOCAL documentation files
3. If still uncertain, search live n8n docs at https://docs.n8n.io/
4. If still uncertain, ASK THE USER

**Never proceed with guessed configurations.**

---

## Common Issues and Solutions

### Issue: IF node with Merge causes both branches to execute
**Cause:** Workflow using v0 execution order
**Solution:** Set workflow settings to `"executionOrder": "v1"`

### Issue: Loop Over Items creates infinite loop
**Cause:** Missing termination condition or `reset: true` without exit logic
**Solution:** Add IF node with exit condition or remove reset option

### Issue: Code node "Cannot read property 'json' of undefined"
**Cause:** Accessing non-existent node
**Solution:** Verify node name spelling exactly matches

### Issue: HTTP Request "JSON parameter need to be valid JSON"
**Cause:** Expression returns string instead of object
**Solution:** Ensure jsonBody expression returns actual object: `={{ { "key": "value" } }}`

### Issue: Webhook returns 404
**Cause:** Workflow not active or wrong n8n instance
**Solution:**
1. Verify workflow is ACTIVE in n8n
2. Verify correct n8n instance (LOCAL vs CLOUD)
3. Check webhook path exactly matches

---

## Production Best Practices

1. **Error resilience**: Always configure error workflows for production
2. **Rate limiting**: Use Loop Over Items + Wait for API rate limits
3. **Memory management**: Use Execute Sub-workflow for large datasets
4. **Credential security**: Never hardcode API keys, use n8n credentials
5. **Testing strategy**: Use Manual Trigger during development, switch to production triggers
6. **Version pinning**: Document typeVersion of critical nodes

---

## Related Documentation

- **Local Node Implementation Reference:** `/Users/mikeboscia/My Drive/GTM Machine content/n8n docs/n8n-node-implementation-reference-v2.md`
- **Local Complete Technical Reference:** `/Users/mikeboscia/My Drive/GTM Machine content/n8n docs/n8n Workflow Automation Platform- Complete Technical Reference.md`
- **Live n8n Docs:** https://docs.n8n.io/
- **n8n Troubleshooting Skill:** `~/.claude/skills/n8n-troubleshooting.md`

---

*Last updated: 2026-01-19*
*Source documents: n8n-node-implementation-reference-v2.md, n8n Workflow Automation Platform- Complete Technical Reference.md, Outreach Orchestrator testing 2026-01-19*
