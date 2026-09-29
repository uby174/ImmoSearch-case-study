# 🏠 Case Study: ImmoSearch

> **German rental search aggregator across 5 major portals — built on reversed-engineered APIs, multi-portal parallel search, and zero-dependency architecture.**
> Source code is private. This repo documents the architecture, engineering decisions, and technical depth.

---

## 📋 Overview

ImmoSearch solves a real problem: searching for rental apartments in Germany means checking 5+ portals separately, each with its own UI, filters, and data format. ImmoSearch queries all five simultaneously, normalizes the results into a single unified listing format, and presents them in one view — no API keys required.

### What it covers

| Portal | Technique |
|--------|----------|
| **ImmoScout24** | Reverse-engineered mobile app API (`POST /search/list`) |
| **Immowelt** | Server-rendered page parsing via stable `data-testid` attributes |
| **Kleinanzeigen** | Embedded JSON schema.org extraction |
| **WG-Gesucht** | String-parsed result page markup |
| **ohne-makler** | Card-based DOM parsing with bookmark IDs |

**No headless browser involved.** Every adapter reads either a JSON API or server-rendered HTML. Fast, lightweight, and runs on any Node.js host.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│            ImmoSearch Frontend (React)           │
│  Search Form ──▶ Results List ──▶ Map View      │
│  Natural Language Query Parsing (Gemini / regex) │
└──────────────────────┬──────────────────────────┘
                       │ POST /api/search-grounding
              ┌────────▼────────┐
              │  Express Server  │
              │  (server.ts)     │
              └────────┬─────────┘
                       │
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
    ┌─────────┐  ┌──────────┐  ┌──────────────┐
    │Parallel │  │ Normalise │  │   Dedupe &   │
    │  Fetch  │  │ to Listing│  │    Sort      │
    │ (5 tap) │  │  schema   │  │              │
    └────┬────┘  └──────────┘  └──────────────┘
         │
    ┌────┴────────────────────────────────┐
    │                                      │
    ▼              ▼            ▼         ▼
┌────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐
│ImmoScout│  │ Immowelt │  │ Klein- │  │WG-Gesucht│
│ mobile  │  │ markup   │  │anzeigen│  │  markup  │
│  API    │  │ parser   │  │ JSON   │  │  parser  │
└────────┘  └──────────┘  └────────┘  └──────────┘
    ┌──────────┐
    │ohne-     │
    │makler    │
    └──────────┘
