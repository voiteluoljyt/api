# voiteluoljyt API

Official public resources for **voiteluoljyt.fi** — API documentation and companion files (e.g., `llms.txt`).

> **Scope**: Read‑only, anonymous REST API for product & category data. No authentication. No PII. Public by design.

---

## About voiteluoljyt.fi

[voiteluoljyt.fi](https://voiteluoljyt.fi) is a Finnish wholesale store for lubricants and engine oils.  
We provide a professional catalogue covering motor oils, transmission fluids, greases and more.  
Our site highlights **Best Selling Engine Oils – Top Choices in 2026**, helping customers discover trusted and popular products.

This repository hosts the public API and related resources for developers and AI agents.

---

## Base URL & Versioning

- **Base URL (production)**: `https://voiteluoljyt.fi`
- **API root**: `/api/v1`
- **Discovery (meta)**: `/api/v1/meta`
- **Well‑known alias**: `/.well-known/api.json` (mirror of `/api/v1/meta`)
- **OpenAPI 3.1.0**: `/.well-known/openapi.json`
- **AI plugin manifest**: `/.well-known/ai-plugin.json`
- **LLM guide**: `/llms.txt` (also returned as `ai_guide` in `/api/v1/meta`)
- **Version string**: returned by `/api/v1/meta` (static semantic date, e.g. `2025-08-23`)

> Preview/staging deployments use the same paths but are explicitly **noindex** and may be rate‑limited. Use the production domain for anything public.

---

## Protocol & Headers

- **Methods**: `GET`, `HEAD`, `OPTIONS` (others → `405`)
- **CORS**: `Access-Control-Allow-Origin: *` (read‑only)
- **CORS methods**: `GET, HEAD, OPTIONS`
- **CORS headers**: `Content-Type, Authorization`
- **Content-Type**: `application/json; charset=utf-8`
- **Cache-Control**: `public, s-maxage=300, stale-while-revalidate=86400`

---

## Endpoints

### 1) API metadata
`GET /api/v1/meta`

Returns API name, version, endpoint URLs, AI guide link, and contact email.

**Response (example)**
```json
{
  "name": "Voiteluöljyt Public API",
  "version": "2025-08-23",
  "docs_url": "https://voiteluoljyt.fi/api/v1/meta",
  "ai_guide": "https://voiteluoljyt.fi/llms.txt",
  "endpoints": {
    "products": "/api/v1/products",
    "product": "/api/v1/products/{idOrSlug}",
    "categories": "/api/v1/categories"
  },
  "formats": ["application/json"],
  "contact": "info@voiteluoljyt.fi",
  "rate_limit_hint": "60 req/min per IP"
}
```

**cURL**
```bash
curl -s https://voiteluoljyt.fi/api/v1/meta | jq
```

---

### 2) Categories (flat list)
`GET /api/v1/categories`

- Source is a static category tree flattened to `{ id, name, parent_id }`.
- English names; IDs are stable. Parent categories have `parent_id = null`.

**Response (truncated)**
```json
{
  "items": [
    { "id": 110, "name": "Motor Oil", "parent_id": null },
    { "id": 111, "name": "Cars and Vans", "parent_id": 110 },
    { "id": 112, "name": "Heavy Duty", "parent_id": 110 }
  ],
  "next_cursor": null
}
```

**cURL**
```bash
curl -s https://voiteluoljyt.fi/api/v1/categories | jq
```

---

### 3) Products — list (no prices)
`GET /api/v1/products`

Returns only **available** products (`available=true` in the database). Intended for listing & discovery.  
The `available` flag is used for filtering but is **not** returned in the JSON payload.

**Query params**
- `category_id` — number. If a main category (id divisible by 10, e.g. `110`), includes its subcategories implicitly.
- `updated_after` — ISO8601 UTC filter.
- `limit` — default 50, max 100.
- `cursor` — keyset pagination token (opaque base64url of `{updated_at,id}`).

**Response (example)**
```json
{
  "items": [
    {
      "id": "UUID",
      "slug": "super-lube-10w-40",
      "name": "Super Lube 10W-40",
      "brand": "Super Lube",
      "category": { "id": 110 },
      "updated_at": "2025-08-20T10:00:00Z",
      "url": "https://voiteluoljyt.fi/product/super-lube-10w-40"
    }
  ],
  "next_cursor": null
}
```

**cURL**
```bash
# First page (default limit=50)
curl -s 'https://voiteluoljyt.fi/api/v1/products' | jq

# Filter by category (incl. its subcategories when main category)
curl -s 'https://voiteluoljyt.fi/api/v1/products?category_id=110&limit=10' | jq

# Incremental sync (updated after timestamp)
curl -s 'https://voiteluoljyt.fi/api/v1/products?updated_after=2025-08-01T00:00:00Z' | jq
```

**Notes**
- Listing **omits prices** by design. Use product detail for per‑volume offers.
- `url` is absolute and suitable for linking when the server can resolve the host; it may be omitted otherwise.

---

### 4) Product — detail (with `offers[]`)
`GET /api/v1/products/{idOrSlug}`

- Lookup tries **UUID id** first, then **slug** fallback.
- Returns only if `available = true`.
- Includes English `description` and a compact `offers[]` array.

**Response (example)**
```json
{
  "id": "UUID",
  "slug": "super-lube-10w-40",
  "name": "Super Lube 10W-40",
  "description": "High‑performance engine oil for ...",
  "brand": "Super Lube",
  "image_url": "https://voiteluoljyt.fi/images/products/super-lube-10w-40.jpg",
  "category": { "id": 110 },
  "updated_at": "2025-08-20T10:00:00Z",
  "url": "https://voiteluoljyt.fi/product/super-lube-10w-40",
  "offers": [
    { "price_cents": 140000, "currency": "EUR", "volume": "IBC 1000L" },
    { "price_cents": 18500,   "currency": "EUR", "volume": "Drum 205L" }
  ]
}
```

**cURL**
```bash
curl -s https://voiteluoljyt.fi/api/v1/products/super-lube-10w-40 | jq
```

**Offers mapping**
- Source columns: `price1..price5` (JSON per column), e.g. `{ "price": 185, "volume": "Drum 205L" }`
- `price_cents = round(price_eur * 100)`
- `currency` is always `EUR`
- `volume` is a raw, human‑readable label (e.g., `IBC 1000L`)
- Non‑numeric prices are skipped; internal fields like `space` are not returned

**Image URL**
- Absolute `http(s)://` URLs are returned as‑is
- Relative filenames are resolved to `https://<domain>/images/products/<filename>`

---

## Pagination

Keyset strategy on `(updated_at DESC, id ASC)`. The server returns an opaque `next_cursor` you can pass back as `cursor=...`.

**Example flow**
1. Call `/api/v1/products?limit=100` → get `next_cursor` in response.
2. If non‑null, fetch `/api/v1/products?limit=100&cursor=<value>`.
3. Repeat until `next_cursor = null`.

The cursor is a base64url encoding of `{ "updated_at": "...", "id": "..." }`. Treat it as opaque.

---

## Error Handling

- `400 { "error": "bad_request" }` – invalid query, invalid cursor, or server validation failed.
- `404 { "error": "not_found" }` – product not found or not available.
- `429 { "error": "rate_limit_exceeded" }` – more than 60 requests per minute from one IP.
  - Response headers: `Retry-After`, `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`
- `405` – methods other than `GET/HEAD/OPTIONS`.

---

## Machine‑readable specs

For API clients, GPT Actions, and AI tooling:

| Resource | URL |
|----------|-----|
| API meta | `https://voiteluoljyt.fi/api/v1/meta` |
| Well‑known discovery | `https://voiteluoljyt.fi/.well-known/api.json` |
| OpenAPI 3.1.0 | `https://voiteluoljyt.fi/.well-known/openapi.json` |
| AI plugin manifest | `https://voiteluoljyt.fi/.well-known/ai-plugin.json` |
| LLM guide | `https://voiteluoljyt.fi/llms.txt` |

**Smoke tests**
```bash
curl -s https://voiteluoljyt.fi/api/v1/meta | jq
curl -s https://voiteluoljyt.fi/.well-known/api.json | jq
curl -s https://voiteluoljyt.fi/.well-known/openapi.json | jq
curl -s https://voiteluoljyt.fi/.well-known/ai-plugin.json | jq
curl -s https://voiteluoljyt.fi/api/v1/categories | jq
curl -s 'https://voiteluoljyt.fi/api/v1/products?limit=10' | jq
curl -s 'https://voiteluoljyt.fi/api/v1/products?category_id=110&limit=5' | jq
curl -s https://voiteluoljyt.fi/api/v1/products/super-lube-10w-40 | jq
```

---

## llms.txt (for LLMs)

The site provides an **LLMs guide** that lists core pages, categories (EN/FI), and machine‑readable references:

- `https://voiteluoljyt.fi/llms.txt`

The guide intentionally links to `.md` versions of selected pages for cleaner ingestion by AI tools, while `robots.txt` prevents those Markdown pages from being indexed by search engines.

The same URL is also exposed in `/api/v1/meta` as the `ai_guide` field.

---

## Language coverage

Endpoint payloads are **English‑first** (`name`, `description`, category names).  
Finnish content is selectively exposed on the website level (`/fi/*`).

---

## Contact

`info@voiteluoljyt.fi`

---

## Notes on Use & Limits

- Anonymous, read‑only access intended for SSR, edge caching, and AI agents.
- **Rate limit**: 60 requests per minute per IP, enforced via Redis (Upstash).
- Excess requests return HTTP `429` with `Retry-After` and `RateLimit-*` headers.
- If the rate limiter is temporarily unavailable, requests are allowed (fail‑open).
- Heavy crawlers should still throttle proactively and use pagination + `updated_after` for incremental syncs.

---

## 🇫🇮 Suomi

README.FI.md

---

Last updated: 2026‑07‑11
