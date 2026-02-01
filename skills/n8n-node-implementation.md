---
name: n8n-node-implementation
description: JSON structures, typeVersions, and parameter patterns for generating n8n workflow JSON
shareable: true
---

# n8n Node Implementation Reference - Claude Code Skill

**Last Updated:** 2026-01-31
**Platform Version:** n8n Cloud (January 2026)
**Purpose:** JSON structures, typeVersions, and parameter patterns for generating n8n workflow JSON

---

## Overview

This skill provides the exact JSON structures and typeVersion values required for generating valid n8n workflow JSON. Using incorrect typeVersions or deprecated parameter structures causes silent failures, missing parameters, and workflows that the UI cannot render.

**Primary Use Cases:**
- Generating workflow JSON via MCP tools or API
- Modifying existing workflow configurations
- Ensuring compatibility with current n8n versions
- Avoiding API-accepted but UI-broken workflows

**Related Skills:**
- `n8n-platform-reference.md` - Platform concepts, expressions, node behavior
- `n8n-workflow-development.md` - Workflow building patterns and anti-patterns
- `n8n-troubleshooting.md` - MCP connection issues, debugging

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

---

## Quick Copy-Paste Node Templates

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

The **connections object** uses an array-of-arrays pattern where:
- Outer array = outputs (index 0 = first output, index 1 = second output for IF nodes)
- Inner arrays = multiple connections from same output

Data flows as arrays of objects: `[{"json": {...}, "binary": {...}}]`.

---

## Complete Node Examples

### IF Node (v2.3)

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

**Available operators by type**:
- **String**: equals, notEquals, contains, notContains, startsWith, notStartsWith, endsWith, notEndsWith, regex, notRegex, isEmpty, isNotEmpty
- **Number**: equals, notEquals, gt, gte, lt, lte
- **Boolean**: true, false
- **Array**: contains, notContains, lengthEquals, lengthNotEquals, lengthGt, lengthGte, lengthLt, lengthLte, isEmpty, isNotEmpty

### Switch Node (v3.4)

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

### Merge Node (v3.2)

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

**Connections from multiple nodes:**
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

### Loop Over Items (v3)

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

### Edit Fields / Set Node (v3.4)

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

### Code Node (v2)

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

### HTTP Request (v4.3)

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

### Webhook (v2.1)

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

### Schedule Trigger (v1.3)

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

### Wait Node (v1.1)

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

### Execute Sub-workflow (v1.3)

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

### Error Trigger (v1)

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

### Stop and Error (v1)

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

---

## Complete Working Workflow Example

### API Integration with Error Handling

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

## Key Gotchas

1. **Common mistake with IF + Merge**: Using v0 execution order causes both branches to execute. **Fix**: Use `"executionOrder": "v1"` in settings.
2. **Uneven Merge item counts**: Position mode silently drops items. **Fix**: Enable `"includeUnpaired": true`.
3. **Loop infinite loop**: Using `reset: true` without exit condition. **Fix**: Always include IF node termination logic.
4. **Wait node resumeUrl**: Accessing after Wait node fails. **Fix**: Send `$execution.resumeUrl` BEFORE the Wait node.

---

**Source Document:** `/Users/mikeboscia/My Drive/GTM Machine content/Research & Strategy/Complete n8n Node Implementation Reference.md`
