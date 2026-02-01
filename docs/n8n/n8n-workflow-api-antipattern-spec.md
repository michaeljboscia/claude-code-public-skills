# n8n Workflow API Anti-Pattern Specification for Claude Code

## Document Metadata
- **Created**: 2025-01-21T15:30:00Z
- **Version**: 1.0.0
- **Purpose**: Prevent "Could not find workflow / Could not find property option" UI rendering failures when creating n8n workflows programmatically via API
- **Applies To**: Claude Code, AI agents, any programmatic n8n workflow generation
- **Reference Bug**: [GitHub Issue #23620](https://github.com/n8n-io/n8n/issues/23620)

---

## Executive Summary

When creating or updating n8n workflows via the Public API, the API may accept and store workflow JSON that the n8n UI cannot render. This causes:

1. Workflow appears **completely blank** in UI (shows "Add first step...")
2. Workflow name reverts to generic name like "My workflow 15"
3. Error toast: **"Could not find workflow"** with subtitle **"Could not find property option"**
4. Workflow data IS stored correctly (GET API returns all nodes/connections)
5. Workflow CANNOT be executed because UI cannot load it

**Root Cause**: The n8n API performs looser validation than the UI. The UI validates node properties against stricter schemas that include option values, parameter structures, and typeVersion-specific requirements that the API does not enforce.

---

## Part 1: The Problem in Depth

### 1.1 What Happens

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   Claude Code       │     │    n8n API          │     │    n8n UI           │
│   Generates JSON    │────▶│    Accepts JSON     │────▶│    FAILS to render  │
│                     │     │    Returns 200 OK   │     │    "Could not find  │
│                     │     │    Stores in DB     │     │     property option"│
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
```

### 1.2 Why It Happens

The n8n architecture has two separate validation layers:

| Layer | Validates | Strictness |
|-------|-----------|------------|
| **API Layer** | JSON structure, required fields, basic types | Loose |
| **UI Layer** | Node schemas, parameter options, typeVersion compatibility, option values | Strict |

The UI attempts to:
1. Load the node definition for the specified `type` and `typeVersion`
2. Validate each parameter value against the node's schema
3. Verify option values exist in dropdowns/selects
4. Render the visual node representation

If ANY of these fail, the entire workflow fails to render.

### 1.3 The Validation Gap

**What API Accepts (loose)**:
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "parameters": {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {"id": "e1", "name": "foo", "value": "bar", "type": "string"}
      ]
    }
  }
}
```

**What UI Expects (strict)**:
- typeVersion 3.4 must actually exist for this node
- The `mode` value "manual" must be a valid option in the node's dropdown
- The `assignments.assignments` structure must match the exact schema for this typeVersion
- The `type: "string"` must be a valid option in the type dropdown
- Each `id` must be in the expected format

**The API doesn't validate any of this. The UI does.**

---

## Part 2: Comprehensive Anti-Patterns

### 2.1 CRITICAL: typeVersion Hallucination

**THE MOST DANGEROUS ANTI-PATTERN**

AI models frequently hallucinate typeVersion numbers that don't exist.

#### ❌ NEVER DO THIS
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4  // ← AI hallucinated this. Does 3.4 exist? Maybe not!
}
```

```json
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3.2  // ← Also potentially hallucinated
}
```

#### ✅ SAFE PATTERN: Use Known-Good Versions

**Verified Working typeVersions (as of n8n 2.x)**:

| Node Type | Safe typeVersion | Notes |
|-----------|------------------|-------|
| `n8n-nodes-base.set` | `3.4` | Confirmed in bug report |
| `n8n-nodes-base.switch` | `3.2` | Confirmed in bug report, BUT still failed |
| `n8n-nodes-base.httpRequest` | `4.2` | Confirmed in multiple working examples |
| `n8n-nodes-base.code` | `2` | Confirmed working |
| `n8n-nodes-base.merge` | `3`, `3.1` | Both seen in working examples |
| `n8n-nodes-base.if` | `2` | Confirmed working |
| `n8n-nodes-base.webhook` | `2` | Confirmed working |
| `n8n-nodes-base.scheduleTrigger` | `1.2` | Confirmed working |
| `n8n-nodes-base.manualTrigger` | `1` | Confirmed working |
| `n8n-nodes-base.executeWorkflowTrigger` | `1.1` | From bug report (but failed) |
| `n8n-nodes-base.telegram` | `1.2` | From bug report (but failed) |

**CRITICAL INSIGHT**: Even these "confirmed" versions failed in the bug report when created via API. The issue is NOT just typeVersion—it's the entire parameter structure.

#### ✅ BEST PRACTICE: Query Instance for Versions

```javascript
// Before creating workflow, query the n8n instance
async function getSafeTypeVersion(nodeType) {
  // Option 1: Fetch an existing workflow that uses this node
  const existingWorkflows = await fetch('/api/v1/workflows');
  // Find a workflow that contains this node type
  // Extract the typeVersion from that working node
  
  // Option 2: Create a minimal workflow via UI first (manual step)
  // Then fetch it to get the correct typeVersion
}
```

---

### 2.2 CRITICAL: Parameter Schema Mismatches

The parameter structure must EXACTLY match what the UI expects for that specific typeVersion.

#### ❌ NEVER DO THIS: Guess Parameter Structure
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "parameters": {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {
          "id": "e1",
          "name": "texto",
          "value": "={{ $json.text }}",
          "type": "string"  // Is this the right structure?
        }
      ]
    },
    "includeOtherFields": true,
    "options": {}
  }
}
```

