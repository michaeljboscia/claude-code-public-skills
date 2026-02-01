---
name: n8n-workflow-development-lessons
description: Hard-won lessons from real n8n projects - debugging patterns, anti-patterns, process improvements
shareable: true
---

# n8n Workflow Development: Lessons Learned
**Last Updated:** 2026-01-26
**Context:** Sequence-to-Queue System Migration (Phases 1 & 2), PSI Diagnostic Workflows Bug Fix, Instantly-HubSpot Engagements Integration

---

## Critical Technical Lessons

### 1. NEVER Use HTTP Request Nodes for Complex JSON with Expressions

**Problem:** n8n's HTTP Request node cannot reliably evaluate complex expressions in JSON body parameter.

**Symptoms:**
- Error: "JSON parameter needs to be valid JSON"
- Expressions like `={{ }}` fail
- Template syntax inside JSON fails
- Single-line JavaScript objects fail
- jsonString body type still fails

**Solution:** Use Code nodes with `this.helpers.httpRequest` instead.

**Example - WRONG (HTTP Request node):**
```javascript
// In HTTP Request node JSON Body field:
={{
  campaign: $('Previous Node').item.json.id,
  ...$('Other Node').item.json.lead_payload,
  skip_if_in_campaign: true
}}
```

**Example - RIGHT (Code node):**
```javascript
const prepData = $('Previous Node').item.json;

const response = await this.helpers.httpRequest({
  method: 'POST',
  url: 'https://api.example.com/endpoint',
  headers: {
    'Authorization': 'Bearer YOUR_TOKEN',
    'Content-Type': 'application/json'
  },
  body: {
    campaign: prepData.campaign_id,
    email: prepData.email,
    first_name: prepData.first_name,
    skip_if_in_campaign: true
  },
  json: true
});

return [{ json: response }];
```

**Why This Matters:**
- Saves hours of debugging expression syntax
- Gives full JavaScript control over body construction
- Easier to debug (console.log works in Code nodes)
- More readable and maintainable

**When to Use HTTP Request Nodes:**
- Simple GET requests with no body
- Static JSON bodies with no expressions
- URL parameters only

---

### 2. Check Database Constraints BEFORE Writing Application Code

**Problem:** Application code used values that violated database check constraints.

**What Happened:**
- Parser mapped `LINKEDIN_ENGAGEMENT` → `linkedin_profile_visit`
- Database constraint only allowed `linkedin_visit`
- Runtime error after workflow deployed

**Prevention:**
```sql
-- ALWAYS run this query before writing application code:
SELECT
  constraint_name,
  check_clause
FROM information_schema.check_constraints
WHERE table_name = 'your_table_name';
```

**Then:** Match application code to database constraints, not your assumptions.

**Lesson:** Database is source of truth. Code must conform to database, not vice versa.

---

### 3. Node Connection Order Matters for Data Flow

**Problem:** Parser running before parent record creation caused duplicate parent records.

**Wrong Flow:**
```
Extract Data → Parse Multi-Touch → Create Sequence → Insert Queue
```
**Result:** Parser outputs N items → Create Sequence runs N times → N duplicate sequence records

**Correct Flow:**
```
Extract Data → Create Sequence → Parse Multi-Touch → Insert Queue
```
**Result:** Create Sequence runs once → Parser outputs N items → Insert Queue runs N times

**Lesson:** When one node outputs N items and downstream node should run once, put the single-run node BEFORE the N-output node.

**Visual Rule:**
```
1-to-1 node → 1-to-N node → N-to-N node
```

---

### 4. Verify External Property Names Before Building Workflows

**Problem:** Used `ba_content_draft_url` when actual HubSpot property was `ba_sequence_google_doc_url`.

**Prevention:**
1. Query HubSpot API for object properties first
2. Use exact field names from API response
3. Test with real data before building full workflow

**Quick Test:**
```javascript
// In a Code node:
const contact = $('Get Contact').item.json;
console.log(Object.keys(contact.properties));
return [{ json: contact.properties }];
```

**Then:** Check output to see actual property names.

---

### 5. NEVER Match Arrays by Index Across Different Data Sources

**Problem:** Merging data from two nodes by array index caused 13% data loss in PSI diagnostic workflows.

**Date Discovered:** 2026-01-25
**Affected Workflows:** Mobile/Desktop On-Demand Diagnostic Audit

