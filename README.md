# game-deals

A Claude Code plugin for finding game deals and price history using the [IsThereAnyDeal](https://isthereanydeal.com/) API.

## Features

- Search for games by name
- Check current prices across 70+ stores
- View historical low prices
- Compare deals across stores with discount percentages

## Prerequisites

An IsThereAnyDeal API key (free). Register at https://isthereanydeal.com/dev/app/.

## Installation

```bash
claude --plugin-dir /path/to/claude-isthereanydeal
```

## Configuration

Create `.claude/game-deals.local.md` in your project:

```markdown
---
api_key: "your-api-key-here"
country: "US"
---
```

Or set the `ITAD_API_KEY` environment variable:

```bash
export ITAD_API_KEY="your-api-key-here"
```

The settings file takes precedence over the environment variable.

### Settings

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `api_key` | Yes | — | Your ITAD API key |
| `country` | No | `US` | 2-char ISO country code for pricing |

## Usage

Ask naturally about game prices:

- "How much does Elden Ring cost?"
- "Is Baldur's Gate 3 on sale?"
- "What's the cheapest price for Hades?"
- "Price history for Cyberpunk 2077"
- "Find deals on indie roguelikes"

## License

MIT