The structure above LOOKS reasonable but may not match the exact schema.

#### ✅ SAFE PATTERN: Use Export-as-Template Approach

**Golden Rule**: Never generate parameter structures from scratch. Always derive from a working UI-created workflow.

```javascript
// SAFE WORKFLOW CREATION PATTERN
async function createWorkflowSafely(n8nClient, workflowSpec) {
  // Step 1: Fetch a known-working template
  const template = await n8nClient.get('/api/v1/workflows/KNOWN_GOOD_TEMPLATE_ID');
  
  // Step 2: Clone the structure, only modifying VALUES not STRUCTURE
  const newWorkflow = JSON.parse(JSON.stringify(template.data));
  
  // Step 3: Modify only the values within the existing structure
  newWorkflow.name = workflowSpec.name;
  
  // Find and modify specific nodes by name
  const setNode = newWorkflow.nodes.find(n => n.type === 'n8n-nodes-base.set');
  if (setNode && workflowSpec.setValues) {
    // Modify values within existing structure, don't replace structure
    setNode.parameters.assignments.assignments[0].value = workflowSpec.setValues[0];
  }
  
  // Step 4: POST the modified template
  return await n8nClient.post('/api/v1/workflows', newWorkflow);
}
```

---

### 2.3 CRITICAL: Invalid Option Values

Many node parameters are dropdowns with specific allowed values. The API doesn't validate these.

#### ❌ NEVER DO THIS: Guess Option Values
```json
{
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "parameters": {
    "inputSource": "passthrough"  // Is "passthrough" a valid option? Who knows!
  }
}
```

```json
{
  "type": "n8n-nodes-base.switch",
  "parameters": {
    "rules": {
      "rules": [
        {
          "operator": {
            "type": "string",
            "operation": "equals"  // Valid operations: equals, notEquals, contains, etc.
          }
        }
      ]
    },
    "options": {
      "fallbackOutput": "extra"  // Is "extra" valid? Maybe it's "none" or "last"?
    }
  }
}
```

#### ✅ SAFE PATTERN: Document Known-Good Option Values

**Verified Option Values**:

##### n8n-nodes-base.if / n8n-nodes-base.switch Operators
```javascript
const VALID_STRING_OPERATORS = [
  'equals',
  'notEquals',
  'contains',
  'notContains',
  'startsWith',
  'endsWith',
  'regex',
  'notRegex',
  'empty',
  'notEmpty'
];

const VALID_NUMBER_OPERATORS = [
  'equals',
  'notEquals',
  'gt',      // greater than
  'gte',     // greater than or equal
  'lt',      // less than
  'lte',     // less than or equal
  'empty',
  'notEmpty'
];

const VALID_BOOLEAN_OPERATORS = [
  'true',
  'false',
  'empty',
  'notEmpty'
];
```

##### n8n-nodes-base.set Mode Options
```javascript
const VALID_SET_MODES = [
  'manual',
  'raw'
];
```

##### n8n-nodes-base.merge Mode Options
```javascript
const VALID_MERGE_MODES = [
  'append',
  'combine',
  'chooseBranch',
  'mergeByIndex',
  'mergeByKey'
];
```

##### Condition Options Version
```javascript
// The "version" inside conditions.options MUST match typeVersion expectations
const VALID_CONDITION_OPTIONS = {
  caseSensitive: true,  // boolean
  leftValue: "",        // Can be empty string
  typeValidation: "strict",  // or "loose"
  version: 2            // CRITICAL: Must match what UI expects
};
```

---

### 2.4 CRITICAL: Connection Name Mismatches

Connections reference nodes by their `name` property. Any mismatch = broken workflow.

