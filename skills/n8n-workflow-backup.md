---
name: n8n-workflow-backup
description: Fast automated backup system for n8n Cloud workflows before making changes
shareable: true
---

# n8n Cloud Workflow Backup System

## Purpose

Fast, automated backups of n8n Cloud workflows before making changes. Prevents catastrophic loss of business automation workflows.

**Key Stats:**
- Backup time: 1-2 seconds (single workflow), ~30 seconds (all 82 workflows)
- Context usage: < 1k tokens (vs 120k tokens with sub-agent method)
- 120x more efficient than previous backup method

---

## Quick Start

**Script Location:** `~/scripts/backup-n8n-workflows.sh`

**Three Usage Modes:**

```bash
# 1. Single workflow (MOST COMMON - use before editing)
~/scripts/backup-n8n-workflows.sh "Pain Sensor Data Collection"

# 2. Multiple specific workflows
~/scripts/backup-n8n-workflows.sh "Workflow 1" "Workflow 2" "Workflow 3"

# 3. Full backup (all 82 workflows - safety net for major changes)
~/scripts/backup-n8n-workflows.sh --full
```

**Typical Workflow:**
```bash
# BEFORE editing workflow
~/scripts/backup-n8n-workflows.sh "Pain Sensor #1"

# Make changes in n8n Cloud UI or via API
# ...

# If something breaks, restore from backup
```

---

## What Gets Backed Up

**Complete workflow data:**
- All nodes and their configurations
- All connections between nodes
- Workflow settings (timezone, execution order, error handling)
- Credentials references (not actual credentials)
- Workflow metadata (name, tags, active status)

**JSON format** - Full n8n workflow export format, ready to re-import

---

## Where Backups Go

**Backup Directory:**
```
~/your-backup-directory/
```
*(Configure BACKUP_DIR in the script to your preferred location)*

**File Naming Convention:**
```
{workflow-name}-backup-{YYYY-MM-DD}.json
```

**Examples:**
- `Pain_Sensor_Data_Collection-backup-2026-01-24.json`
- `SEOptimer_Enhanced_Extraction-backup-2026-01-24.json`
- `Bulk_PSI_DESKTOP_Scanner-backup-2026-01-24.json`

