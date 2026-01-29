# Setup Guide: Dual-Repo Auto-Sync System

Complete setup instructions for the automated Claude Code skill publishing system.

---

## System Overview

**What This System Does:**
1. Watches `~/.claude/` for changes
2. Automatically syncs to **private repo** (full versions)
3. Automatically syncs to **public repo** (shareable skills only, sanitized)
4. Generates catalog of shared skills
5. Runs in background, survives reboots

**Zero Manual Steps:** Edit skills, mark `shareable: true`, system handles the rest.

---

## Prerequisites

### 1. Install fswatch

```bash
brew install fswatch
```

### 2. Verify Git Authentication

```bash
# Test private repo access
cd ~/claude-code-config && git pull

# Test public repo access
cd ~/claude-code-public-skills && git pull
```

Both should work without password prompts.

---

## Setup Steps

### Step 1: Enhanced Sync Script

The enhanced sync script is already installed at `~/scripts/sync-claude-config.sh`.

**Test it manually first:**

```bash
~/scripts/sync-claude-config.sh
```

You should see:
- Phase 1: Private repo sync (all files)
- Phase 2: Public repo sync (shareable skills only)
- Summary with links to both repos

### Step 2: File Watcher Script

Already created at `~/scripts/watch-claude-config.sh`.

**Test it manually:**

```bash
~/scripts/watch-claude-config.sh &
```

This will start watching `~/.claude/` for changes. Test by editing a file:

```bash
echo "# Test" >> ~/.claude/test.md
```

Watch the log:

```bash
tail -f ~/Library/Logs/claude-config-watcher.log
```

You should see "Change detected, syncing..." messages.

Kill the test watcher:

```bash
pkill -f watch-claude-config.sh
```

### Step 3: Load Launchd Daemon (Always-On)

The launchd plist is at `~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist`.

**Load the daemon:**

```bash
launchctl load ~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist
```

**Verify it's running:**

```bash
launchctl list | grep claude-config-watcher
```

You should see output like:
```
12345   0   com.binaryanvil.claude-config-watcher
```

**Check logs:**

```bash
tail -f ~/Library/Logs/claude-config-watcher.log
```

---

## Usage

### Marking Skills as Shareable

Add frontmatter to any skill file:

```markdown
---
shareable: true
title: Your Skill Name
category: database|infrastructure|development|ai-assisted
requires:
  - List of dependencies
license: MIT
version: 1.0.0
---

# Your Skill Name

[Rest of skill content...]
```

**That's it!** The file watcher will detect the change and auto-sync to both repos within 30 seconds.

### Checking Sync Status

**View watcher log:**
```bash
tail -f ~/Library/Logs/claude-config-watcher.log
```

**Check last sync time:**
```bash
cd ~/claude-code-config && git log -1 --format="%ar: %s"
cd ~/claude-code-public-skills && git log -1 --format="%ar: %s"
```

**Force manual sync:**
```bash
~/scripts/sync-claude-config.sh
```

---

## Sanitization Rules

The system automatically strips sensitive information from public versions:

| Original | Replaced With |
|----------|---------------|
| `192.168.x.x` | `YOUR-INTERNAL-IP` |
| `homebox`, `homebox2` | `your-n8n-server` |
| `sk-ant-api03-...` | `your-anthropic-api-key` |
| `sk-proj-...` | `your-openai-api-key` |
| `ghp_...` | `your-github-pat` |
| `pat-na2-...` | `your-hubspot-pat` |
| `C1sc0123` | `your-password` |
| `wojvoruhqpmtjtuyrjpf` | `your-supabase-project-ref` |
| `webhook/abc-123` | `webhook/your-webhook-id` |
| `Binary Anvil` | `YourCompany` |
| `binaryanvil.com` | `yourcompany.com` |
| `mboscia` | `yourusername` |

**Review public versions before making repo public:**