#### ❌ NEVER DO THIS: Inconsistent Naming
```json
{
  "nodes": [
    {
      "name": "When Executed by Another Workflow",  // ← Space in name
      "type": "n8n-nodes-base.executeWorkflowTrigger"
    },
    {
      "name": "Extrair Comando",  // ← Special character (accented)
      "type": "n8n-nodes-base.set"
    }
  ],
  "connections": {
    "When Executed by Another Workflow": {  // Must EXACTLY match
      "main": [[{"node": "Extrair Comando", "type": "main", "index": 0}]]
    }
  }
}
```

#### ✅ SAFE PATTERN: Simple ASCII Names, Validate Before Submit
```javascript
// SAFE NAMING CONVENTIONS
const SAFE_NODE_NAME_PATTERN = /^[a-zA-Z][a-zA-Z0-9_]*$/;

function validateNodeName(name) {
  // Prefer simple names
  if (!SAFE_NODE_NAME_PATTERN.test(name.replace(/\s+/g, '_'))) {
    console.warn(`Node name "${name}" may cause issues`);
  }
}

// VALIDATE CONNECTIONS MATCH NODES
function validateConnections(workflow) {
  const nodeNames = new Set(workflow.nodes.map(n => n.name));
  
  for (const [sourceName, connections] of Object.entries(workflow.connections)) {
    if (!nodeNames.has(sourceName)) {
      throw new Error(`Connection source "${sourceName}" not found in nodes`);
    }
    
    for (const outputs of connections.main || []) {
      for (const target of outputs) {
        if (!nodeNames.has(target.node)) {
          throw new Error(`Connection target "${target.node}" not found in nodes`);
        }
      }
    }
  }
}
```

---

### 2.5 CRITICAL: Missing Required Fields

Each node must have ALL required fields, even if they seem optional.

#### ❌ NEVER DO THIS: Omit Fields
```json
{
  "nodes": [
    {
      "name": "My Node",
      "type": "n8n-nodes-base.set",
      "parameters": {}
      // Missing: id, typeVersion, position
    }
  ]
}
```

#### ✅ SAFE PATTERN: Include All Required Fields
```json
{
  "nodes": [
    {
      "id": "unique-uuid-here",           // REQUIRED: Unique identifier
      "name": "My Node",                   // REQUIRED: Display name
      "type": "n8n-nodes-base.set",        // REQUIRED: Node type
      "typeVersion": 3.4,                  // REQUIRED: Version number
      "position": [250, 300],              // REQUIRED: [x, y] coordinates
      "parameters": {}                     // REQUIRED: Even if empty
    }
  ]
}
```

**Required Node Fields**:
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique UUID for this node instance |
| `name` | string | Display name (must be unique in workflow) |
| `type` | string | Full node type identifier |
| `typeVersion` | number | Node version number |
| `position` | [number, number] | Canvas position [x, y] |
| `parameters` | object | Node configuration (can be empty {}) |

**Optional But Recommended**:
| Field | Type | Description |
|-------|------|-------------|
| `credentials` | object | Credential references |
| `webhookId` | string | For webhook nodes only |

---

### 2.6 CRITICAL: Metadata and Settings

Modern n8n versions (1.6+) expect certain metadata fields.

#### ❌ NEVER DO THIS: Omit Metadata
```json
{
  "name": "My Workflow",
  "nodes": [...],
  "connections": {}
}
```

#### ✅ SAFE PATTERN: Include Full Metadata
```json
{
  "name": "My Workflow",
  "nodes": [...],
  "connections": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "meta": {
    "templateCredsSetupCompleted": true
  },
  "pinData": {}
}
```

**Required Workflow Fields**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Workflow display name |
| `nodes` | array | Yes | Array of node objects |
| `connections` | object | Yes | Connection mappings |
| `active` | boolean | Recommended | Workflow enabled state |
| `settings` | object | Recommended | Execution settings |
| `settings.executionOrder` | string | Recommended | Should be "v1" |
| `meta` | object | Recommended | Metadata |
| `pinData` | object | Recommended | Pinned test data |

---

### 2.7 MODERATE: ID Format Issues

Node IDs should follow UUID format for safety.

#### ❌ RISKY: Custom ID Formats
```json
{
  "id": "trigger-001",      // Custom format - may cause issues
  "id": "set-001",          // Custom format
  "id": "switch-001"        // Custom format
}
```

#### ✅ SAFE PATTERN: Use UUIDs
```javascript
const crypto = require('crypto');

function generateNodeId() {
  return crypto.randomUUID();
  // Returns: "550e8400-e29b-41d4-a716-446655440000"
}
```

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

