---
name: Find Game Deals
description: This skill should be used when the user asks to "find game deals", "check game prices", "how much does [game] cost", "is [game] on sale", "price history for [game]", "cheapest price for [game]", "where to buy [game]", "best deal on [game]", or mentions IsThereAnyDeal, ITAD, or game pricing. Provides the ability to look up current game prices across stores and historical price data using the IsThereAnyDeal API.
---

# Find Game Deals

Look up current game prices across stores and check historical price data using the IsThereAnyDeal (ITAD) API.

## API Key Setup

Before making any API calls, obtain the API key:

1. **Check settings file first:** Read `.claude/game-deals.local.md` in the project root. If it exists, parse the YAML frontmatter to extract the `api_key` field.
2. **Fall back to environment variable:** If no settings file exists or `api_key` is empty, use the `ITAD_API_KEY` environment variable.
3. **Prompt if missing:** If neither source provides a key, inform the user they need an API key from https://isthereanydeal.com/dev/app/ and can configure it by either:
   - Creating `.claude/game-deals.local.md` with `api_key` in the frontmatter
   - Setting the `ITAD_API_KEY` environment variable

Read the API key with:
```bash
# Try settings file first
if [[ -f ".claude/game-deals.local.md" ]]; then
  ITAD_API_KEY=$(sed -n '/^---$/,/^---$/{ /^---$/d; p; }' ".claude/game-deals.local.md" | grep '^api_key:' | sed 's/api_key: *//' | sed 's/^"\(.*\)"$/\1/' | tr -d ' ')
fi

# Fall back to env var (already set if exported)
echo "${ITAD_API_KEY:-}"
```

## Country/Region

Determine the country code for pricing:

1. **Check settings file:** Look for a `country` field in `.claude/game-deals.local.md` frontmatter.
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
curl -s -X POST "https://api.isthereanydeal.com/games/prices/v3?country=$COUNTRY&capacity=10&key=$ITAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["GAME_UUID"]'
```

Use `capacity=10` to limit to the top 10 deals per game. Set `deals=true` as a query parameter if the user only wants to see discounted prices.

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
- Current best price, store name, and discount percentage
- Historical low price, the store it was at, and when it occurred
- Whether the current price is at or near the historical low

**Deal table** (if multiple stores have the game):

| Store | Price | Discount | DRM | Link |
|-------|-------|----------|-----|------|
| Steam | $39.99 | -33% | Steam | [Buy](url) |
| GOG | $35.99 | -40% | DRM-Free | [Buy](url) |

Highlight the best deal. Note any deals at historical low prices (flag "H" in the response). Include deal expiry dates when available.

## Multiple Games

To look up multiple games at once, search for each game individually to get their UUIDs, then batch the UUIDs into a single prices/overview call (up to 200 per request). Present results grouped by game.

## API Reference

For complete endpoint documentation including all parameters, response schemas, and curl examples, consult **`references/api-reference.md`**.

## Settings File Template

The settings file at `.claude/game-deals.local.md` supports these fields:

```markdown
---
api_key: "your-api-key-here"
country: "US"
---
```

- `api_key` — IsThereAnyDeal API key (get one at https://isthereanydeal.com/dev/app/)
- `country` — 2-character ISO country code for pricing (default: US)

## Important Notes

- The API has no hard rate limit but has heuristic abuse detection. Avoid unnecessary repeated calls.
- Affiliate links in store URLs must not be modified (per ITAD Terms of Service).
- POST endpoints accept JSON arrays of game UUIDs. Always set `Content-Type: application/json`.
- Game IDs are UUIDs (e.g., `018d937f-012f-73b8-ab2c-898516969e6a`), not plain strings.
