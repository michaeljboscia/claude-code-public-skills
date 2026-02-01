# Complete n8n Node Implementation Reference

---

## ⛔ CRITICAL: MANDATORY typeVersion REFERENCE ⛔

**STOP. READ THIS FIRST. EVERY NODE MUST USE THESE EXACT typeVersion VALUES.**

Any AI/LLM generating n8n workflow JSON MUST use the versions below. Using outdated versions (especially v1) will cause silent failures, missing parameters, and broken workflows.

### Current typeVersion Values (January 2026)

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

### Quick Copy-Paste Node Templates

```json
// IF Node - MUST be typeVersion 2.3
{
  "type": "n8n-nodes-base.if",
  "typeVersion": 2.3
}

// Switch Node - MUST be typeVersion 3.4
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3.4
}

// Merge Node - MUST be typeVersion 3.2
{
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3.2
}

// HTTP Request - MUST be typeVersion 4.3
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3
}

// Set/Edit Fields - MUST be typeVersion 3.4
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4
}

// Code Node - MUST be typeVersion 2
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2
}

// Webhook - MUST be typeVersion 2.1
{
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2.1
}

// Filter - MUST be typeVersion 2.3
{
  "type": "n8n-nodes-base.filter",
  "typeVersion": 2.3
}

// Schedule Trigger - MUST be typeVersion 1.3
{
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.3
}

// Split In Batches / Loop - MUST be typeVersion 3
{
  "type": "n8n-nodes-base.splitInBatches",
  "typeVersion": 3
}
```

---

## ⛔ DEPRECATED PARAMETER STRUCTURES - DO NOT USE ⛔

### IF Node: v1 vs v2+ Parameter Structure

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

### Switch Node: v1/v2 vs v3+ Parameter Structure

**❌ WRONG - v1/v2 structure (DEPRECATED):**
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

### Set/Edit Fields Node: v2 vs v3+ Parameter Structure