### 2.8 MODERATE: Nested Condition Structure

The IF and Switch nodes have particularly complex parameter structures.

#### ❌ DANGEROUS: Malformed Condition Structure
```json
{
  "type": "n8n-nodes-base.switch",
  "parameters": {
    "rules": {
      "rules": [
        {
          "outputKey": "bs",
          "conditions": {
            "options": {
              "version": 2,  // What version? For what?
              "combinator": "and"
            },
            "conditions": [
              {
                "id": "c1",
                "leftValue": "={{ $json.comando }}",
                "rightValue": "/bs",
                "operator": {
                  "type": "string",
                  "operation": "equals"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

This structure is from the bug report—it looks reasonable but FAILS in UI.

#### ✅ SAFE PATTERN: Copy from Working UI Export

The ONLY safe way to get condition structures:

1. Manually create the condition in n8n UI
2. Export the workflow
3. Copy the exact JSON structure
4. Use that as your template

**Example of UI-exported IF condition (verified working)**:
```json
{
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "leftValue": "",
        "typeValidation": "strict",
        "version": 2
      },
      "conditions": [
        {
          "id": "64159b0b-7a8c-4def-a4a1-e8f5a00fe66d",
          "leftValue": "={{ $json.probClean }}",
          "rightValue": 75,
          "operator": {
            "type": "number",
            "operation": "gte"
          }
        }
      ],
      "combinator": "and"
    },
    "options": {}
  },
  "type": "n8n-nodes-base.if",
  "typeVersion": 2
}
```

**Key Differences from Bug Report Version**:
- `options.version` is at the CORRECT nesting level
- `combinator` is at the TOP level of conditions, not inside options
- IDs are proper UUIDs
- typeVersion is `2` not a complex decimal

---

## Part 3: The Golden Template Approach

### 3.1 Core Principle

**NEVER generate n8n workflow JSON from scratch. ALWAYS derive from working templates.**

### 3.2 Implementation Strategy

```javascript
/**
 * SAFE WORKFLOW CREATION SYSTEM
 * 
 * This system prevents the "Could not find property option" error
 * by NEVER generating workflow JSON from scratch.
 */

class SafeN8nWorkflowBuilder {
  constructor(n8nClient, templateLibraryId) {
    this.client = n8nClient;
    this.templateLibraryId = templateLibraryId;
    this.templateCache = new Map();
  }

  /**
   * Step 1: Build a template library via UI
   * 
   * Create these workflows MANUALLY in the n8n UI, then note their IDs:
   * - "TEMPLATE_BASIC_HTTP" - Simple HTTP request workflow
   * - "TEMPLATE_IF_BRANCH" - Workflow with IF branching
   * - "TEMPLATE_SWITCH" - Workflow with Switch node
   * - "TEMPLATE_SET_DATA" - Workflow with Set node
   * - "TEMPLATE_LOOP" - Workflow with loop construct
   * - "TEMPLATE_ERROR_HANDLING" - Workflow with error handling
   */
  
  /**
   * Step 2: Fetch and cache templates
   */
  async loadTemplate(templateName) {
    if (this.templateCache.has(templateName)) {
      return JSON.parse(JSON.stringify(this.templateCache.get(templateName)));
    }
    
    const templateId = this.getTemplateId(templateName);
    const response = await this.client.get(`/api/v1/workflows/${templateId}`);
    
    if (!response.data) {
      throw new Error(`Template ${templateName} not found`);
    }
    
    this.templateCache.set(templateName, response.data);
    return JSON.parse(JSON.stringify(response.data));
  }
  
  /**
   * Step 3: Build new workflow from template
   * 
   * @param {string} templateName - Which template to use
   * @param {object} modifications - What to change (VALUES ONLY)
   */
  async buildWorkflow(templateName, modifications) {
    const workflow = await this.loadTemplate(templateName);
    
    // Generate new IDs
    workflow.id = undefined; // Let n8n generate
    workflow.name = modifications.name || workflow.name;
    
    // Regenerate all node IDs
    const idMap = new Map();
    for (const node of workflow.nodes) {
      const oldId = node.id;
      const newId = this.generateUUID();
      idMap.set(oldId, newId);
      node.id = newId;
    }
    
    // Apply value modifications (NOT structural changes)
    if (modifications.nodeValues) {
      for (const [nodeName, values] of Object.entries(modifications.nodeValues)) {
        const node = workflow.nodes.find(n => n.name === nodeName);
        if (node) {
          this.applyValueChanges(node.parameters, values);
        }
      }
    }
    
    // Validate before returning
    this.validateWorkflow(workflow);
    
    return workflow;
  }
  
