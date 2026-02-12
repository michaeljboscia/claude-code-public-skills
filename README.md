# Claude Code Public Skills

> Curated skills and foundational documentation for AI-assisted development with Claude Code CLI

**Status:** 🚧 In Development (Private Testing Phase)

This repository contains battle-tested patterns, protocols, and skills developed through real-world AI-assisted development projects. These skills make Claude Code smarter, faster, and more reliable at complex tasks.

---

## Quick Context (For AI)

**What This Is:**
- Collection of `.md` skill files that enhance Claude Code's capabilities
- Foundational documentation showing architectural patterns for AI-assisted development
- Real-world protocols refined through iteration (not theoretical best practices)

**How To Use:**
1. Clone this repo to your local machine
2. Copy skills to `~/.claude/skills/`
3. Reference them in your `~/.claude/CLAUDE.md` or project-specific docs
4. Claude Code will use these patterns automatically

**Auto-Generated:** This repo is automatically maintained via sync script. Skills marked `shareable: true` are published here with sensitive information sanitized.

---

## Table of Contents

- [Available Skills](#available-skills)
- [Foundational Documentation](#foundational-documentation)
- [Installation](#installation)
- [Skill Categories](#skill-categories)
- [Contributing](#contributing)
- [Philosophy](#philosophy)

---

## Available Skills

**Total Skills:** *Auto-generated catalog will appear here*

<!-- BEGIN AUTO-GENERATED CATALOG -->
### Available Skills (9 total)

#### [n8n-api-mcp-server](skills/n8n-api-mcp-server.md)

#### [n8n-node-implementation](skills/n8n-node-implementation.md)

#### [n8n-platform-reference](skills/n8n-platform-reference.md)

#### [n8n-troubleshooting](skills/n8n-troubleshooting.md)

#### [n8n-workflow-backup](skills/n8n-workflow-backup.md)

#### [n8n-workflow-development-lessons](skills/n8n-workflow-development-lessons.md)

#### [n8n-workflow-development](skills/n8n-workflow-development.md)

#### [pptx-toolkit](skills/pptx-toolkit.md)

#### [supabase-api](skills/supabase-api.md)

<!-- END AUTO-GENERATED CATALOG -->

---

## Foundational Documentation

**Core Philosophy:**
- How to structure instructions for AI agents
- Strategic push-back framework (questioning problems before building solutions)
- Documentation standards optimized for AI comprehension

**Protocols:**
- Backup protocols before database modifications
- File organization taxonomy
- Git workflow for bug investigation

---

## Installation

### Option 1: Copy Individual Skills

```bash
# Copy a specific skill
curl -o ~/.claude/skills/supabase-schema.md \
  https://raw.githubusercontent.com/michaeljboscia/claude-code-public-skills/main/skills/supabase-schema.md
```

### Option 2: Clone Entire Repository

```bash
# Clone the repo
git clone https://github.com/michaeljboscia/claude-code-public-skills.git

# Symlink skills directory (keeps skills up-to-date)
ln -s ~/claude-code-public-skills/skills ~/.claude/skills/public

# Reference in your CLAUDE.md
echo "- Public Skills: see ~/.claude/skills/public/" >> ~/.claude/CLAUDE.md
```

### Option 3: Use as Git Submodule

```bash
# Add as submodule to your own config repo
cd ~/.claude
git submodule add https://github.com/michaeljboscia/claude-code-public-skills.git public-skills
```

---

## Skill Categories

### Database & Infrastructure
- **Supabase Management** - PostgreSQL patterns, backup protocols, edge function deployment
- **n8n Workflows** - Automation orchestration, webhook patterns

### Data Collection
- **Pain Sensor Systems** - Multi-source data collection and aggregation
- **API Integration** - DataForSEO, Wappalyzer, Adyntel, and more

### Development Workflows
- **Git Workflows** - Bug investigation, structured PR formats
- **Documentation Standards** - README structure for AI agents

### AI-Assisted Development
- **Strategic Push-Back** - Question problems before building solutions
- **Explain Before Executing** - Protocol for transparent changes

---

## Contributing

**This repo is auto-maintained** from a private development environment. Skills are published here after being battle-tested in production.

**Want to contribute improvements?**
1. Open an issue describing the enhancement
2. If it's a pattern you've developed, share it!
3. Maintainer will review and potentially incorporate

---

## Philosophy

### Why These Skills Exist

**The Problem:** Claude Code is powerful, but generic. Every project has patterns, anti-patterns, and lessons learned that should inform future work.

**The Solution:** Capture these patterns as skills - structured markdown files that give Claude Code domain-specific expertise.

### Key Principles

1. **Comprehensive Over Terse** - AI needs context to execute correctly. Better to over-explain than assume knowledge.

2. **Real-World Battle-Tested** - These aren't theoretical best practices. These are patterns refined through actual failures and iterations.

3. **Shareable But Not Dogmatic** - Use what works for you. Adapt to your context. Ignore what doesn't fit.

4. **Living Documentation** - Skills evolve as we learn. Anti-patterns discovered today become protocols tomorrow.

---

## Skill File Format

All skills follow a consistent format:

```markdown
---
shareable: true
title: Skill Name
category: database|infrastructure|development|ai-assisted
requires:
  - Dependencies or tools needed
license: MIT
version: 1.0.0
---

# Skill Name

## Quick Context (For AI)
[2-3 sentences explaining what this skill does and when to use it]

## When to Use This

[Clear triggers for when this skill applies]

## Step-by-Step Protocol

[Numbered steps with examples]

## Anti-Patterns (NEVER Do This)

[Common mistakes to avoid]

## Examples

[Real-world examples with expected output]
```

---

## License

MIT License - See [LICENSE](LICENSE) for details

---

## About

**Maintained by:** Mike Boscia ([@michaeljboscia](https://github.com/michaeljboscia))

**Built with:** Claude Code CLI, battle-tested in production GTM automation infrastructure

**Questions?** Open an issue or DM on Twitter/LinkedIn

---

**Last Updated:** 2026-02-11 21:51:21 EST
