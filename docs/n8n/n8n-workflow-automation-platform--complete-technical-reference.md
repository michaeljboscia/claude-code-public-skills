# n8n Workflow Automation Platform: Complete Technical Reference

The **n8n workflow automation platform** provides a node-based visual environment for building integrations and automations. This reference documents current n8n Cloud functionality with exact syntax, parameters, and patterns verified against official documentation—critical for avoiding hallucinated nodes or deprecated syntax that commonly plague AI-generated workflows.

## Data structure fundamentals

All data flows through n8n as **arrays of items**, where each item contains a `json` object (and optionally a `binary` object). Understanding this structure is essential:

```javascript
[
  { "json": { "name": "John", "email": "john@example.com" }, "binary": {} },
  { "json": { "name": "Jane", "email": "jane@example.com" }, "binary": {} }
]
```

Every node receives items, processes them, and outputs items in this format. The Code node must return data in this structure (though v0.166.0+ auto-wraps if `json` key is omitted). Binary data for files uses the `binary` key with properties like `mimeType`, `fileName`, and `fileSize`.

---

## Expression syntax reference

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

**Accessing other nodes** uses the pattern `$('NodeName').method()`:
- `$('HTTP Request').first().json.data` — First item's data field
- `$('Webhook').all()[0].json` — Array access to first item
- `$('Database').item.json` — Corresponding item by index
- `$('API Call').itemMatching(index)` — Item matching specific index

**JMESPath queries** for complex JSON filtering:
```javascript
{{ $jmespath($json, 'users[?status==`active`].name') }}
{{ $jmespath($json, 'items[?price > `100`]') }}
```

**Luxon date operations** (built-in):
```javascript
{{ $now.plus({ days: 7 }).toFormat('yyyy-MM-dd') }}
{{ DateTime.fromISO($json.date).diff($now, 'days').days }}
```

---

## Core flow control nodes

### IF Node
Creates **true/false branches** based on conditions.

**Parameters:**
- **Conditions**: Boolean comparisons using data type operators
- **Combine Conditions**: `AND` (all must match) or `OR` (any matches)
- **Ignore Case**: Case-insensitive string comparisons
- **Less Strict Type Validation**: Attempts type conversion

**Outputs:** True branch, False branch

```javascript
// Example condition: Check if status equals "active"
// Field: {{ $json.status }}
// Operation: equals
// Value: active
```

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

**Critical Modes:**

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
- **Loop**: Continue processing (connect back to processing nodes)
- **Done**: All items processed (continue workflow)

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

**Webhook Resume Options:**
- Authentication (None, Basic, Header, JWT)
- Response configuration
- Timeout limits
- `$execution.resumeUrl` provides unique webhook URL per execution

---

## Data transformation nodes

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

**Options:**
- **Ignore Case**: Case-insensitive comparisons
- **Less Strict Type Validation**: Attempt type coercion

### Sort Node
Orders items by field values.

**Types:**
- **Simple**: Sort by field(s), ascending/descending
- **Random**: Randomize order
- **Code**: Custom JavaScript sorting logic

### Limit Node
Restricts output to maximum items.

**Parameters:**
- **Max Items**: Maximum count
- **Keep**: `First Items` or `Last Items`

### Aggregate Node
Combines separate items into grouped items.

**Aggregate Types:**
- **Individual Fields**: Aggregate specific fields with optional renaming
- **All Item Data**: Aggregate entire items into single field

**Options:**
- **Merge Lists**: Flatten nested arrays
- **Include Binaries**: Preserve binary data

### Summarize Node
Performs **pivot table-like** aggregations.

**Aggregation Methods:**
- Append, Average, Concatenate, Count, Count Unique, Max, Min, Sum

**Split By**: Group results by field values (like SQL GROUP BY)

```javascript
// Example: Sum "amount" field grouped by "category"
// Field to Summarize: amount (Sum)
// Split By: category
```

### Compare Datasets Node
Compares two input datasets for differences.

**Outputs:**
- Items only in Input 1
- Items only in Input 2
- Items in both (matching)
- Items that are different

**Configuration:** Define key field(s) for matching records between inputs.

---

## Code Node (JavaScript/Python)

The Code node replaced the deprecated **Function** and **Function Item** nodes.

### Execution Modes

**Run Once for All Items** (default):
```javascript
// Access all items, return array
const items = $input.all();
const total = items.reduce((sum, item) => sum + item.json.amount, 0);
return [{ json: { total } }];
```

**Run Once for Each Item**:
```javascript
// Access current item via $json, return single item
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

### Binary Data Handling
```javascript
// Get binary buffer (recommended method)
const buffer = await this.helpers.getBinaryDataBuffer(itemIndex, 'data');