  /**
   * Apply value changes WITHOUT modifying structure
   */
  applyValueChanges(target, changes) {
    for (const [key, value] of Object.entries(changes)) {
      if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
        if (target[key] && typeof target[key] === 'object') {
          this.applyValueChanges(target[key], value);
        }
        // Don't create new nested objects - that would change structure
      } else {
        // Only modify existing keys
        if (key in target) {
          target[key] = value;
        }
      }
    }
  }
  
  /**
   * Validate workflow before submission
   */
  validateWorkflow(workflow) {
    // Check all required fields
    if (!workflow.name) throw new Error('Workflow name required');
    if (!workflow.nodes || !Array.isArray(workflow.nodes)) {
      throw new Error('Nodes array required');
    }
    
    const nodeNames = new Set();
    for (const node of workflow.nodes) {
      // Check required node fields
      if (!node.id) throw new Error(`Node missing id`);
      if (!node.name) throw new Error(`Node missing name`);
      if (!node.type) throw new Error(`Node ${node.name} missing type`);
      if (node.typeVersion === undefined) {
        throw new Error(`Node ${node.name} missing typeVersion`);
      }
      if (!node.position || !Array.isArray(node.position)) {
        throw new Error(`Node ${node.name} missing position`);
      }
      if (!node.parameters) {
        throw new Error(`Node ${node.name} missing parameters`);
      }
      
      // Check for duplicate names
      if (nodeNames.has(node.name)) {
        throw new Error(`Duplicate node name: ${node.name}`);
      }
      nodeNames.add(node.name);
    }
    
    // Validate connections
    if (workflow.connections) {
      for (const [sourceName, connections] of Object.entries(workflow.connections)) {
        if (!nodeNames.has(sourceName)) {
          throw new Error(`Connection source "${sourceName}" not in nodes`);
        }
        
        for (const outputs of connections.main || []) {
          for (const target of outputs) {
            if (!nodeNames.has(target.node)) {
              throw new Error(`Connection target "${target.node}" not in nodes`);
            }
          }
        }
      }
    }
    
    return true;
  }
  
  generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = Math.random() * 16 | 0;
      const v = c === 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    });
  }
  
  getTemplateId(templateName) {
    // Map template names to workflow IDs in your n8n instance
    const templates = {
      'TEMPLATE_BASIC_HTTP': 'your-http-template-id',
      'TEMPLATE_IF_BRANCH': 'your-if-template-id',
      // ... etc
    };
    return templates[templateName];
  }
}
```

### 3.3 Template Library Specification

Create these templates MANUALLY in n8n UI:

#### Template: BASIC_HTTP
```
Manual Trigger → HTTP Request → Set (format response)
```

#### Template: IF_BRANCH
```
Trigger → IF → [True Branch] → Response
              → [False Branch] → Response
```

#### Template: SWITCH_MULTI
```
Trigger → Switch → [Branch 1] → Action
                 → [Branch 2] → Action
                 → [Default] → Action
```

#### Template: DATA_TRANSFORM
```
Trigger → HTTP Request → Set → Code → Respond
```

#### Template: ERROR_HANDLER
```
Trigger → Try (sub-workflow) → Success Response
                             → Error Handler → Error Response
