---
name: n8n-platform-reference
description: Comprehensive reference for n8n platform concepts, expressions, and node behavior
shareable: true
---

# n8n Workflow Automation Platform - Claude Code Skill

**Last Updated:** 2026-01-31
**Platform Version:** n8n Cloud (January 2026)
**Purpose:** Comprehensive reference for n8n platform concepts, expressions, and node behavior

---

## Overview

The n8n workflow automation platform provides a node-based visual environment for building integrations and automations. This skill documents platform fundamentals, expression syntax, and node behavior patterns—critical for understanding how n8n works before generating workflows.

**Primary Use Cases:**
- Understanding n8n data flow and item structure
- Writing expressions with correct syntax
- Selecting appropriate nodes for workflow patterns
- Avoiding deprecated syntax and AI hallucinations
- Troubleshooting workflow execution issues

**Related Skills:**
- `n8n-node-implementation.md` - JSON structures for workflow generation (typeVersions, parameters)
- `n8n-workflow-development.md` - Workflow building patterns and anti-patterns
- `n8n-troubleshooting.md` - MCP connection issues, debugging

---

## Data Structure Fundamentals

All data flows through n8n as **arrays of items**, where each item contains a `json` object (and optionally a `binary` object):

```javascript
[
  { "json": { "name": "John", "email": "john@example.com" }, "binary": {} },
  { "json": { "name": "Jane", "email": "jane@example.com" }, "binary": {} }
]
```

Every node receives items, processes them, and outputs items in this format. The Code node must return data in this structure (though v0.166.0+ auto-wraps if `json` key is omitted). Binary data for files uses the `binary` key with properties like `mimeType`, `fileName`, and `fileSize`.

---

## Expression Syntax Reference

Expressions use **double curly braces** `{{ }}` and provide access to workflow data through built-in variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `$json` | Current item's JSON data | `{{ $json.email }}` |
| `$input.all()` | All input items as array | `{{ $input.all() }}` |
| `$input.first()` | First input item | `{{ $input.first().json.name }}` |
| `$input.last()` | Last input item | `{{ $input.last().json.id }}` |
| `$input.item` | Current item being processed | `{{ $input.item.json.status }}` |
| `$('NodeName')` | Access another node's output | `{{ $('Webhook').first().json.body }}` |
| `$binary` | Current item's binary data | `{{ $binary.data.fileName }}` |
| `$execution.id` | Current execution ID | `{{ $execution.id }}` |
| `$workflow.name` | Workflow name | `{{ $workflow.name }}` |
| `$now` | Current Luxon DateTime | `{{ $now.toFormat('yyyy-MM-dd') }}` |
| `$today` | Today at midnight | `{{ $today }}` |
| `$vars` | Custom workflow variables | `{{ $vars.apiEndpoint }}` |
| `$env` | Environment variables (self-hosted) | `{{ $env.API_KEY }}` |
| `$itemIndex` | Current item's index | `{{ $itemIndex }}` |
| `$runIndex` | Current run index in loops | `{{ $runIndex }}` |

### Accessing Other Nodes

Uses the pattern `$('NodeName').method()`:
- `$('HTTP Request').first().json.data` — First item's data field
- `$('Webhook').all()[0].json` — Array access to first item
- `$('Database').item.json` — Corresponding item by index
- `$('API Call').itemMatching(index)` — Item matching specific index

### JMESPath Queries

For complex JSON filtering:
```javascript
{{ $jmespath($json, 'users[?status==`active`].name') }}
{{ $jmespath($json, 'items[?price > `100`]') }}
```

### Luxon Date Operations

Built-in date library:
```javascript
{{ $now.plus({ days: 7 }).toFormat('yyyy-MM-dd') }}
{{ DateTime.fromISO($json.date).diff($now, 'days').days }}
```

---

## Core Flow Control Nodes

### IF Node
Creates **true/false branches** based on conditions.

**Parameters:**
- **Conditions**: Boolean comparisons using data type operators
- **Combine Conditions**: `AND` (all must match) or `OR` (any matches)
- **Ignore Case**: Case-insensitive string comparisons
- **Less Strict Type Validation**: Attempts type conversion

**Outputs:** True branch (output 0), False branch (output 1)

### Switch Node
Routes items to **multiple branches** based on rules.

**Modes:**
- **Rules Mode**: Define conditions for each output branch
- **Expression Mode**: Use expression to calculate output name dynamically

**Parameters:**
- **Rules**: Condition definitions with named outputs
- **Fallback Output**: Default for unmatched items
- **Send data to all matching outputs**: Route to multiple branches simultaneously