**❌ WRONG - v2 structure (DEPRECATED):**
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 2,
  "parameters": {
    "values": {
      "string": [
        {"name": "fullName", "value": "={{$json.firstName}} {{$json.lastName}}"}
      ]
    },
    "options": {}
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

### HTTP Request Node: v3 vs v4+ Parameter Structure

**❌ WRONG - v3 structure (DEPRECATED):**
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

### Code Node: v1 vs v2 Parameter Structure

**❌ WRONG - v1 structure (DEPRECATED):**
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

### Filter Node: v1 vs v2+ Parameter Structure

**❌ WRONG - v1 structure (DEPRECATED):**
```json
{
  "type": "n8n-nodes-base.filter",
  "typeVersion": 1,
  "parameters": {
    "conditions": {
      "string": [
        {"value1": "={{$json.status}}", "operation": "equals", "value2": "active"}
      ]
    }
  }
}
```

**✅ CORRECT - v2.3 structure:**
```json
{
  "type": "n8n-nodes-base.filter",
  "typeVersion": 2.3,
  "parameters": {
    "conditions": {
      "options": {"typeValidation": "strict"},
      "conditions": [
        {
          "leftValue": "={{ $json.status }}",
          "rightValue": "active",
          "operator": {"type": "string", "operation": "equals"}
        }
      ],
      "combinator": "and"
    },
    "looseTypeValidation": false
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

## Workflow JSON Foundation

Every n8n workflow follows this structure, with `connections` defining data flow using node names as keys:

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

The **connections object** uses an array-of-arrays pattern where the outer array represents outputs (index 0 = first output, index 1 = second output for IF nodes) and inner arrays allow multiple connections from the same output. Data flows as arrays of objects: `[{"json": {...}, "binary": {...}}]`.

---

## Flow Control Nodes

### IF Node: Binary conditional branching

The IF node routes items to **true** (output 0) or **false** (output 1) branches based on conditions. It supports string, number, date, boolean, array, and object comparisons with AND/OR combinators.

```json
{
  "id": "if-node-uuid",
  "name": "Check Status",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2.3,
  "position": [500, 300],
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "typeValidation": "strict"
      },
      "conditions": [
        {
          "id": "condition-1",
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

**Connection configuration for true/false branches:**

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

**Common mistake**: Using IF + Merge in workflows with v0 execution order causes both branches to execute. **Fix**: Ensure workflow settings use `"executionOrder": "v1"`.

**Available operators by type**:
- **String**: equals, notEquals, contains, notContains, startsWith, notStartsWith, endsWith, notEndsWith, regex, notRegex, isEmpty, isNotEmpty
- **Number**: equals, notEquals, gt, gte, lt, lte
- **Boolean**: true, false
- **Array**: contains, notContains, lengthEquals, lengthNotEquals, lengthGt, lengthGte, lengthLt, lengthLte, isEmpty, isNotEmpty

### Switch Node: Multi-branch routing

Switch routes to multiple outputs using **Rules Mode** (condition-based) or **Expression Mode** (programmatic index calculation).

```json
{
  "id": "switch-node-uuid",
  "name": "Route by Type",
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3.4,
  "position": [500, 300],
  "parameters": {
    "mode": "rules",
    "rules": {
      "rules": [
        {
          "outputKey": "email",
          "conditions": {
            "options": {
              "caseSensitive": true,
              "typeValidation": "strict"
            },
            "conditions": [
              {
                "id": "cond-1",
                "leftValue": "={{ $json.type }}",
                "rightValue": "email",
                "operator": {
                  "type": "string",
                  "operation": "equals"
                }
              }
            ],
            "combinator": "and"
          }
        },
        {
          "outputKey": "sms",
          "conditions": {
            "options": {
              "caseSensitive": true,
              "typeValidation": "strict"
            },
            "conditions": [
              {
                "id": "cond-2",
                "leftValue": "={{ $json.type }}",
                "rightValue": "sms",
                "operator": {
                  "type": "string",
                  "operation": "equals"
                }
              }
            ],
            "combinator": "and"
          }
        }
      ]
    },
    "options": {
      "fallbackOutput": "extra",
      "ignoreCase": true
    }
  }
}
```

**Decision criteria**: Use **IF** for binary yes/no decisions; use **Switch** when routing to **3+ destinations** or when you need named output branches. Switch's Expression Mode excels when the output index can be calculated: `={{ {"high": 0, "medium": 1, "low": 2}[$json.priority] }}`.

### Merge Node: Combining data streams

Merge combines data from multiple inputs with four modes: **Append**, **Combine** (Matching Fields, Position, All Combinations), **SQL Query**, and **Choose Branch**.

```json
{
  "id": "merge-node-uuid",
  "name": "Merge Data",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3.2,
  "position": [700, 300],
  "parameters": {
    "mode": "combine",
    "combineBy": "combineByFields",
    "fieldsToMatchString": "id",
    "joinMode": "enrichInput1",
    "options": {
      "fuzzyCompare": true,
      "multipleMatches": "first"
    }
  }
}
```

**Connections receiving from multiple nodes:**

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

**Mode selection guide**:
| Mode | Use Case |
|------|----------|
| **Append** | Concatenate items from all inputs sequentially |
| **Combine by Fields** | Join datasets by matching field values (inner/outer/left/right joins) |
| **Combine by Position** | Pair items by index (item 0 with item 0) |
| **SQL Query** | Complex queries across inputs (v1.49.0+) |
| **Choose Branch** | Select which input to output (for conditional paths) |

**Common mistake**: Uneven item counts with Position mode silently drops items. **Fix**: Enable `"includeUnpaired": true` to keep all items.

### Split In Batches (Loop Over Items): Controlled iteration

Processes items in configurable batch sizes, essential for rate-limited APIs and nodes that only process the first item.

```json
{
  "id": "loop-node-uuid",
  "name": "Loop Over Items",
  "type": "n8n-nodes-base.splitInBatches",
  "typeVersion": 3,
  "position": [500, 300],
  "parameters": {
    "batchSize": 10,
    "options": {
      "reset": false
    }
  }
}
```

**Loop connection pattern** (output 0 = loop, output 1 = done):

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

**Context expressions for loop state:**
```javascript
{{ $("Loop Over Items").context["noItemsLeft"] }}     // boolean: true when done
{{ $("Loop Over Items").context["currentRunIndex"] }} // current iteration (0-based)
```

**Critical anti-pattern**: Using `reset: true` without an IF node exit condition causes infinite loops. Always include termination logic when reset is enabled.

### Split Out: Array explosion

Converts a single item containing an array into multiple individual items—the inverse of aggregation.

```json
{
  "id": "splitout-node-uuid",
  "name": "Split Out",
  "type": "n8n-nodes-base.splitOut",
  "typeVersion": 1,
  "position": [500, 300],
  "parameters": {
    "fieldToSplitOut": "items",
    "include": "allOtherFields",
    "options": {
      "destinationFieldName": "item",
      "disableDotNotation": false
    }
  }
}
```

**Use case**: An API returns `{"orders": [{...}, {...}, {...}]}` and you need to process each order individually. Split Out on `orders` produces three items.

---

## Data Manipulation Nodes

### Edit Fields (Set Node): Field manipulation

Creates, modifies, or removes fields without code. Supports **Manual Mapping** (GUI) and **JSON Output** modes.

```json
{
  "id": "set-node-uuid",
  "name": "Edit Fields",
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [500, 300],
  "parameters": {
    "mode": "manual",
    "duplicateItem": false,
    "assignments": {
      "assignments": [
        {
          "id": "assign-1",
          "name": "fullName",
          "value": "={{ $json.firstName }} {{ $json.lastName }}",
          "type": "string"
        },
        {
          "id": "assign-2",
          "name": "processedAt",
          "value": "={{ $now.toISO() }}",
          "type": "string"
        }
      ]
    },
    "includeOtherFields": true,
    "options": {
      "dotNotation": true
    }
  }
}
```

**Dot notation behavior**: With `dotNotation: true`, setting field `user.email` creates nested structure `{"user": {"email": "..."}}`. Disable to create literal key `"user.email"`.

### Code Node: Custom JavaScript/Python

Executes custom code with two modes: **Run Once for All Items** (process entire dataset) and **Run Once for Each Item** (per-item processing).

```json
{
  "id": "code-node-uuid",
  "name": "Code",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [500, 300],
  "parameters": {
    "mode": "runOnceForAllItems",
    "jsCode": "const items = $input.all();\n\nreturn items.map(item => ({\n  json: {\n    ...item.json,\n    total: item.json.price * item.json.quantity,\n    processedAt: $now.toISO()\n  }\n}));"
  }
}
```

**Essential built-in variables:**

| Variable | Mode | Description |
|----------|------|-------------|
| `$input.all()` | All Items | Array of all input items |
| `$input.first()` | All Items | First input item |
| `$json` | Each Item | Current item's JSON data directly |
| `$('NodeName').item.json` | Each Item | Linked item from specific node |
| `$execution.id` | Both | Current execution ID |
| `$workflow.name` | Both | Workflow name |
| `$now` | Both | Current DateTime (Luxon) |

**Return structure requirement:**
```javascript
// Must return array of objects with json key
return [
  { json: { name: "Alice", score: 95 } },
  { json: { name: "Bob", score: 87 } }
];
```

**Common errors and fixes:**

| Error | Cause | Solution |
|-------|-------|----------|
| `Task Request Timeout (60 seconds)` | Processing too many items | Use Loop Over Items to batch |
| `Cannot read property 'json' of undefined` | Accessing non-existent node | Verify node name spelling exactly |
| `Cannot find module` | External npm module | Set `NODE_FUNCTION_ALLOW_EXTERNAL` env var |

**Code vs deprecated Function Node**: The Function node (deprecated v0.198.0) used `items` directly and `item` for single processing. Migrate to Code node's `$input.all()` and `$json` patterns.

### Filter Node: Conditional item removal

Passes items matching conditions, discards others. Does not branch—simply removes non-matching items from the flow.

```json
{
  "id": "filter-node-uuid",
  "name": "Filter",
  "type": "n8n-nodes-base.filter",
  "typeVersion": 2.3,
  "position": [500, 300],
  "parameters": {
    "conditions": {
      "options": {
        "typeValidation": "strict"
      },
      "conditions": [
        {
          "id": "filter-cond-1",
          "leftValue": "={{ $json.status }}",
          "rightValue": "active",
          "operator": {
            "type": "string",
            "operation": "equals"
          }
        },
        {
          "id": "filter-cond-2",
          "leftValue": "={{ $json.score }}",
          "rightValue": 50,
          "operator": {
            "type": "number",
            "operation": "gt"
          }
        }
      ],
      "combinator": "and"
    },
    "looseTypeValidation": false
  }
}
```

**Decision**: Use **Filter** to remove items from a single stream. Use **IF** when you need to process non-matching items differently (two branches).

---

## Control Flow Nodes

### Wait Node: Execution pause and resume

Pauses workflow execution with four resume modes: **After Time Interval**, **At Specified Time**, **On Webhook Call**, and **On Form Submitted**.

```json
{
  "id": "wait-node-uuid",
  "name": "Wait for Approval",
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "position": [500, 300],
  "parameters": {
    "resume": "webhook",
    "httpMethod": "POST",
    "responseCode": 200,
    "options": {
      "webhookSuffix": "/approval-callback"
    },
    "limitWaitTime": {
      "limitType": "afterTimeInterval",
      "amount": 24,
      "unit": "hours"
    }
  }
}
```

**Critical webhook resume pattern**: Send `$execution.resumeUrl` to the external service **before** the Wait node:

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

**Common mistakes:**
- Accessing `$execution.resumeUrl` after the Wait node (URL is only valid before pause)
- n8n Cloud 100-second timeout: Use polling patterns for long waits
- Missing `WEBHOOK_URL` environment variable: Resume URLs show `localhost:5678` behind reverse proxy

### Execute Sub-workflow: Modular workflow calls

Calls another workflow, enabling code reuse and memory optimization for large datasets.

```json
{
  "id": "exec-wf-uuid",
  "name": "Process Order",
  "type": "n8n-nodes-base.executeWorkflow",
  "typeVersion": 1.3,
  "position": [500, 300],
  "parameters": {
    "source": "database",
    "workflowId": {
      "__rl": true,
      "mode": "list",
      "value": "workflow-abc123"
    },
    "mode": "once",
    "options": {}
  }
}
```

**Sub-workflow trigger configuration:**
```json
{
  "id": "exec-trigger-uuid",
  "name": "When Executed by Another Workflow",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "typeVersion": 1.1,
  "position": [250, 300],
  "parameters": {
    "inputSource": "workflowInputs",
    "workflowInputs": {
      "values": [
        {"name": "orderId", "type": "number"},
        {"name": "customerEmail", "type": "string"}
      ]
    }
  }
}
```

**Synchronous vs asynchronous**: Set `waitForSubWorkflow: true` to receive output data; `false` for fire-and-forget patterns. Sub-workflow executions do not count toward plan limits.

---

## Error Handling Nodes

### Error Trigger: Centralized error handling

Starts a dedicated error workflow when production workflows fail. Does not require activation.

```json
{
  "id": "error-trigger-uuid",
  "name": "Error Trigger",
  "type": "n8n-nodes-base.errorTrigger",
  "typeVersion": 1,
  "position": [250, 300],
  "parameters": {}
}
```

**Error data structure received:**
```json
{
  "execution": {
    "id": "231",
    "url": "https://n8n.example.com/execution/231",
    "error": {
      "message": "Request failed with status 500",
      "stack": "Error: Request failed..."
    },
    "lastNodeExecuted": "HTTP Request",
    "mode": "trigger"
  },
  "workflow": {
    "id": "1",
    "name": "Daily Sync"
  }
}
```

**Setup**: Create error workflow with Error Trigger → notification nodes. In production workflows, set **Options > Settings > Error workflow** to your error handler.

**Important**: Error workflows only trigger for production/scheduled runs, not manual test executions.

### Stop and Error: Deliberate failures

Throws custom errors to halt execution with meaningful messages for error workflows.

```json
{
  "id": "stop-error-uuid",
  "name": "Stop and Error",
  "type": "n8n-nodes-base.stopAndError",
  "typeVersion": 1,
  "position": [500, 300],
  "parameters": {
    "errorType": "errorMessage",
    "errorMessage": "Invalid order: missing required field 'customerId'"
  }
}
```

**Use case**: Validation failures, business rule violations, or graceful termination with specific error information.

### Node-level error settings

Every node supports these error handling options:

| Setting | Behavior |
|---------|----------|
| **Stop Workflow** (default) | Halts execution, triggers error workflow |
| **Continue on Fail** | Continues with error info passed forward |
| **Continue Using Error Output** | Routes to separate error branch |
| **Retry on Fail** | Up to 5 retries with max 5000ms interval |

**Complete error handling pattern:**
```json
"connections": {
  "HTTP Request": {
    "main": [
      [{"node": "Process Response", "type": "main", "index": 0}],
      [{"node": "Handle Error", "type": "main", "index": 0}]
    ]
  }
}
```

---

## HTTP & API Nodes

### HTTP Request: Universal API integration

The HTTP Request node supports all HTTP methods, multiple authentication types, pagination, and comprehensive request/response configuration.

```json
{
  "id": "http-node-uuid",
  "name": "API Call",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.3,
  "position": [500, 300],
  "parameters": {
    "method": "POST",
    "url": "https://api.example.com/orders",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {"name": "Content-Type", "value": "application/json"}
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ { \"orderId\": $json.id, \"items\": $json.lineItems } }}",
    "options": {
      "response": {
        "response": {"responseFormat": "json"}
      },
      "timeout": 30000
    }
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "cred-123",
      "name": "API Key"
    }
  }
}
```

**Authentication types**: Basic Auth, Digest Auth, Header Auth, Bearer Token, OAuth1, OAuth2 (Authorization Code, Client Credentials, PKCE), Query Auth, Custom Auth.

**Pagination configuration:**
```json
"options": {
  "pagination": {
    "paginationMode": "updateAParameterInEachRequest",
    "parameters": {
      "parameters": [
        {"name": "page", "value": "={{ $pageCount }}"}
      ]
    },
    "maxRequests": 100,
    "paginationComplete": "={{ $response.body.hasMore === false }}"
  }
}
```

**Common errors and solutions:**

| Error | Solution |
|-------|----------|
| `400 Bad Request - check parameters` | Verify query param names; use Array Format option for arrays |
| `403 Forbidden` | Check credentials, permissions, rate limits |
| `429 Rate Limit` | Implement batching with Loop Over Items + Wait nodes |
| `JSON parameter need to be valid JSON` | Ensure expressions return objects, not strings |
| `Response body is not valid JSON` | Set Response Format to "Text" for non-JSON APIs |

### Webhook Node: Receive external requests

Creates HTTP endpoints for receiving data from external services. Can function as a trigger or mid-workflow node.

```json
{
  "id": "webhook-node-uuid",
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2.1,
  "webhookId": "unique-webhook-id",
  "position": [250, 300],
  "parameters": {
    "httpMethod": "POST",
    "path": "orders/new",
    "authentication": "headerAuth",
    "responseMode": "lastNode",
    "responseData": "firstEntryJson",
    "options": {
      "allowedOrigins": "*",
      "rawBody": true
    }
  }
}
```

**Response modes:**
- **Immediately**: Returns "Workflow got started" without waiting
- **When Last Node Finishes**: Returns output from final node
- **Using 'Respond to Webhook' Node**: Custom response at any workflow point

**URL structure**: Test URL (development) vs Production URL (active workflows). Only production URLs work when workflow is activated.

**Path parameters**: Support dynamic routes like `/orders/:orderId/items/:itemId`. Access via `{{ $json.params.orderId }}`.

---

## Trigger Nodes

### Schedule Trigger: Time-based automation

Runs workflows on schedules using intervals or cron expressions. Workflow must be activated.

```json
{
  "id": "schedule-uuid",
  "name": "Schedule Trigger",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.3,
  "position": [250, 300],
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "cronExpression",
          "expression": "0 9 * * 1-5"
        }
      ]
    }
  }
}
```

**Cron expression examples** (n8n uses 6-field format with seconds):
```
*/30 * * * * *    Every 30 seconds
0 */5 * * * *     Every 5 minutes
0 0 9 * * *       Daily at 9:00 AM
0 0 9 * * 1-5     Weekdays at 9:00 AM
0 0 0 1 * *       First of month at midnight
```

**Timezone configuration**: Set in workflow settings or via `GENERIC_TIMEZONE` environment variable. Default is `America/New_York` for self-hosted instances.

### Webhook Trigger: Event-driven starts

Identical to Webhook node but specifically positioned as the workflow's starting point.

```json
{
  "id": "webhook-trigger-uuid",
  "name": "Webhook Trigger",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2.1,
  "webhookId": "trigger-webhook-id",
  "position": [250, 300],
  "parameters": {
    "httpMethod": "POST",
    "path": "incoming-data",
    "responseMode": "onReceived",
    "options": {}
  }
}
```

**Max payload**: 16MB default, configurable via `N8N_PAYLOAD_SIZE_MAX`.

### Manual Trigger: Development and testing

Starts workflows via the Execute Workflow button. Does not output data—simply initiates the workflow.

```json
{
  "id": "manual-trigger-uuid",
  "name": "Manual Trigger",
  "type": "n8n-nodes-base.manualTrigger",
  "typeVersion": 1,
  "position": [250, 300],
  "parameters": {}
}
```

**Limitation**: Only one Manual Trigger allowed per workflow.

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

## Complete Working Examples

### API Integration with Error Handling and Retry

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

### Rate-Limited Batch Processing

```json
{
  "name": "Rate Limited Batch",
  "nodes": [
    {
      "id": "manual-1",
      "name": "Manual Trigger",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [250, 300],
      "parameters": {}
    },
    {
      "id": "code-1",
      "name": "Get Items",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [450, 300],
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "return Array.from({length: 100}, (_, i) => ({json: {id: i + 1}}));"
      }
    },
    {
      "id": "loop-1",
      "name": "Loop",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [650, 300],
      "parameters": {"batchSize": 10, "options": {}}
    },
    {
      "id": "http-1",
      "name": "API Call",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [850, 250],
      "parameters": {
        "url": "=https://api.example.com/process/{{ $json.id }}",
        "method": "POST"
      }
    },
    {
      "id": "wait-1",
      "name": "Wait",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [1050, 250],
      "parameters": {"resume": "timeInterval", "amount": 1, "unit": "seconds"}
    },
    {
      "id": "noop-1",
      "name": "Done",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [850, 400],
      "parameters": {}
    }
  ],
  "connections": {
    "Manual Trigger": {"main": [[{"node": "Get Items", "type": "main", "index": 0}]]},
    "Get Items": {"main": [[{"node": "Loop", "type": "main", "index": 0}]]},
    "Loop": {
      "main": [
        [{"node": "API Call", "type": "main", "index": 0}],
        [{"node": "Done", "type": "main", "index": 0}]
      ]
    },
    "API Call": {"main": [[{"node": "Wait", "type": "main", "index": 0}]]},
    "Wait": {"main": [[{"node": "Loop", "type": "main", "index": 0}]]}
  },
  "settings": {"executionOrder": "v1"}
}
```

---

## Production Best Practices

**Error resilience**: Always configure error workflows for production workflows. Use "Continue on Fail Using Error Output" for non-critical nodes to prevent full workflow failures.

**Rate limiting**: Combine Loop Over Items with Wait nodes rather than relying solely on HTTP Request batching—provides more control and visibility.

**Memory management**: For workflows processing thousands of items, use Execute Sub-workflow to clear memory between batches. Sub-workflow executions don't count toward plan limits.

**Credential security**: Never hardcode API keys. Use n8n's credentials system with environment variables for sensitive values.

**Testing strategy**: Use Manual Trigger during development, then switch to production triggers. Remember that Error Trigger workflows only fire for production executions, not manual tests.

**Version pinning**: Document the `typeVersion` of critical nodes. When updating n8n, verify that existing workflows maintain expected behavior with newer node versions.

---

## Document Metadata

**Created**: 2026-01-18T16:45:00Z  
**Updated**: 2026-01-18T17:15:00Z  
**Version**: 2.0 (Added CRITICAL typeVersion reference section)

---

## Bibliography

1. n8n Documentation - Export and import workflows. https://docs.n8n.io/workflows/export-import/
2. n8n Documentation - Understanding the data structure. https://docs.n8n.io/courses/level-two/chapter-1/
3. n8n Documentation - If Node. https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/
4. n8n Documentation - Switch Node. https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/
5. n8n Documentation - Loop Over Items (Split in Batches). https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/
6. n8n Documentation - Edit Fields (Set). https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/
7. n8n Documentation - Using the Code node. https://docs.n8n.io/code/code-node/
8. n8n Documentation - Wait Node. https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/
9. n8n Documentation - Error handling. https://docs.n8n.io/flow-logic/error-handling/
10. n8n Documentation - HTTP Request credentials. https://docs.n8n.io/integrations/builtin/credentials/httprequest/
11. n8n Documentation - HTTP Request node common issues. https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/common-issues/
12. n8n Documentation - Building a mini-workflow. https://docs.n8n.io/courses/level-one/chapter-2/
13. n8n Documentation - Versioning. https://docs.n8n.io/integrations/creating-nodes/build/reference/node-versioning/
14. Autom8this - The n8n IF Node and Switch Node. https://autom8this.com/the-n8n-if-node-and-switch-node/
15. Latenode - N8N Import Workflow JSON: Complete Guide. https://latenode.com/blog/low-code-no-code-platforms/n8n-setup-workflows-self-hosting-templates/n8n-import-workflow-json-complete-guide-file-format-examples-2025
16. GitHub n8n-docs - Subworkflow usage. https://github.com/n8n-io/n8n-docs/blob/main/_snippets/flow-logic/subworkflow-usage.md
17. GitHub n8n Issues - HTTP Request node error output behavior. https://github.com/n8n-io/n8n/issues/18113