// Access binary metadata
const fileName = items[0].binary.data.fileName;
const mimeType = items[0].binary.data.mimeType;
```

### JavaScript Environment Limitations
- ❌ **No file system access** — Use Read/Write Files nodes
- ❌ **No HTTP requests** — Use HTTP Request node
- ❌ **No DOM access** — Server-side execution only
- ✅ **Luxon** available for dates
- ✅ **console.log()** outputs to browser dev console
- ✅ **async/await** supported

### Python Support (v1.111.0+)
```python
# Use _items for all-items mode
# Use _item for per-item mode
# Standard library available
# Does NOT support n8n built-in methods like $workflow
```

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

### Pagination Example
```javascript
// Using "Update a Parameter" mode for offset pagination
// Parameter Name: offset
// Parameter Value: {{ $pageCount * 100 }}
// Complete When: {{ $response.body.data.length === 0 }}
```

---

## Webhook Node

Creates HTTP endpoints to **trigger workflows**.

### Key Configuration

| Setting | Description |
|---------|-------------|
| **HTTP Method** | GET, POST, PUT, PATCH, DELETE, HEAD, or All |
| **Path** | URL path appended to webhook base URL |
| **Authentication** | None, Basic Auth, Header Auth, JWT Auth |
| **Respond** | Immediately, When Last Node Finishes, Using Respond to Webhook node |

### URL Types
- **Test URL**: For development, active during "Listen for Test Event"
- **Production URL**: Active only when workflow is published/active

### Respond to Webhook Node
Provides custom response control when Webhook's "Respond" is set to "Using Respond to Webhook node".

**Response Options:**
- All Incoming Items, First Incoming Item
- Custom JSON, Text, Binary File
- JWT Token, Redirect, No Data

⚠️ **Cloud Limit**: Webhook timeout is **100 seconds** (Cloudflare limit)—returns 524 error if exceeded.

---

## Database nodes

### PostgreSQL Node

**Operations:** Execute SQL Query, Insert, Update, Delete, Select

**Example Parameterized Query:**
```sql
SELECT * FROM users WHERE email = $1 AND status = $2
-- Parameters: ["user@example.com", "active"]
```

### MySQL Node

**Operations:** Execute SQL, Insert, Update, Delete

**Configuration:**
- Host, Database, Username, Password, Port (default: 3306)
- SSL support, SSH Tunnel option
- Query Parameters in Options for parameterized queries

⚠️ **Gotcha**: DECIMAL values return as strings by default—enable "Output Decimals as Numbers" option.

### Supabase Node

**Operations (Row resource):** Create, Delete, Get, Get All, Update

- Requires Supabase credentials (API URL + API Key)
- Supports custom schemas via "Use Custom Schema" option
- Row Level Security applies based on API key permissions

---

## Integration nodes reference

### Google Drive Node
**Operations:**
- File: Copy, Create from text, Delete, Download, Move, Share, Update, Upload
- Folder: Create, Delete, Share
- Search: Find files/folders

### AWS S3 Node
**Operations:**
- Bucket: Create, Delete, Get All, Search
- File: Copy, Delete, Download, Get All, Upload
- Folder: Create, Delete, Get All

### HubSpot Node
**Resources:** Contact, Company, Deal, Ticket, Engagement, Form, Contact List
**Operations vary by resource**: Create, Update, Delete, Get, Get All, Search

### Salesforce Node  
**Resources:** Account, Contact, Lead, Opportunity, Case, Task, Custom Object
**Operations:** Create, Upsert, Update, Delete, Get, Get All, Search (SOQL)

---

## Schedule Trigger Node

**Trigger Intervals:**
- Seconds, Minutes, Hours, Days, Weeks, Months
- **Custom (Cron)**: Standard cron expressions (supports 6-field format with seconds)

**Cron Examples:**
```
0 8 * * *        # 8:00 AM daily
30 8 * * 1-5     # 8:30 AM weekdays
0 0 1 * *        # Midnight on 1st of month
*/15 * * * *     # Every 15 minutes
```

⚠️ **Critical**: Workflow must be **saved AND published** for Schedule Trigger to execute. Variable values in schedule only evaluate when workflow is published.

---

## Execute Workflow Node (Sub-workflows)

Calls another workflow as a **sub-workflow**.

### Reference Methods
- **Database ID**: Workflow ID from URL
- **URL**: Full workflow URL
- **From Local File**: JSON file path
- **By Parameter**: Inline JSON

### Input Configuration (in sub-workflow)
The sub-workflow uses **Execute Workflow Trigger** node with input options:
- **Define using fields**: Specify expected input fields and types
- **Define using JSON example**: Provide example JSON structure
- **Accept all data**: No validation

### Options
- **Wait for Sub-Workflow Completion**: Block parent until sub-workflow finishes
- **Run once with all items** vs **Run once for each item**

⚠️ Sub-workflow executions **don't count** toward Cloud execution limits.

---

## Error handling patterns

### Error Trigger Node
Captures workflow failures and triggers error handling workflow.

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

⚠️ **Critical**: Error Trigger only fires for **automatic** workflow executions, not manual runs.

### Per-Node Error Settings

| Setting | Options |
|---------|---------|
| **On Error** | Stop Workflow, Continue, Continue (using error output) |
| **Retry On Fail** | Enable with Max Tries and Wait Between Tries |

### Stop And Error Node
Deliberately fails workflow with custom error message—useful for validation failures.

### Retry Pattern with Exponential Backoff
```javascript
// In HTTP Request node options:
// Retry On Fail: Enabled
// Max Tries: 3
// Wait Between Tries: 1000ms