### Merge Node
Combines data from **multiple workflow branches**.

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Append** | Concatenates all items into single list | Combining results from parallel branches |
| **Combine by Fields** | Joins items matching field values | SQL-like JOIN operations |
| **Combine by Position** | Pairs items by array index | Matching ordered datasets |
| **Multiplex** | Cartesian product of all inputs | Cross-joining datasets |
| **Choose Branch** | Outputs from selected input only | Conditional branch selection |

**Clash Handling** (for field name conflicts):
- `Always Add Input Number`: Prefix keys with input number
- `Prefer Input 1/2`: Keep specified input's value
- `Merge`: Combine objects, prefer specified for primitives

### Loop Over Items (Split in Batches)
Processes items in **batches** with loop control.

**Parameters:**
- **Batch Size**: Items per iteration (e.g., 1 for API rate limiting)
- **Reset**: Treat incoming data as new set

**Outputs:**
- **Loop** (output 0): Continue processing (connect back to processing nodes)
- **Done** (output 1): All items processed (continue workflow)

**Context Variables:**
```javascript
$node["Loop Over Items"].context["noItemsLeft"]  // Boolean
$node["Loop Over Items"].context["currentRunIndex"]  // Number
```

⚠️ **Critical**: Always include termination condition to prevent infinite loops.

### Wait Node
Pauses execution until **resume condition** is met.

**Resume Operations:**
- **After Time Interval**: Wait seconds/minutes/hours/days
- **At Specified Time**: Wait until specific datetime
- **On Webhook Call**: Wait for HTTP request to `$execution.resumeUrl`
- **On Form Submitted**: Wait for form completion

⚠️ **Cloud Limit**: Webhook timeout is **100 seconds** (Cloudflare limit)

---

## Data Transformation Nodes

### Edit Fields (Set) Node
Sets or modifies workflow data fields.

**Modes:**
- **Manual Mapping**: GUI-based field configuration
- **JSON Output**: Write JSON directly

**Key Options:**
- **Keep Only Set Fields**: Discard all other input fields
- **Include Binary Data**: Preserve binary data in output
- **Support Dot Notation**: Enable `parent.child` syntax (default: on)

### Filter Node
Removes items that don't match conditions.

**Operators by Type:**

| Type | Operators |
|------|-----------|
| String | exists, is empty, equals, contains, starts with, ends with, matches regex |
| Number | exists, equals, greater than, less than, ≥, ≤ |
| Boolean | is true, is false, equals |
| Array | exists, is empty, contains, length comparisons |
| Date & Time | is after, is before, equals |

### Aggregate Node
Combines separate items into grouped items.

**Aggregate Types:**
- **Individual Fields**: Aggregate specific fields with optional renaming
- **All Item Data**: Aggregate entire items into single field

### Summarize Node
Performs **pivot table-like** aggregations.

**Aggregation Methods:** Append, Average, Concatenate, Count, Count Unique, Max, Min, Sum

**Split By**: Group results by field values (like SQL GROUP BY)

---

## Code Node (JavaScript/Python)

The Code node replaced the deprecated **Function** and **Function Item** nodes.

### Execution Modes

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

### Available Built-in Variables

```javascript
// Input access
$input.all()           // All items array
$input.first()         // First item
$input.last()          // Last item
$input.item            // Current item (in "each item" mode)
$json                  // Current item's json (in "each item" mode)

// Other node access
$('NodeName').all()    // All items from node
$('NodeName').first()  // First item from node
$('NodeName').item     // Matching item from node

// Workflow data
$workflow.id
$workflow.name
$execution.id
$execution.mode
$now                   // Luxon DateTime
$today

// Persistent storage
const staticData = $getWorkflowStaticData('global');
staticData.lastRun = Date.now();
```

### Environment Limitations
- ❌ **No file system access** — Use Read/Write Files nodes
- ❌ **No HTTP requests** — Use HTTP Request node
- ❌ **No DOM access** — Server-side execution only
- ✅ **Luxon** available for dates
- ✅ **console.log()** outputs to browser dev console
- ✅ **async/await** supported

---

## HTTP Request Node

The most versatile node for API integrations.

### Parameters

| Parameter | Options |
|-----------|---------|
| **Method** | GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS |
| **URL** | Endpoint URL (expressions supported) |
| **Authentication** | None, Predefined Credential Type, Generic Credential Type |
| **Send Headers** | Key-value header pairs |
| **Send Query Params** | URL query parameters |
| **Send Body** | JSON, Form-Data, Form Urlencoded, Raw, n8n Binary File |

### Authentication Methods
- **Predefined**: OAuth2, API Key, Basic Auth for specific services
- **Generic**: Header Auth, Query Auth, Digest Auth, Bearer Token, Custom Auth

### Pagination Modes
- **Response Contains Next URL**: API returns next page URL
- **Response Contains Page Number**: Increment page/offset parameter
- **Update a Parameter**: Custom pagination with expressions