```bash
cd ~/claude-code-public-skills/skills
cat some-skill.md | grep -E "192\.|homebox|sk-|ghp_|C1sc0|Binary"
```

If you find any unsanitized sensitive info, add regex to `sanitize_file()` function in sync script.

---

## Maintenance

### Stop the Watcher

```bash
launchctl unload ~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist
```

### Restart the Watcher

```bash
launchctl unload ~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist
launchctl load ~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist
```

### Update Sync Script

If you modify `~/scripts/sync-claude-config.sh`, restart the watcher:

```bash
launchctl kickstart -k gui/$(id -u)/com.binaryanvil.claude-config-watcher
```

### View Error Logs

```bash
tail -f ~/Library/Logs/claude-config-watcher.error.log
```

---

## Troubleshooting

### Watcher Not Starting

**Check if fswatch is installed:**
```bash
which fswatch
```

If not found:
```bash
brew install fswatch
```

**Check launchd status:**
```bash
launchctl list | grep claude-config-watcher
```

**Check error log:**
```bash
cat ~/Library/Logs/claude-config-watcher.error.log
```

### Sync Not Happening

**Check if watcher is running:**
```bash
ps aux | grep watch-claude-config
```

**Manually trigger sync to test:**
```bash
~/scripts/sync-claude-config.sh
```

**Check for git authentication issues:**
```bash
cd ~/claude-code-config && git pull
cd ~/claude-code-public-skills && git pull
```

### No Skills Appearing in Public Repo

**Check if skills have frontmatter:**
```bash
grep -l "^shareable: true" ~/.claude/skills/*.md
```

If none found, add frontmatter to skills you want to share.

**Check sync log:**
```bash
grep "Found shareable" ~/Library/Logs/claude-config-watcher.log | tail
```

---

## Making Public Repo Actually Public

**Current status:** Both repos are private for testing.

**When ready to go public:**

1. **Review all published skills:**
   ```bash
   cd ~/claude-code-public-skills/skills
   ls -la
   cat *.md | grep -E "192\.|homebox|sk-|Binary Anvil|C1sc0"
   ```

2. **Make repo public on GitHub:**
   - Go to https://github.com/michaeljboscia/claude-code-public-skills/settings
   - Scroll to "Danger Zone"
   - Click "Change visibility" → "Make public"

3. **Announce on social media:**
   - Twitter thread explaining the philosophy
   - LinkedIn post for professional audience
   - Include example skills and use cases

---

## Architecture Diagram

```
~/.claude/skills/supabase-schema.md
  └─ frontmatter: shareable: true
     └─ fswatch detects file change (within 30s)
        └─ ~/scripts/watch-claude-config.sh triggers
           └─ ~/scripts/sync-claude-config.sh runs
              ↓
        ┌─────┴─────┐
        ↓           ↓
  PRIVATE REPO    PUBLIC REPO
  (full version)  (sanitized)
        ↓           ↓
  Git commit      Git commit
  Git push        Git push
        ↓           ↓
    GitHub        GitHub
   (private)    (private for now)
```

---

## Files Involved

| File | Purpose |
|------|---------|
| `~/scripts/sync-claude-config.sh` | Enhanced dual-repo sync script |
| `~/scripts/watch-claude-config.sh` | File watcher wrapper |
| `~/Library/LaunchAgents/com.binaryanvil.claude-config-watcher.plist` | Launchd daemon config |
| `~/Library/Logs/claude-config-watcher.log` | Watcher activity log |
| `~/Library/Logs/claude-config-watcher.error.log` | Error log |

---

## Next Steps

1. ✅ System is installed and ready
2. ⏳ Test by marking a skill `shareable: true`
3. ⏳ Verify it appears in public repo (sanitized)
4. ⏳ Load launchd daemon for always-on automation
5. ⏳ Review public repo before making public
6. ⏳ Flip repo to public visibility
7. ⏳ Share with community

---

**Questions or issues?** Check logs first, then review troubleshooting section above.