**The Bug:**
```javascript
// WRONG - Index-based matching
const savedRecords = $('Save Core Metrics').all();      // From Supabase (may reorder)
const extractedData = $('Extract Core + Full Response').all();  // Original order

for (let i = 0; i < savedRecords.length; i++) {
  const saved = savedRecords[i].json;
  const extracted = extractedData[i].json;  // ASSUMES SAME INDEX = SAME RECORD
  // ...
}
```

**Why It Fails:**
- Supabase (or any external service) may return items in different order than sent
- If some inserts fail, array lengths differ and indices shift
- Large batches increase probability of misalignment
- Result: Data paired with wrong foreign keys → silent data corruption

**The Fix:**
```javascript
// RIGHT - Key-based matching (use URL, ID, or other unique identifier)
const savedRecords = $('Save Core Metrics').all();
const extractedData = $('Extract Core + Full Response').all();

// Build lookup by unique key
const extractedByUrl = {};
for (const item of extractedData) {
  if (item.json.url) {
    extractedByUrl[item.json.url] = item.json;
  }
}

for (const saved of savedRecords) {
  const extracted = extractedByUrl[saved.json.url];  // Match by URL, not position
  if (extracted) {
    // Now guaranteed to be the same record
  }
}
```

**How to Detect This Bug:**
```sql
-- Find records with scores but missing detail data
SELECT
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE id IN (SELECT fk_id FROM detail_table)) as has_details
FROM parent_table
WHERE score IS NOT NULL;
-- If has_details < total, you may have this bug
```

**Key Signs:**
- Some % of records missing related data (we had 13%)
- Problem affects specific executions, not random records
- Larger batches have higher failure rates
- Multiple downstream tables affected by same merge node

**Lesson:** NEVER assume two arrays from different sources are in the same order. Always match by a unique key field.

---

### 6. JSON.stringify() is Mandatory for User-Generated Content in HTTP Request Nodes

**Date Discovered:** 2026-01-26
**Affected Workflow:** Instantly to HubSpot - Webhook Handler

**Problem:** User-generated content (reply text, email bodies) contains quotes, newlines, and special characters that break JSON templates.

**Symptoms:**
- Error: "JSON parameter needs to be valid JSON"
- Works with test data, fails with real user input
- Intermittent failures based on content

**The Bug:**
```javascript
// WRONG - Direct template interpolation
={
  "properties": {
    "reply_text": "{{ $json.reply_text }}"
  }
}
```

If `reply_text` contains `"Hello\nHow are you?"` the JSON becomes invalid.

**The Fix:**
```javascript
// RIGHT - JSON.stringify escapes everything properly
={
  "properties": {
    "reply_text": {{ JSON.stringify(($json.reply_text || '').substring(0, 256)) }}
  }
}
```

**Pattern Breakdown:**
- `$json.reply_text || ''` - Handle null/undefined
- `.substring(0, 256)` - Truncate to safe length (HubSpot events = 256 char limit)
- `JSON.stringify()` - Escape quotes, newlines, special chars

**When to Use:**
- ANY field that could contain user input
- Email bodies, reply text, notes, comments
- Anything from external webhooks

**Lesson:** Never trust user content in JSON templates. Always escape with JSON.stringify().

---

### 7. HubSpot Email Engagements vs Behavioral Events - Choose Wisely

**Date Discovered:** 2026-01-26
**Context:** Instantly email sync showing as generic "activity" instead of emails

**The Problem:**
- Built workflow using HubSpot Behavioral Events API (`/events/v3/send`)
- Emails showed as generic activity blips in timeline
- User feedback: "these custom events aren't particularly helpful"

**What We Should Have Done:**
- Use HubSpot Engagements API (`/crm/v3/objects/emails`)
- Emails appear as actual email cards with subject, body, direction

**Decision Matrix:**

| Use Case | API to Use |
|----------|-----------|
| Logging actual emails (sent/received) | `/crm/v3/objects/emails` (Engagements) |
| Tracking events (opens, clicks, page views) | `/events/v3/send` (Behavioral Events) |
| Creating notes, tasks, calls | `/crm/v3/objects/{type}` (Engagements) |

**Key Differences:**