### Advanced Options
- **Batching**: Group requests with interval delays
- **Timeout**: Request timeout in milliseconds
- **Retry On Fail**: Automatic retry with configurable attempts
- **Response Format**: JSON (auto-parse), Text, or File

---

## Webhook Node

Creates HTTP endpoints to **trigger workflows**.

| Setting | Description |
|---------|-------------|
| **HTTP Method** | GET, POST, PUT, PATCH, DELETE, HEAD, or All |
| **Path** | URL path appended to webhook base URL |
| **Authentication** | None, Basic Auth, Header Auth, JWT Auth |
| **Respond** | Immediately, When Last Node Finishes, Using Respond to Webhook node |

### URL Types
- **Test URL**: For development, active during "Listen for Test Event"
- **Production URL**: Active only when workflow is published/active

⚠️ **Cloud Limit**: Webhook timeout is **100 seconds**—returns 524 error if exceeded.

---

## Error Handling Patterns

### Error Trigger Node
Captures workflow failures and triggers error handling workflow.

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

⚠️ **Critical**: Error Trigger only fires for **automatic** workflow executions, not manual runs.

### Per-Node Error Settings

| Setting | Options |
|---------|---------|
| **On Error** | Stop Workflow, Continue, Continue (using error output) |
| **Retry On Fail** | Enable with Max Tries and Wait Between Tries |

---

## Anti-Patterns and Common Mistakes

### Nodes that DON'T EXIST (AI hallucinations)
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

// ❌ WRONG: String "undefined" check
{{ $if($json.value === "undefined", "empty", $json.value) }}

// ✅ CORRECT: Nullish coalescing
{{ $json.value ?? "empty" }}
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

// ❌ WRONG: Missing json wrapper (pre-v0.166.0)
return [{ name: "John" }];

// ✅ CORRECT: Proper structure
return [{ json: { name: "John" } }];
```

### Sub-node Expression Behavior
In AI sub-nodes, expressions **always resolve to first item only**:
```javascript
// In regular nodes: {{ $json.name }} runs for each item
// In sub-nodes: {{ $json.name }} ALWAYS returns first item's value
```

### Static Data in Manual Executions
`$getWorkflowStaticData()` does **NOT persist** during manual test runs—only works in active/triggered workflows.

---

## Cloud vs Self-Hosted Differences

| Feature | Cloud | Self-Hosted |
|---------|-------|-------------|
| Environment variables | Limited (use Variables) | Full control |
| OAuth setup | Simplified with pre-configured URLs | Manual configuration |
| File system access | Restricted | Full access |
| Custom nodes | Supported | Supported |
| Execution timeout | Plan-based limits | Configurable |
| Webhook payload max | 16MB | Configurable |
| Credential storage | n8n-managed encrypted | Your database |

---

## Performance Optimization

### Cloud Memory Limits

| Plan | Approximate Memory | Execution Timeout |
|------|-------------------|-------------------|
| Starter | ~250MB | ~3 minutes |
| Pro | ~512MB | Higher |
| Enterprise | Configurable | Configurable |

### Optimization Strategies

1. **Split large datasets**: Process 50-200 items per batch instead of thousands
2. **Use sub-workflows**: Memory freed after each sub-workflow completes
3. **Avoid Code node for simple transforms**: Built-in nodes are more memory efficient
4. **Manual executions use more memory**: Don't test large datasets manually

---

## Node Selection Quick Reference

### Core Nodes (Always Available)
Manual Trigger, Schedule Trigger, Webhook, HTTP Request, Code, Edit Fields (Set), Filter, IF, Switch, Merge, Loop Over Items, Wait, Limit, Sort, Aggregate, Summarize, Compare Datasets, Execute Workflow, Error Trigger, Stop And Error, Respond to Webhook, No Operation

### Commonly Used Integration Nodes
PostgreSQL, MySQL, Supabase, Google Drive, Google Sheets, Gmail, Slack, HubSpot, Salesforce, AWS S3, Airtable, Notion, Discord, Telegram, OpenAI, Anthropic

---

## Key Gotchas

1. **Webhook timeout**: 100 seconds on Cloud (Cloudflare limit)
2. **Error Trigger**: Only fires for production executions, not manual tests
3. **Static data**: Doesn't persist in manual test runs
4. **Sub-node expressions**: Always resolve to first item only
5. **Simple Memory Node**: Doesn't work in Queue Mode (use Redis/Postgres)
6. **Schedule Trigger**: Workflow must be **saved AND published** to execute

---

**Source Document:** `/Users/mikeboscia/My Drive/GTM Machine content/Research & Strategy/n8n Workflow Automation Platform- Complete Technical Reference.md`