// For custom backoff, use Loop Over Items + Wait node
```

---

## Credential management

### Creating Credentials
1. Open node requiring authentication
2. Select "Create New" in Credential dropdown
3. Enter authentication details
4. Test connection (if available)
5. Save

### Credential Types
- **Predefined**: 400+ service-specific credentials
- **Generic**: Basic Auth, Header Auth, OAuth1/2, API Key, Digest, Custom Auth

### OAuth2 Setup
1. Create app in service provider
2. Copy n8n's OAuth Redirect URL to service
3. Enter Client ID, Client Secret, Scopes in n8n
4. Click "Connect my account"

**Grant Types:** Authorization Code (most common), Client Credentials, PKCE

### Sharing (Paid Plans)
Shared users can **use** but cannot **view/edit** credential details.

---

## Performance optimization

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

### Batch Processing
```javascript
// Loop Over Items configuration for rate-limited APIs
// Batch Size: 1 (for strict rate limits)
// Batch Size: 10-50 (for batched API calls)
// Batch Size: 100-200 (for database operations)
```

---

## Anti-patterns and common mistakes

### ❌ Nodes that DON'T EXIST (AI hallucinations)
- ~~"Google Trends" node~~ → Use HTTP Request
- ~~"SerpApi" standalone node~~ → Only exists as AI Agent tool sub-node
- ~~"Cron Trigger"~~ → Use **Schedule Trigger**
- ~~"Start" node~~ → **Removed in v2.0**, use Manual Trigger

### ❌ Deprecated Syntax

| Deprecated | Current Replacement |
|------------|---------------------|
| Function node | **Code** node |
| Function Item node | **Code** node (Run Once for Each Item) |
| Item Lists node | **Split Out**, **Aggregate**, **Sort**, **Remove Duplicates** |
| `items` global variable | `$input.all()` |
| `$evaluateExpression()` | Direct expressions in Code node |

### ❌ Wrong Expression Syntax
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

// ✅ CORRECT: Nullish coalescing or undefined keyword
{{ $json.value ?? "empty" }}
```

### ❌ Code Node Mistakes
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

### ❌ Sub-node Expression Behavior
In AI sub-nodes, expressions **always resolve to first item only**:
```javascript
// In regular nodes: {{ $json.name }} runs for each item
// In sub-nodes: {{ $json.name }} ALWAYS returns first item's value
```

### ❌ Static Data in Manual Executions
`$getWorkflowStaticData()` does **NOT persist** during manual test runs—only works in active/triggered workflows.

### ❌ Simple Memory Node in Queue Mode
Does not work in production Queue Mode (workers don't share memory). Use **Redis Chat Memory** or **Postgres Chat Memory** instead.

---

## Cloud vs self-hosted differences

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

## Quick reference: node availability

### Core Nodes (Always Available)
Manual Trigger, Schedule Trigger, Webhook, HTTP Request, Code, Edit Fields (Set), Filter, IF, Switch, Merge, Loop Over Items, Wait, Limit, Sort, Aggregate, Summarize, Compare Datasets, Execute Workflow, Error Trigger, Stop And Error, Respond to Webhook, No Operation

### Commonly Used Integration Nodes
PostgreSQL, MySQL, Supabase, Google Drive, Google Sheets, Gmail, Slack, HubSpot, Salesforce, AWS S3, Airtable, Notion, Discord, Telegram, OpenAI, Anthropic

This reference reflects n8n Cloud functionality as of January 2025. Always verify specific node parameters against the official documentation at docs.n8n.io for the most current information.