```

---

## Part 4: Workaround Strategies

### 4.1 Create-Then-Patch Strategy

Instead of creating complete workflows via API:

```javascript
async function createWorkflowSafely(n8nClient, spec) {
  // Step 1: Create minimal workflow via API
  const minimal = {
    name: spec.name,
    nodes: [
      {
        id: generateUUID(),
        name: 'Manual Trigger',
        type: 'n8n-nodes-base.manualTrigger',
        typeVersion: 1,
        position: [250, 300],
        parameters: {}
      }
    ],
    connections: {},
    active: false,
    settings: { executionOrder: 'v1' }
  };
  
  const created = await n8nClient.post('/api/v1/workflows', minimal);
  const workflowId = created.data.id;
  
  // Step 2: Open in UI to verify it renders
  console.log(`Verify workflow renders: ${n8nClient.baseUrl}/workflow/${workflowId}`);
  
  // Step 3: If you MUST add more nodes, use the import endpoint
  // or guide user to add nodes via UI
  
  return workflowId;
}
```

### 4.2 UI Import Endpoint (If Available)

Some n8n versions have a separate import endpoint that may handle validation differently:

```javascript
// Try the import endpoint instead of create
async function importWorkflow(n8nClient, workflow) {
  try {
    // This endpoint may perform UI-like validation
    return await n8nClient.post('/api/v1/workflows/import', workflow);
  } catch (error) {
    console.error('Import failed, trying create endpoint');
    return await n8nClient.post('/api/v1/workflows', workflow);
  }
}
```

### 4.3 Verification Step

After creating ANY workflow via API, verify it renders:

```javascript
async function verifyWorkflowRenders(n8nClient, workflowId) {
  // Fetch the workflow
  const workflow = await n8nClient.get(`/api/v1/workflows/${workflowId}`);
  
  // Check for signs of corruption
  const checks = {
    hasName: !!workflow.data.name,
    hasNodes: workflow.data.nodes?.length > 0,
    hasConnections: Object.keys(workflow.data.connections || {}).length > 0,
    allNodesHaveIds: workflow.data.nodes?.every(n => n.id),
    allNodesHaveTypes: workflow.data.nodes?.every(n => n.type),
  };
  
  const failed = Object.entries(checks).filter(([k, v]) => !v);
  if (failed.length > 0) {
    console.error('Workflow may not render. Failed checks:', failed);
    return false;
  }
  
  // The ONLY way to truly verify is to attempt UI access
  console.log(`Manual verification required: ${n8nClient.baseUrl}/workflow/${workflowId}`);
  return true;
}
```

---

## Part 5: Safe Node Reference Library

### 5.1 Manual Trigger (SAFEST - Start Here)

```json
{
  "id": "uuid-here",
  "name": "Manual Trigger",
  "type": "n8n-nodes-base.manualTrigger",
  "typeVersion": 1,
  "position": [250, 300],
  "parameters": {}
}
```

### 5.2 Schedule Trigger

```json
{
  "id": "uuid-here",
  "name": "Schedule Trigger",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [250, 300],
  "parameters": {
    "rule": {
      "interval": [
        {
          "triggerAtHour": 6
        }
      ]
    }
  }
}
```

### 5.3 HTTP Request (Simple GET)

```json
{
  "id": "uuid-here",
  "name": "HTTP Request",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [450, 300],
  "parameters": {
    "url": "https://api.example.com/data",
    "options": {}
  }
}
```

### 5.4 HTTP Request (With Auth)

```json
{
  "id": "uuid-here",
  "name": "HTTP Request",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [450, 300],
  "parameters": {
    "url": "https://api.example.com/data",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpBasicAuth",
    "options": {}
  },
  "credentials": {
    "httpBasicAuth": {
      "id": "credential-id-here",
      "name": "My API Credentials"
    }
  }
}
```

### 5.5 Code Node

```json
{
  "id": "uuid-here",
  "name": "Code",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [650, 300],
  "parameters": {
    "jsCode": "// Your code here\nreturn $input.all();"
  }
}
```

### 5.6 Merge Node

```json
{
  "id": "uuid-here",
  "name": "Merge",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3,
  "position": [650, 300],
  "parameters": {}
}
```

### 5.7 Webhook (Trigger)

```json
{
  "id": "uuid-here",
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [250, 300],
  "parameters": {
    "httpMethod": "POST",
    "path": "my-webhook-path",
    "options": {}
  },
  "webhookId": "unique-webhook-id"
}
```

---

## Part 6: Diagnostic Tools

### 6.1 Pre-Submit Validator

```javascript
/**
 * Validate workflow JSON before submitting to n8n API
 * Returns array of warnings/errors
 */