| Aspect | Behavioral Events | Email Engagements |
|--------|------------------|-------------------|
| Timeline appearance | Generic activity card | Native email card |
| Property limit | 256 chars | 65KB+ |
| Tracking data | Can include custom props | Has `hs_email_*` props |
| Association | Automatic by contact ID | Explicit via `associations` array |

**Lesson:** Step back and ask "What would native HubSpot do?" before choosing an API.

---

## Process Lessons

### 5. Backup Before Major Rebuilds

**What to Backup:**
- Export workflow JSON via n8n API or UI
- Save to: `~/your-backup-folder/[project]/backups/[workflow-name]-backup-YYYY-MM-DD.json`
- Include date and "before what" in filename

**Why:**
- Enables rollback if rebuild fails
- Serves as documentation of "before" state
- Allows comparison of old vs new approach

**Backup Command Pattern:**
```bash
# For n8n Cloud workflows:
# Use n8n UI: Settings → Export → Download JSON

# Or via API:
curl https://your-n8n-cloud.app.n8n.cloud/api/v1/workflows/WORKFLOW_ID \
  -H "X-N8N-API-KEY: YOUR_KEY" > backup.json
```

---

### 6. Test Early, Test Often, Test Small

**Anti-Pattern:** Build entire workflow, then test.

**Better Pattern:**
1. Build first 2-3 nodes
2. Test with real data
3. Verify output format
4. Build next 2-3 nodes
5. Repeat

**Why:**
- Catches issues early when they're easy to fix
- Prevents compound errors
- Provides confidence each section works
- Easier to debug (know exactly which node failed)

**Example Testing Cadence:**
- Test 1: Get contact data, verify HubSpot properties
- Test 2: Extract doc ID, verify parsing
- Test 3: Read doc, verify text extraction
- Test 4: Parse touches, verify regex matching
- Test 5: Write to database, verify constraint satisfaction
- Test 6: Full end-to-end

---

### 7. User Frustration is a Signal, Not Noise

**Signals That Current Approach is Fundamentally Broken:**
- "READ WHAT IT SAYS DAMNIT"
- "why can't either of you fix this?"
- Multiple attempts at same approach all fail
- Workarounds getting increasingly complex

**What to Do:**
1. STOP trying variations of the broken approach
2. Step back and question the tool/method
3. Search for alternative tools/approaches
4. Ask: "Is there a completely different way to do this?"

**Example from Phase 2:**
- Tried 8 different expression syntaxes in HTTP Request node
- All failed
- User frustration mounting
- **Switch:** Replaced HTTP Request with Code node
- **Result:** Worked immediately

**Lesson:** Don't polish a broken tool. Switch tools.

---

### 8. Document As You Go, Not After

**Problem:** Trying to remember what you changed after the fact.

**Better:**
- Keep a scratch file open during development
- Log each change as you make it
- Note why you made the change
- Capture error messages verbatim
- Record what you tried that didn't work

**Example Scratch Log:**
```
2026-01-23 14:30 - Changed ba_content_draft_url to ba_sequence_google_doc_url
  Why: Field not found error, checked HubSpot properties, found correct name

2026-01-23 14:45 - Replaced HTTP Request with Code node for Create Sequence
  Why: Expression evaluation failed 8 times, HTTP Request node can't handle complex JSON

2026-01-23 15:10 - Fixed action type mapping
  Why: Database constraint violation, queried DB to find allowed values
  linkedin_profile_visit → linkedin_visit
  linkedin_connection_request → linkedin_connect
```

**Then:** Convert scratch log to completion report at end.

---

## n8n-Specific Patterns

### 9. Code Node Patterns

**Pattern: Accessing Previous Node Data**
```javascript
// Single item from previous node
const data = $('Node Name').item.json;

// First item from node that outputs multiple
const first = $('Node Name').first().json;

// All items from previous node
const allItems = $('Node Name').all();

// Specific item by index
const third = $('Node Name').itemMatching(2).json; // 0-indexed
```

**Pattern: HTTP Request in Code Node**
```javascript
const response = await this.helpers.httpRequest({
  method: 'POST',
  url: 'https://api.example.com/endpoint',
  headers: {
    'Authorization': 'Bearer TOKEN',
    'Content-Type': 'application/json'
  },
  body: {
    field1: 'value1',
    field2: $json.dynamicValue
  },
  json: true  // Auto-parses response as JSON
});

return [{ json: response }];
```

