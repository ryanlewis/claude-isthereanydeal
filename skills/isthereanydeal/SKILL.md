---
name: IsThereAnyDeal
description: This skill should be used when the user asks to "find game deals", "check game prices", "how much does [game] cost", "is [game] on sale", "price history for [game]", "cheapest price for [game]", "where to buy [game]", "best deal on [game]", "compare game prices", or mentions IsThereAnyDeal, ITAD, or game pricing. Uses the IsThereAnyDeal API to look up current game prices across stores and retrieve historical price data.
version: 0.1.0
---

# IsThereAnyDeal

Look up current game prices across stores and check historical price data using the IsThereAnyDeal (ITAD) API.

## API Key Setup

Before making any API calls, obtain the API key:

1. **Check settings file first:** Read `.claude/isthereanydeal.local.json` in the project root (or `~/.claude/isthereanydeal.local.json` for global config). If it exists, extract the `api_key` field from the JSON.
2. **Fall back to environment variable:** If no settings file exists or `api_key` is empty, use the `ITAD_API_KEY` environment variable.
3. **Prompt if missing:** If neither source provides a key, inform the user they need to create an app at https://isthereanydeal.com/apps/ to get an API key, then configure it by either:
   - Creating `.claude/isthereanydeal.local.json` with an `api_key` field
   - Setting the `ITAD_API_KEY` environment variable

To obtain the key: read `.claude/isthereanydeal.local.json` (or `~/.claude/isthereanydeal.local.json`) with the Read tool and parse the JSON to extract the `api_key` value. If the file does not exist or has no `api_key` field, check the `ITAD_API_KEY` environment variable.
## Country/Region

Determine the country code for pricing:

1. **Check settings file:** Look for a `country` field in `.claude/isthereanydeal.local.json`.
2. **Infer from context:** If the user's locale, timezone, or other context clues indicate a region, use the corresponding 2-character ISO country code.
3. **Default to US:** If no country can be determined, default to `US`.

## Core Workflow

### Step 1: Search for the Game

Resolve the game name to an ITAD game ID using the search endpoint:

```bash
curl -s "https://api.isthereanydeal.com/games/search/v1?title=GAME_NAME&results=5&key=$ITAD_API_KEY"
```

Parse the response to find the best match. If multiple results are returned, select the one whose `title` most closely matches the user's query, preferring `type: "game"` over DLC or packages. If the match is ambiguous, present the top results and ask the user to clarify.

Extract the `id` field (UUID) from the matched game — this is needed for all subsequent API calls.

### Step 2: Get Current Prices

Fetch current prices across all stores:

```bash
curl -s -X POST "https://api.isthereanydeal.com/games/prices/v3?country=$COUNTRY&capacity=0&key=$ITAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["GAME_UUID"]'
```

Use `capacity=0` to return all available prices across all stores. Optionally add `deals=true` as a query parameter if the user only wants to see discounted prices.

### Step 3: Get Price Overview and History

Fetch the historical low and current best deal summary:

```bash
curl -s -X POST "https://api.isthereanydeal.com/games/overview/v2?country=$COUNTRY&key=$ITAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["GAME_UUID"]'
```

This returns the current best price and the all-time historical low in a single call.

### Step 4: Format and Present Results

Present the results in a clear, readable format. Include:

**Price summary:**
- Game title
- Current Steam price (always include this prominently, even if it's not the best deal)
- Current best price, store name, and discount percentage
- Historical low price, the store it was at, and when it occurred
- Whether the current price is at or near the historical low

**Deal table** (all stores that have the game):

| Store | Price | Discount | DRM | Link |
|-------|-------|----------|-----|------|
| Steam | $59.99 | — | Steam | [Buy](url) |
| GOG | $35.99 | -40% | DRM-Free | [Buy](url) |
| Humble | $39.99 | -33% | Steam | [Buy](url) |

Always list the Steam price first in the table, then sort remaining stores by price (lowest first). Highlight the best overall deal. Note any deals at historical low prices (flag "H" in the response). Include deal expiry dates when available. If Steam is not among the results, note that explicitly.

## Multiple Games

To look up multiple games at once, search for each game individually to get their UUIDs, then batch the UUIDs into a single prices/overview call (up to 200 per request). Present results grouped by game.

## API Reference

For complete endpoint documentation including all parameters, response schemas, and curl examples, consult **`references/api-reference.md`**.

## Settings File Template

The settings file at `.claude/isthereanydeal.local.json` supports these fields:

```json
{
  "api_key": "your-api-key-here",
  "country": "US"
}
```

- `api_key` — IsThereAnyDeal API key (create an app at https://isthereanydeal.com/apps/ to get one)
- `country` — 2-character ISO country code for pricing (default: US)

## Important Notes

- The API has no hard rate limit but has heuristic abuse detection. Avoid unnecessary repeated calls.
- Affiliate links in store URLs must not be modified (per ITAD Terms of Service).
- POST endpoints accept JSON arrays of game UUIDs. Always set `Content-Type: application/json`.
- Game IDs are UUIDs (e.g., `018d937f-012f-73b8-ab2c-898516969e6a`), not plain strings.