function validateWorkflowJSON(workflow) {
  const issues = [];
  
  // Check top-level structure
  if (!workflow.name) {
    issues.push({ level: 'error', message: 'Missing workflow name' });
  }
  
  if (!Array.isArray(workflow.nodes)) {
    issues.push({ level: 'error', message: 'Nodes must be an array' });
    return issues; // Can't continue
  }
  
  if (workflow.nodes.length === 0) {
    issues.push({ level: 'warning', message: 'Workflow has no nodes' });
  }
  
  // Check each node
  const nodeNames = new Set();
  const nodeIds = new Set();
  
  for (let i = 0; i < workflow.nodes.length; i++) {
    const node = workflow.nodes[i];
    const prefix = `Node[${i}]`;
    
    // Required fields
    if (!node.id) {
      issues.push({ level: 'error', message: `${prefix}: Missing id` });
    } else if (nodeIds.has(node.id)) {
      issues.push({ level: 'error', message: `${prefix}: Duplicate id ${node.id}` });
    } else {
      nodeIds.add(node.id);
    }
    
    if (!node.name) {
      issues.push({ level: 'error', message: `${prefix}: Missing name` });
    } else if (nodeNames.has(node.name)) {
      issues.push({ level: 'error', message: `${prefix}: Duplicate name "${node.name}"` });
    } else {
      nodeNames.add(node.name);
    }
    
    if (!node.type) {
      issues.push({ level: 'error', message: `${prefix}: Missing type` });
    }
    
    if (node.typeVersion === undefined) {
      issues.push({ level: 'error', message: `${prefix}: Missing typeVersion` });
    }
    
    if (!node.position || !Array.isArray(node.position) || node.position.length !== 2) {
      issues.push({ level: 'error', message: `${prefix}: Invalid position` });
    }
    
    if (node.parameters === undefined) {
      issues.push({ level: 'warning', message: `${prefix}: Missing parameters object` });
    }
    
    // Check for potentially problematic typeVersions
    const knownVersions = {
      'n8n-nodes-base.manualTrigger': [1],
      'n8n-nodes-base.scheduleTrigger': [1, 1.1, 1.2],
      'n8n-nodes-base.httpRequest': [1, 2, 3, 4, 4.1, 4.2],
      'n8n-nodes-base.code': [1, 2],
      'n8n-nodes-base.set': [1, 2, 3, 3.4],
      'n8n-nodes-base.if': [1, 2],
      'n8n-nodes-base.switch': [1, 2, 3, 3.2],
      'n8n-nodes-base.merge': [1, 2, 3, 3.1],
      'n8n-nodes-base.webhook': [1, 1.1, 2],
    };
    
    if (node.type && knownVersions[node.type]) {
      if (!knownVersions[node.type].includes(node.typeVersion)) {
        issues.push({
          level: 'warning',
          message: `${prefix}: typeVersion ${node.typeVersion} not in known list for ${node.type}`
        });
      }
    }
  }
  
  // Check connections
  if (workflow.connections) {
    for (const [sourceName, connections] of Object.entries(workflow.connections)) {
      if (!nodeNames.has(sourceName)) {
        issues.push({
          level: 'error',
          message: `Connection source "${sourceName}" not found in nodes`
        });
      }
      
      if (connections.main) {
        for (const outputs of connections.main) {
          for (const target of outputs) {
            if (!nodeNames.has(target.node)) {
              issues.push({
                level: 'error',
                message: `Connection target "${target.node}" not found in nodes`
              });
            }
          }
        }
      }
    }
  }
  
  return issues;
}
```

### 6.2 Post-Create Diagnostic

```javascript
/**
 * After creating workflow, check if it's likely to render
 */
async function diagnoseWorkflow(n8nClient, workflowId) {
  const response = await n8nClient.get(`/api/v1/workflows/${workflowId}`);
  const workflow = response.data;
  
  const diagnosis = {
    workflowId,
    retrieved: true,
    name: workflow.name,
    nodeCount: workflow.nodes?.length || 0,
    connectionCount: Object.keys(workflow.connections || {}).length,
    issues: [],
    recommendation: null
  };
  
  // Check if name looks like it was reset
  if (workflow.name.match(/^My workflow \d+$/)) {
    diagnosis.issues.push('Name appears to have been reset to default');
    diagnosis.recommendation = 'Workflow may not render correctly in UI';
  }
  
  // Check for empty nodes
  if (diagnosis.nodeCount === 0) {
    diagnosis.issues.push('No nodes in workflow');
  }
  
  // Check for orphaned connections
  const nodeNames = new Set(workflow.nodes?.map(n => n.name) || []);
  for (const sourceName of Object.keys(workflow.connections || {})) {
    if (!nodeNames.has(sourceName)) {
      diagnosis.issues.push(`Orphaned connection from "${sourceName}"`);
    }
  }
  
  if (diagnosis.issues.length === 0) {
    diagnosis.recommendation = 'Workflow appears valid, verify in UI';
  } else {
    diagnosis.recommendation = 'Workflow has issues, consider recreating';
  }
  
  return diagnosis;
}
```

---

## Part 7: Claude Code Specific Instructions

### 7.1 Before Generating Any n8n Workflow

```
STOP. Before generating n8n workflow JSON, follow this checklist:

1. □ Do I have a working template to base this on?
   - If NO: Ask user to provide a working workflow export from their n8n instance
   - If YES: Proceed to step 2

2. □ Am I modifying VALUES or STRUCTURE?
   - Only modify VALUES within existing parameter structures
   - NEVER add new keys to parameter objects
   - NEVER change the nesting/hierarchy of parameters

3. □ For each node I'm creating:
   - □ Is the typeVersion from the template or verified list?
   - □ Are all parameters from the template structure?
   - □ Is the ID a proper UUID format?
   - □ Are position coordinates reasonable numbers?