**Pattern: Outputting Multiple Items**
```javascript
// Parse something and output N results
const results = [];

for (const item of items) {
  results.push({
    json: {
      field1: item.value,
      field2: processedData
    }
  });
}

return results;
```

---

### 10. Debugging n8n Workflows

**Enable Node Execution View:**
- Click on node after execution
- View "Output" tab
- See exact JSON that node produced

**Console Logging in Code Nodes:**
```javascript
console.log('Debug:', JSON.stringify(data, null, 2));
// View in n8n execution logs
```

**Test Individual Nodes:**
- Use "Test" button on nodes
- Don't run entire workflow every time
- Faster iteration

**Check Previous Node Data:**
```javascript
// In Code node, check what previous node actually output:
const prev = $('Previous Node').item.json;
console.log('Keys:', Object.keys(prev));
console.log('Full data:', JSON.stringify(prev, null, 2));
return [{ json: prev }];
```

---

## When Things Go Wrong

### Problem: "Cannot read property of undefined"

**Likely Causes:**
1. Previous node didn't execute (check connections)
2. Previous node output different structure than expected
3. Typo in property name

**Debug:**
```javascript
const data = $('Previous Node').item.json;
console.log('Available keys:', Object.keys(data));
console.log('Full structure:', JSON.stringify(data, null, 2));
```

---

### Problem: Node Executes Multiple Times When It Shouldn't

**Likely Cause:** Previous node outputs multiple items.

**Fix:** Reorder nodes so single-execution node runs before multi-output node.

**Or:** Use `.first()` to grab only first item:
```javascript
const singleRecord = $('Previous Node').first().json;
```

---

### Problem: Database Constraint Violation

**Likely Cause:** Application code uses values not allowed by database.

**Fix:**
1. Query database constraints
2. Update application code to match
3. Never update database to match code (database is source of truth)

---

## Anti-Patterns to Avoid

### ❌ Don't: Generate workflow JSON from scratch
✅ Do: Clone from a known-good workflow that has examples of the nodes you need

**This is critical enough to have its own section.** See `n8n-workflow-development.md` → "⛔ CRITICAL: Workflow Creation Anti-Pattern" for the full "Golden Template Approach" - the n8n API accepts malformed JSON that the UI cannot render, causing silent failures.

### ❌ Don't: Build entire workflow before testing
✅ Do: Test every 2-3 nodes

### ❌ Don't: Use HTTP Request nodes for complex JSON
✅ Do: Use Code nodes with this.helpers.httpRequest

### ❌ Don't: Guess at property names
✅ Do: Query API to verify exact names

### ❌ Don't: Ignore database constraints
✅ Do: Check constraints before writing code

### ❌ Don't: Keep trying variations of a broken approach
✅ Do: Switch tools/methods when approach is fundamentally broken

### ❌ Don't: Skip backups before major changes
✅ Do: Always backup workflows before rebuilds

### ❌ Don't: Match arrays by index across different data sources
✅ Do: Match by unique key (URL, ID, etc.) - arrays from different sources may have different order

---

## Checklist for New n8n Workflows

**Before Building:**
- [ ] **Find a template workflow** - Clone from a known-good workflow with similar nodes (see `n8n-workflow-development.md` → Golden Template Approach)
- [ ] Query external APIs to verify property names
- [ ] Check database constraints for allowed values
- [ ] Create backup of any workflow being modified
- [ ] Map out node flow on paper (1-to-1 vs 1-to-N vs N-to-N)

**During Development:**
- [ ] Test every 2-3 nodes with real data
- [ ] Use Code nodes for complex JSON operations
- [ ] Log changes and errors in scratch file
- [ ] Verify node execution order for multi-output scenarios
- [ ] When merging data from multiple sources, match by unique key (NOT by array index)

**After Building:**
- [ ] End-to-end test with multiple data variations
- [ ] Document what changed and why
- [ ] Update skills documentation with new learnings
- [ ] Create completion report

---

## Related Skills

- `n8n-workflow-development.md` - Comprehensive n8n workflow patterns
- `n8n-troubleshooting.md` - MCP connection issues and debugging
- `n8n-api-mcp-server.md` - n8n REST API and MCP tool reference

---

**Remember:** Every painful debugging session is a learning opportunity. Capture it. Share it. Don't repeat it.