**Notes:**
- Special characters in workflow names are replaced with underscores
- One backup per workflow per day (won't duplicate if run multiple times same day)
- Backups are valid JSON files - can be inspected with `jq` or text editor

---

## Retention Policy

**Auto-Cleanup:** Backups older than **7 days** are automatically deleted

**Rationale:**
- Keeps recent history without accumulating massive storage
- 7 days = enough time to notice and recover from issues
- Old backups unlikely to be useful (workflows evolve quickly)

**Manual Cleanup (if needed):**
```bash
# See all backups
ls -lh ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/

# Delete specific old backup
rm ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/Old_Workflow-backup-2025-12-01.json
```

---

## How to Restore a Workflow

### Option 1: n8n Cloud UI (Easiest)

1. Go to https://your-n8n-cloud.app.n8n.cloud
2. Click "+" (Add Workflow)
3. Click "Import from File"
4. Select backup JSON file from backups directory
5. Workflow is restored with all nodes and connections

### Option 2: n8n API (Programmatic)

```bash
# Read backup file and import
BACKUP_FILE="~/your-backup-directory/Workflow_Name-backup-2026-01-24.json"

curl -X POST "https://your-n8n-cloud.app.n8n.cloud/api/v1/workflows" \
  -H "X-N8N-API-KEY: YOUR-API-KEY" \
  -H "Content-Type: application/json" \
  -d @"$BACKUP_FILE"
```

### Option 3: Use MCP Tools

```bash
# Read backup file
cat ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/Workflow_Name-backup-2026-01-24.json

# Use mcp__n8n-cloud__n8n_create_workflow with the backup JSON
```

**Important Notes:**
- Restoring creates a NEW workflow (doesn't overwrite existing)
- You may need to reactivate the workflow after restore
- Credentials need to be reconnected (credential IDs may differ)
- Webhook URLs will be regenerated (update any external integrations)

---

## Troubleshooting

### Error: "Failed to fetch workflow list from n8n Cloud"

**Cause:** API connection issue or invalid API key

**Fix:**
1. Verify n8n Cloud is accessible: `curl -I https://your-n8n-cloud.app.n8n.cloud`
2. Check API key in script matches `~/.claude.json` (line 401)
3. Verify MCP server is configured: `claude mcp get n8n-cloud`

### Error: "Workflow not found: [name]"

**Cause:** Workflow name doesn't match exactly (case-sensitive)

**Fix:**
1. List all workflows to see exact names:
   ```bash
   curl -s "https://your-n8n-cloud.app.n8n.cloud/api/v1/workflows" \
     -H "X-N8N-API-KEY: ..." | jq -r '.data[].name'
   ```
2. Copy exact workflow name (including spaces, capitalization)
3. Use quotes around workflow name: `"Pain Sensor #1"`

### Error: "Already backed up today"

**Cause:** Backup already exists for today's date

**Not an error!** Script skips duplicate backups to save time. If you need to force a new backup:
```bash
# Delete today's backup first
rm ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/Workflow_Name-backup-$(TZ='America/New_York' date +%Y-%m-%d).json

# Re-run backup
~/scripts/backup-n8n-workflows.sh "Workflow Name"
```

### Backup takes longer than 30 seconds

**Normal for --full mode** with 82 workflows. Times vary based on:
- Workflow complexity (number of nodes)
- Network latency to n8n Cloud
- Parallel job count (currently 10)

**To speed up:**
- Only backup workflows you're actually changing (selective mode)
- Use --full sparingly (once a week for safety)

### Invalid JSON error

**Cause:** n8n API returned error or incomplete data

**Fix:**
1. Check workflow exists in n8n Cloud UI
2. Verify workflow isn't corrupted
3. Try backing up a different workflow first
4. Check n8n Cloud status page

---

## When to Use

### ALWAYS Use Before:
- ✅ Modifying workflow nodes or connections
- ✅ Changing workflow settings
- ✅ Deleting workflows
- ✅ Testing complex workflow changes
- ✅ Updating credentials in workflows
- ✅ Experimenting with new node types

### Consider Using For:
- 🔶 Weekly full backup as safety net (`--full`)
- 🔶 Before bulk workflow operations
- 🔶 Before major n8n Cloud platform updates

### Don't Need For:
- ❌ Just viewing workflows (read-only)
- ❌ Checking workflow execution logs
- ❌ Reading workflow documentation

---

## Performance Comparison

**Old Method (Sub-Agent with MCP tools):**
- Time: 2-3 minutes
- Context usage: ~120k tokens
- Manual process

**New Method (This Script):**
- Time: 1-2 seconds (single), ~30 seconds (full)
- Context usage: < 1k tokens
- Automated, one command

**Improvement:** 6x faster, 120x less context

---

## Configuration

**Script Location:** `~/scripts/backup-n8n-workflows.sh`

**Configurable Variables (edit script if needed):**
```bash
N8N_API_URL="https://your-n8n-cloud.app.n8n.cloud"
BACKUP_DIR="$HOME/My Drive/GTM Machine content/Outbound content pipeline/backups"
RETENTION_DAYS=7        # How many days to keep backups
PARALLEL_JOBS=10        # How many workflows to backup in parallel (--full mode)
```

**API Key:** Stored in script (synced from `~/.claude.json`)

---

## Related Documentation

- **n8n Cloud Instance:** https://your-n8n-cloud.app.n8n.cloud
- **n8n API Docs:** https://docs.n8n.io/api/
- **MCP Configuration:** `~/.claude/data-locations.md` (n8n section)
- **Workflow Development:** `~/.claude/skills/n8n-workflow-development-lessons.md`

---

## Quick Cheat Sheet

```bash
# Before editing ANY workflow
~/scripts/backup-n8n-workflows.sh "Workflow Name"

# Weekly full backup (Sunday night)
~/scripts/backup-n8n-workflows.sh --full

# Check backup worked
ls -lht ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/ | head

# View backup contents
cat ~/My\ Drive/GTM\ Machine\ content/Outbound\ content\ pipeline/backups/Workflow_Name-backup-2026-01-24.json | jq .

# Restore from backup
# Use n8n Cloud UI: Import from File
```

**Remember:** 1-2 seconds to backup. Hours (or impossible) to recover without backup. Always backup first!
