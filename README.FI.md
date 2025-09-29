# voiteluoljyt API

Viralliset julkiset resurssit sivustolle **voiteluoljyt.fi** — API-dokumentaatio ja oheistiedostot (esim. `llms.txt`).

> **Kattavuus**: Vain luku -tyyppinen, anonyymi REST API tuotetietoihin ja kategorioihin. Ei autentikointia. Ei henkilötietoja. Julkinen suunnittelultaan.

---

## Tietoa voiteluoljyt.fi:stä

[voiteluoljyt.fi](https://voiteluoljyt.fi) on suomalainen tukkukauppa voiteluaineille ja moottoriöljyille.  
Tarjoamme ammattimaisen katalogin, joka kattaa moottoriöljyt, vaihteistoöljyt, rasvat ja paljon muuta.  
Sivustomme esittelee **Parhaiten myyvät moottoriöljyt – huippuvalinnat vuodelle 2025**, jotta asiakkaat löytävät luotettavia ja suosittuja tuotteita.

Tämä repositorio sisältää julkisen API:n ja siihen liittyvät resurssit kehittäjille ja tekoälyagenteille.

---

## Perus-URL ja versiointi

- **Perus-URL (tuotanto)**: `https://voiteluoljyt.fi`
- **API-juuri**: `/api/v1`
- **Discovery (meta)**: `/api/v1/meta`  
  _Tunnettu alias_: `/.well-known/api.json` (peili `/api/v1/meta`).
- **Versiomerkkijono**: palautetaan `/api/v1/meta` -kutsussa (esim. `2025-08-23`).

> Esiversiot käyttävät samoja polkuja, mutta niissä on **noindex**-asetus ja mahdollinen rajoitettu nopeus. Käytä tuotantodomainia julkisiin sovelluksiin.

---

## Protokolla ja headerit

- **Metodit**: `GET`, `HEAD`, `OPTIONS` (muut → `405`)
- **CORS**: `Access-Control-Allow-Origin: *` (vain luku)
- **Content-Type**: `application/json; charset=utf-8`
- **Cache-Control**: `public, s-maxage=300, stale-while-revalidate=86400`

---

## Endpointit

### 1) API metadata
`GET /api/v1/meta`

Palauttaa API:n nimen, version, endpointit ja yhteystiedon.

**Vastaus (esimerkki)**
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

### 2) Kategoriat (tasalista)
`GET /api/v1/categories`

- Lähde on staattinen kategoriapuu, joka litistetään muotoon `{ id, name, parent_id }`.
- Nimet ovat englanniksi; ID:t pysyvät vakaina. Pääkategoriat: `parent_id = null`.

**Vastaus (katkaistu)**
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

### 3) Tuotteet — listaus (ei hintoja)
`GET /api/v1/products`

Palauttaa vain **saatavilla olevat** tuotteet. Tarkoitettu listaukseen ja hakuihin.

**Kyselyparametrit**
- `category_id` — numero. Jos pääkategoria (esim. `110`), sisältää automaattisesti alaluokat.
- `updated_after` — ISO8601 UTC -suodatin.
- `limit` — oletus 50, maksimi 100.
- `cursor` — keyset-paginaation tunniste (base64url).

**Vastaus (esimerkki)**
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
# Ensimmäinen sivu (oletus limit=50)
curl -s 'https://voiteluoljyt.fi/api/v1/products' | jq

# Suodata kategorian mukaan (sis. alaluokat)
curl -s 'https://voiteluoljyt.fi/api/v1/products?category_id=110&limit=10' | jq

# Inkrementaalinen synkronointi (päivityksen jälkeen)
curl -s 'https://voiteluoljyt.fi/api/v1/products?updated_after=2025-08-01T00:00:00Z' | jq
```

**Huomiot**
- Listauksessa **ei ole hintoja**. Käytä tuotedetailia saadaksesi tarjoukset.
- `url` on absoluuttinen ja sopii linkitykseen.

---

### 4) Tuote — detail (sis. `offers[]`)
`GET /api/v1/products/{idOrSlug}`

- Haku yrittää ensin **UUID id**:llä, sitten **slug**:illa.
- Palauttaa vain jos `available = true`.
- Sisältää englanninkielisen `description`-kentän ja `offers[]`-taulukon.

**Vastaus (esimerkki)**
```json
{
  "id": "UUID",
  "slug": "super-lube-10w-40",
  "name": "Super Lube 10W-40",
  "description": "High-performance engine oil for ...",
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

**Tarjouskenttien mappaus**
- `price_cents = round(price_eur * 100)`  
- `currency` on aina `EUR`.  
- `volume` on vapaatekstinä (esim. `IBC 1000L`).

---

## Paginointi

Keyset-strategia `(updated_at DESC, id ASC)` mukaan. Palvelin palauttaa `next_cursor`-tunnisteen, jota voi käyttää seuraavassa kutsussa `cursor=...`.

---

## Virheiden käsittely

- `400 { "error": "bad_request" }` – virheellinen kysely.
- `404 { "error": "not_found" }` – tuotetta ei löytynyt tai ei ole saatavilla.
- `405` – muut metodit kuin `GET/HEAD/OPTIONS`.

---

## llms.txt (tekoälylle)

Sivustolla on **LLMs guide**, joka listaa keskeiset sivut, kategoriat (EN/FI) ja koneellisesti luettavat viittaukset:

- `https://voiteluoljyt.fi/llms.txt`

---

## Roadmap / Lisäosat

- **OpenAPI**: `/.well-known/openapi.json` (TBD). Tällä hetkellä käytä `/api/v1/meta`.
- **Kielikattavuus**: Payloadit ovat pääosin englanniksi; FI-sisältöä on rajatusti sivustotasolla.

---

## Yhteystiedot

`info@voiteluoljyt.fi`

---

## Käyttörajoitukset

- Julkinen, vain luku. Tarkoitettu SSR-, edge caching- ja AI-agenttikäyttöön.  
- Suositeltu rajoitus: ~**60 pyyntöä/min/IP**. Raskaat crawlerit säädettävä.

---