```

---

## 🔧 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 19, Vite, Tailwind CSS 4, Motion (animations) |
| **Backend** | Express.js, TypeScript |
| **Query Parsing** | Gemini (when key present) → rule-based regex fallback |
| **Geocoding** | Google Maps API → OpenStreetMap Nominatim fallback |
| **Database** | Firebase (auth + saved listings) |
| **Mobile** | Native Android APK (thin client) |
| **Scraping fallback** | web-eyes (Scrapling / curl_cffi TLS fingerprint) |
| **License Server** | Cloudflare Worker (Wrangler) |

---

## 🔄 Key Engineering Patterns

### Portal Adapter Interface

Every portal follows the same `PortalAdapter` contract:

```typescript
interface PortalAdapter {
  name: string;
  search(query: SearchQuery): Promise<Listing[]>;
}
```

The engine runs all 5 adapters in parallel, then:
1. Normalize all results into the `Listing` shape
2. Filter (commission-free, price, rooms, size)
3. Deduplicate (same listing might appear on multiple portals)
4. Sort by relevance

A portal that fails is **reported, never substituted** — nothing on screen is fabricated.

### Reverse-Engineering: ImmoScout24 Mobile API

ImmoScout24's mobile app uses a JSON API at `api.mobile.immobilienscout24.de`. The adapter spoofs the app's User-Agent string and sends proper search parameters:

- `pagesize` up to 100 (one request usually suffices)
- `equipment` filters (balcony, kitchen, garden, parking)
- Radius search around geocoded coordinates
- `searchType=region` with path-style geocodes for city-wide results

**HTTP 412** is how unsupported parameter combinations are diagnosed.

### Stable Parsing (Immowelt)

Immowelt's page uses build-hashed CSS class names that change on every deploy. The adapter reads `data-testid` attributes instead (`cardmfe-price-testid`, `cardmfe-keyfacts-testid`, `cardmfe-description-box-address`), which are stable across builds because they're used by Immowelt's own E2E tests.

### Anti-Bot Stealth Fallback

When a portal request comes back blocked (401/403/429/503) or returns a bot-wall challenge page, the request is retried through `web_eyes_fetch.py` using Scrapling's `curl_cffi` transport — presenting a browser TLS fingerprint instead of Node's native one. This is deliberately off the happy path; it's insurance against portals that start blocking.

### Natural Language Query Parsing

Users type `3 Zimmer bis 1200 € in Berlin` and the system understands:
- City: Berlin
- Min rooms: 3
- Max price: 1200

Two modes:
1. **Gemini-powered** (when API key present) — full natural language understanding
2. **Rule-based fallback** — regex patterns for German real estate terminology

---

## 📊 API Endpoints

| Route | Purpose |
|-------|---------|
| `POST /api/search-grounding` | Main search — returns `[]` with per-portal counts in `X-Portal-Report` header |
| `GET /api/portals` | Registered adapters + rejected portals |
| `POST /api/portals/search` | Search with diagnostics; accepts `portals` filter |
| `POST /api/parse-query` | Natural language → structured filters |
| `POST /api/geocode` | City/PLZ → coordinates (404 = not found, never invents) |
| `GET /api/redirect?url=` | Safe outbound hop — restricted to 5 supported domains only |

---

## 📱 Android Thin Client

The Android APK is a thin WebView wrapper around the web app:

- **Offline state:** Saved listings persist locally on the device
- **No API keys needed:** All 5 portals work without any account or key
- **Privacy:** Saved listings never leave the device
- **Distribution:** Direct APK sideload (not Play Store)

---

## 🎯 Key Engineering Decisions

### 1. No Headless Browser

Every competing solution uses Playwright/Puppeteer to render full pages. ImmoSearch reads server-rendered HTML or JSON APIs directly. This means:
- **10x faster** page loads (no Chromium startup overhead)
- **Runs on minimal infrastructure** (Cloudflare Worker, small VPS)
- **Lower detection surface** (no browser fingerprint to block)

### 2. Graceful Degradation

| Component | Degradation |
|-----------|-------------|
| Gemini API key missing | Falls back to regex parser |
| Google Maps key missing | Falls back to OpenStreetMap Nominatim |
| Portal blocked | Reports failure, other portals still return results |
| web-eyes venv missing | Original error is preserved, portal reports failure |

### 3. One Listing Shape for All Portals

The `Listing` type unifies:
- **Price:** Some portals report warm rent (Gesamtmiete), some Kaltmiete. Each listing carries `priceKind: 'warm' | 'cold' | undefined` and the UI labels accordingly.
- **Location:** Only ImmoScout24 and Immowelt publish street/postcode. Missing fields are left empty, never guessed.
- **Date:** Relative dates (`vor 8 Tagen`) normalized to UTC.

### 4. Commission-Free Filtering

`onlyProvisionsfrei` hides Immowelt and most Kleinanzeigen results because neither portal publishes a commission flag. The system is transparent about this limitation in the UI rather than guessing.

### 5. Throttled and Honest

- Portal requests throttled to one per host every 400ms with retry/backoff on 429 and 5x
- No fabricated data — a failed portal is named in red, never replaced
- `parse:test` and `portals:test` scripts verify parsing rules and live portal connectivity

---

## 💡 Lessons Learned

1. **Reverse-engineering mobile APIs** is often more reliable than scraping web pages — mobile endpoints tend to be more stable because the app binary pins a specific API contract
2. **UI test attributes (`data-testid`)** survive CSS class churn because they exist specifically to be stable reference points
3. **Fallbacks are mandatory** for scraper-based systems — portals change markup, deploy anti-bot measures, and rate-limit aggressively
4. **German real estate terminology** is its own mini-language (`warm`/`cold` rent, `provisionsfrei`, `Zimmer` as a fractional unit like `2,5-Zimmer`)
5. **Honest failure reporting** builds more trust than silent degradation

---

**Author:** Muhammad Umer | [GitHub](https://github.com/uby174) | [LinkedIn](https://www.linkedin.com/in/muhammad-umer-/)