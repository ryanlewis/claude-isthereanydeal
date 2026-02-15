# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code plugin that provides game deal lookups via the [IsThereAnyDeal](https://isthereanydeal.com/) API. It's a skill-only plugin with no hooks, commands, or MCP servers.

## Architecture

```
.claude-plugin/plugin.json    → Plugin manifest (name, version, metadata)
skills/isthereanydeal/
  SKILL.md                    → Skill definition (trigger conditions, workflow, formatting rules)
  references/api-reference.md → Full ITAD API reference (endpoints, params, response schemas)
```

The skill is triggered by natural language queries about game prices/deals. It reads an API key from `.claude/isthereanydeal.local.json` (project or global) or the `ITAD_API_KEY` env var, then calls the ITAD REST API to search games, fetch prices, and present results.

## Key Design Decisions

- **Settings use JSON** (`.claude/isthereanydeal.local.json`), not YAML frontmatter in `.local.md` files
- **Three-tier config resolution**: project settings file → global settings file → environment variable
- **API key resolution is in SKILL.md**, not in a hook or external script — Claude reads the file and follows the instructions
- **POST endpoints accept batches** of up to 200 game UUIDs — the skill is designed to handle multi-game queries in a single request
