# voiteluoljyt API

Official public resources for **voiteluoljyt.fi** — API documentation and companion files (e.g., `llms.txt`).

> **Scope**: Read‑only, anonymous REST API for product & category data. No authentication. No PII. Public by design.

---

## About voiteluoljyt.fi

[voiteluoljyt.fi](https://voiteluoljyt.fi) is a Finnish wholesale store for lubricants and engine oils.  
We provide a professional catalogue covering motor oils, transmission fluids, greases and more.  
Our site highlights **Best Selling Engine Oils – Top Choices in 2025**, helping customers discover trusted and popular products.

This repository hosts the public API and related resources for developers and AI agents.

---

## Base URL & Versioning

- **Base URL (production)**: `https://voiteluoljyt.fi`
- **API root**: `/api/v1`
- **Discovery (meta)**: `/api/v1/meta`  
  _Well‑known alias_: `/.well-known/api.json` (mirror of `/api/v1/meta`).
- **Version string**: returned by `/api/v1/meta` (static semantic date, e.g. `2025-08-23`).

> Preview/staging deployments use the same paths but are explicitly **noindex** and may be rate‑limited. Use the production domain for anything public.

---

## Protocol & Headers

- **Methods**: `GET`, `HEAD`, `OPTIONS` (others → `405`)
- **CORS**: `Access-Control-Allow-Origin: *` (read‑only)
- **Content-Type**: `application/json; charset=utf-8`
- **Cache-Control**: `public, s-maxage=300, stale-while-revalidate=86400`

---

## Endpoints

### 1) API metadata
`GET /api/v1/meta`

Returns API name, version, endpoint URLs, and contact email.

**Response (example)**
```json
{
  "name": "Voiteluöljyt Public API",
  "version": "2025-08-23",
  "docs_url": "https://voiteluoljyt.fi/api/v1/meta",
  "endpoints": {
    "products": "/api/v1/products",
    "product": "/api/v1/products/{idOrSlug}",
    "categories": "/api/v1/categories"
  },
  "formats": ["application/json"],
  "contact": "info@voiteluoljyt.fi",
  "rate_limit_hint": "60 req/min (soft)"
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

Returns only **available** products. Intended for listing & discovery.

**Query params**
- `category_id` — number. If a top category (e.g., `110`), includes its subcategories implicitly.
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
      "available": true,
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

# Filter by category (incl. its subcategories)
curl -s 'https://voiteluoljyt.fi/api/v1/products?category_id=110&limit=10' | jq

# Incremental sync (updated after timestamp)
curl -s 'https://voiteluoljyt.fi/api/v1/products?updated_after=2025-08-01T00:00:00Z' | jq
```

**Notes**
- Listing **omits prices** by design. Use product detail for per‑volume offers.
- `url` is absolute and suitable for linking.

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
  "available": true,
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
- `price_cents = round(price_eur * 100)`  
- `currency` is always `EUR`.  
- `volume` is a raw, human‑readable label (e.g., `IBC 1000L`).

---

## Pagination

Keyset strategy on `(updated_at DESC, id ASC)`. The server returns an opaque `next_cursor` you can pass back as `cursor=...`.

**Example flow**
1. Call `/api/v1/products?limit=100` → get `next_cursor` in response.
2. If non‑null, fetch `/api/v1/products?limit=100&cursor=<value>`.
3. Repeat until `next_cursor = null`.

---

## Error Handling

- `400 { "error": "bad_request" }` – invalid query or server validation failed.
- `404 { "error": "not_found" }` – product not found or not available.
- `405` – methods other than `GET/HEAD/OPTIONS`.

---

## llms.txt (for LLMs)

The site provides an **LLMs guide** that lists core pages, categories (EN/FI), and machine‑readable references:

- `https://voiteluoljyt.fi/llms.txt`

The guide intentionally links to `.md` versions of selected pages for cleaner ingestion by AI tools, while `robots.txt` prevents those Markdown pages from being indexed by search engines.

---

## Roadmap / Optional Artifacts

- **OpenAPI**: `/.well-known/openapi.json` (TBD). For now, use `/api/v1/meta` for discovery.
- **Language coverage**: Endpoint payloads are currently English‑first; FI content is selectively exposed on the website level.

---

## Contact

`info@voiteluoljyt.fi`

---

## Notes on Use & Limits

- Anonymous, read‑only access intended for SSR, edge caching, and AI agents.  
- Consider a soft rate limit of ~**60 requests/min/IP**. Heavy crawlers should throttle accordingly.

---

## 🇫🇮 Suomi

README.FI.md

