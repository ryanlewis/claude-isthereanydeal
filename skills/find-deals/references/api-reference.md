# IsThereAnyDeal API Reference

Base URL: `https://api.isthereanydeal.com`

Authentication: API key passed as query parameter `key`. Example: `?key=YOUR_API_KEY`

## Endpoints

### GET /games/search/v1

Search for games by title. Use this first to resolve a game name to an ITAD game ID (UUID).

**Parameters:**
- `title` (required, string) — game name to search for
- `results` (optional, integer 1-100, default 20) — max results
- `key` (required) — API key

**Example request:**
```bash
curl -s "https://api.isthereanydeal.com/games/search/v1?title=Elden+Ring&results=5&key=$ITAD_API_KEY"
```

**Response:** JSON array of game objects:
```json
[
  {
    "id": "018d937f-012f-73b8-ab2c-898516969e6a",
    "slug": "elden-ring",
    "title": "Elden Ring",
    "type": "game",
    "mature": false
  }
]
```

Fields:
- `id` — UUID, used as the game identifier for all other endpoints
- `slug` — URL-friendly name
- `title` — display name
- `type` — "game", "dlc", "package", "bundle"

---

### GET /games/lookup/v1

Find a single game by exact title or Steam appid. Not a full search — requires exact match.

**Parameters:**
- `title` (optional, string) — exact game title
- `appid` (optional, integer) — Steam application ID
- `key` (required) — API key

**Example request:**
```bash
curl -s "https://api.isthereanydeal.com/games/lookup/v1?title=Elden+Ring&key=$ITAD_API_KEY"
```

**Response:**
```json
{
  "found": true,
  "game": {
    "id": "018d937f-012f-73b8-ab2c-898516969e6a",
    "slug": "elden-ring",
    "title": "Elden Ring",
    "type": "game",
    "mature": false
  }
}
```

---

### POST /games/prices/v3

Get current prices across all stores for one or more games. This is the primary endpoint for finding deals.

**Query parameters:**
- `country` (optional, 2-char ISO code, default "US") — pricing region
- `deals` (optional, boolean) — if true, only return discounted prices
- `vouchers` (optional, boolean) — include voucher-eligible items
- `capacity` (optional, integer >= 0) — max prices per game (0 = unlimited)
- `shops` (optional, array of integers) — filter to specific shop IDs
- `key` (required) — API key

**Request body:** JSON array of 1-200 game UUIDs:
```json
["018d937f-012f-73b8-ab2c-898516969e6a"]
```

**Example request:**
```bash
curl -s -X POST "https://api.isthereanydeal.com/games/prices/v3?country=US&capacity=5&key=$ITAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["018d937f-012f-73b8-ab2c-898516969e6a"]'
```

**Response:** JSON array of per-game pricing data:
```json
[
  {
    "id": "018d937f-012f-73b8-ab2c-898516969e6a",
    "deals": [
      {
        "shop": { "id": 61, "name": "Steam" },
        "price": { "amount": 59.99, "amountInt": 5999, "currency": "USD" },
        "regular": { "amount": 59.99, "amountInt": 5999, "currency": "USD" },
        "cut": 0,
        "voucher": null,
        "flag": null,
        "drm": [{ "id": 1, "name": "Steam" }],
        "platforms": [{ "id": 1, "name": "Windows" }],
        "timestamp": "2024-01-15T12:00:00Z",
        "expiry": null,
        "url": "https://store.steampowered.com/app/1245620/"
      }
    ]
  }
]
```

Key response fields:
- `deals[].shop.name` — store name (Steam, GOG, Humble Store, etc.)
- `deals[].price.amount` — current price
- `deals[].regular.amount` — regular (non-sale) price
- `deals[].cut` — discount percentage (0-100)
- `deals[].url` — link to buy
- `deals[].drm[].name` — DRM type
- `deals[].platforms[].name` — supported platforms
- `deals[].flag` — "H" for historical low, "S" for store low, "N" for new

---

### POST /games/overview/v2

Get a summary of current best price and historical low for games. Lighter than /prices/v3 — use for quick overviews.

**Query parameters:**
- `country` (optional, 2-char ISO code, default "US") — pricing region
- `shops` (optional, array of integers) — filter to specific shop IDs
- `vouchers` (optional, boolean, default true) — include voucher prices
- `key` (required) — API key

**Request body:** JSON array of 1-200 game UUIDs:
```json
["018d937f-012f-73b8-ab2c-898516969e6a"]
```

**Example request:**
```bash
curl -s -X POST "https://api.isthereanydeal.com/games/overview/v2?country=US&key=$ITAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["018d937f-012f-73b8-ab2c-898516969e6a"]'
```

**Response:**
```json
{
  "prices": [
    {
      "id": "018d937f-012f-73b8-ab2c-898516969e6a",
      "current": {
        "shop": { "id": 61, "name": "Steam" },
        "price": { "amount": 39.99, "amountInt": 3999, "currency": "USD" },
        "regular": { "amount": 59.99, "amountInt": 5999, "currency": "USD" },
        "cut": 33,
        "timestamp": "2024-01-15T12:00:00Z",
        "expiry": "2024-01-22T12:00:00Z",
        "url": "https://..."
      },
      "lowest": {
        "shop": { "id": 61, "name": "Steam" },
        "price": { "amount": 29.99, "amountInt": 2999, "currency": "USD" },
        "regular": { "amount": 59.99, "amountInt": 5999, "currency": "USD" },
        "cut": 50,
        "timestamp": "2023-12-22T12:00:00Z"
      }
    }
  ],
  "bundles": []
}
```

Key response fields:
- `prices[].current` — current best deal (null if no active deal)
- `prices[].current.shop.name` — store with the best current price
- `prices[].current.price.amount` — best current price
- `prices[].current.cut` — current discount percentage
- `prices[].current.expiry` — when the deal expires (can be null)
- `prices[].lowest` — all-time historical low price
- `prices[].lowest.price.amount` — lowest price ever recorded
- `prices[].lowest.cut` — biggest discount ever
- `prices[].lowest.timestamp` — when the lowest price occurred
- `bundles` — any active bundles containing the queried games

---

### GET /games/info/v2

Get detailed game information including tags, developers, reviews.

**Parameters:**
- `id` (required, UUID) — game ID
- `key` (required) — API key

**Example request:**
```bash
curl -s "https://api.isthereanydeal.com/games/info/v2?id=018d937f-012f-73b8-ab2c-898516969e6a&key=$ITAD_API_KEY"
```

**Response includes:** title, type, tags, developers, publishers, reviews (Steam score), release date, URLs.

---

## Notes

- All POST endpoints accept JSON arrays of game UUIDs in the request body
- Maximum 200 game IDs per request
- Country defaults to "US" if not specified
- No hard rate limit, but heuristic abuse detection is in place
- Affiliate links in URLs must not be modified (per Terms of Service)
- API key obtained free at https://isthereanydeal.com/dev/app/