4. □ For connections:
   - □ Does every source name exactly match a node name?
   - □ Does every target name exactly match a node name?
   - □ Are there no typos or encoding issues in names?

5. □ Before returning JSON:
   - □ Run the validateWorkflowJSON function
   - □ Fix any errors
   - □ Warn user about any warnings
```

### 7.2 Response Template for Workflow Requests

When user asks Claude Code to create an n8n workflow:

```
I'll help you create this n8n workflow. However, I need to follow safe practices to avoid the "Could not find property option" rendering bug.

**IMPORTANT**: Workflows created via API can fail to render in the UI even when the API accepts them. To prevent this:

1. **Safest approach**: I'll create a minimal workflow with just a trigger node. You can then add the remaining nodes via the n8n UI.

2. **Template approach**: If you have an existing workflow that uses similar nodes, share its JSON export and I'll modify that instead of creating from scratch.

3. **Risk acknowledgment**: If you want me to generate the full workflow, understand that it may fail to render in the UI. You would need to:
   - Verify it loads in the UI after creation
   - Potentially recreate it manually if it doesn't render

Which approach would you prefer?
```

### 7.3 If User Insists on Full Generation

```
Understood. I'll generate the full workflow with these safeguards:

1. Using only verified typeVersion numbers
2. Using minimal parameter structures
3. Including full validation output
4. Providing a diagnostic script to run after creation

**Generated Workflow**: [JSON here]

**Validation Results**: [Results here]

**Post-Creation Steps**:
1. After API returns 200, immediately open: /workflow/{id}
2. If you see "Could not find workflow" error, the workflow needs manual recreation
3. If it renders, verify each node opens without errors

**Fallback**: If this fails, I've also prepared a minimal version with just the trigger that you can enhance via UI.
```

---

## Part 8: Quick Reference Card

### DO ✅

- Derive workflow JSON from UI-exported templates
- Use UUIDs for node IDs
- Include all required fields (id, name, type, typeVersion, position, parameters)
- Validate connections match node names exactly
- Use verified typeVersions from known-good list
- Include metadata fields (settings.executionOrder, meta)
- Verify workflow renders in UI after API creation

### DON'T ❌

- Generate workflow JSON from scratch
- Invent typeVersion numbers
- Guess parameter structures
- Assume option values (mode, operation, fallbackOutput)
- Omit any required fields
- Use special characters in node names
- Skip post-creation UI verification
- Trust API 200 response means workflow will render

### Verified Safe typeVersions

| Node | Safe Versions |
|------|---------------|
| manualTrigger | 1 |
| scheduleTrigger | 1.2 |
| httpRequest | 4.2 |
| code | 2 |
| merge | 3 |
| webhook | 2 |
| if | 2 |

### Risky Nodes (Complex Schemas)

- `n8n-nodes-base.set` (assignments structure)
- `n8n-nodes-base.switch` (rules structure)
- `n8n-nodes-base.if` (conditions structure)
- Any node with credential requirements
- Any node with resource mapper

**For these nodes: ALWAYS use UI export as template.**

---

## Bibliography

1. GitHub Issue #23620: "Workflows created via API show 'Could not find workflow / Could not find property option' error in UI" - https://github.com/n8n-io/n8n/issues/23620 (December 2025)

2. n8n Documentation: Export and Import Workflows - https://docs.n8n.io/workflows/export-import/

3. n8n Documentation: Node Versioning - https://docs.n8n.io/integrations/creating-nodes/build/reference/node-versioning/

4. n8n Community: "How do I update node typeVersion?" - https://community.n8n.io/t/how-do-i-update-node-typeversion/29617

5. GitHub Issue #19530: "n8n node throwing error when trying to create a workflow" - https://github.com/n8n-io/n8n/issues/19530

6. GitHub Issue #16377: "'n8n' Node Fails to Update Workflow with 'Cannot be parsed' Error" - https://github.com/n8n-io/n8n/issues/16377

7. Latenode Blog: "N8N Export/Import Workflows: Complete JSON Guide + Troubleshooting Common Failures 2025" - https://latenode.com/blog/low-code-no-code-platforms/n8n-setup-workflows-self-hosting-templates/n8n-export-import-workflows-complete-json-guide-troubleshooting-common-failures-2025

8. Latenode Blog: "N8N Import Workflow JSON: Complete Guide + File Format Examples 2025" - https://latenode.com/blog/low-code-no-code-platforms/n8n-setup-workflows-self-hosting-templates/n8n-import-workflow-json-complete-guide-file-format-examples-2025

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-01-21T15:30:00Z | Initial comprehensive specification |
