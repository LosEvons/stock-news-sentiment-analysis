# stock-news-sentiment-analysis

**Stock Price Timeline with News Sentiment Analysis**
Version 1.0 · May 2026

---

## 1. Overview

stock-news-sentiment-analysis is a web application that overlays news sentiment data onto a stock price timeline. The user searches for a ticker symbol and selects a date range; the application fetches historical OHLCV candle data and related news articles, scores each article's headline using an ML sentiment model, and renders everything in a unified interactive chart. News events appear as annotated markers on the price timeline — clicking a marker reveals the article headline, source, publication date, and sentiment verdict.

The project is intended as a portfolio piece demonstrating a polyglot microservice architecture. Each service is independently containerised and communicates over a shared Docker network. There is deliberately no persistent database: all data is held in an in-memory TTL cache within the stock service, keeping the infrastructure lightweight and the setup frictionless.

---

## 2. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | Angular 17 + TypeScript | Demonstrates enterprise-grade SPA skills; strict typing throughout |
| Charting | TradingView Lightweight Charts | MIT-licensed, purpose-built for financial time series; small bundle |
| Stock service | Go (net/http + go-cache) | Performant HTTP server; idiomatic concurrency for fan-out API calls |
| Sentiment service | Python 3.12 + FastAPI + HuggingFace Transformers | Best ecosystem for ML model inference; FastAPI is async and lightweight |
| ML model | ProsusAI/finbert | Fine-tuned on financial news; no additional training required |
| Cache | In-memory TTL (go-cache / patrickmn) | Zero infrastructure dependency; adequate for a portfolio-scale project |
| Containerisation | Docker + Docker Compose | Single-command startup; isolated environments per service |

> **Note:** Angular is a strong choice when targeting enterprise employers (large companies, finance sector). If target employers skew toward startups, React may read more naturally. The backend design is frontend-agnostic.

---

## 3. Data Sources

### 3.1 Finnhub (finnhub.io)

A single Finnhub API key covers both data needs — stock candles and company news — under a free tier that allows 60 requests per minute. No secondary API account is required.

| Data | Endpoint | Key Parameters |
|---|---|---|
| Historical OHLCV candles | `GET /stock/candle` | symbol, resolution (D/W/M), from/to (Unix timestamps) |
| Company news articles | `GET /company-news` | symbol, from/to (YYYY-MM-DD) |
| Ticker symbol search | `GET /search` | q (search string) |
| Company profile | `GET /stock/profile2` | symbol |

**Base URL:** `https://finnhub.io/api/v1` · **Authentication:** `token` query parameter or `X-Finnhub-Token` header.

### 3.2 Rate Limit Handling

The stock service must enforce Finnhub's 60 req/min ceiling. Implement a simple token-bucket rate limiter in Go using `golang.org/x/time/rate`. On HTTP 429 responses, apply exponential backoff with jitter. The in-memory cache (see Section 5.1) reduces external calls significantly during repeated queries for the same symbol and date range.

---

## 4. Architecture

### 4.1 Service Topology

Three containers communicate over a private Docker bridge network. Only the frontend container exposes a host port.

```
  ┌──────────────────┐          ┌──────────────────────┐
  │   Angular SPA    │  REST    │    stock-service      │
  │   (nginx :80)    │ ───────► │    (Go :8080)        │
  └──────────────────┘          └──────────┬───────────┘
         ▲ host port 4200                   │ REST
                                 ┌──────────▼───────────┐
                                 │  sentiment-service   │
                                 │  (Python/FastAPI :8000)│
                                 └──────────────────────┘
                                           ▲
                                    Docker network only
                                   (not exposed to host)
```

> **Note:** The sentiment service is not reachable from outside Docker. All external traffic enters through the Angular/nginx container, which proxies API calls to stock-service. stock-service calls sentiment-service internally.

### 4.2 Request Flow

The following sequence describes what happens when a user loads a chart for a given ticker and date range:

1. Angular sends `GET /api/candles` and `GET /api/news` to stock-service.
2. stock-service checks its in-memory cache for each request.
3. On a cache miss, stock-service fetches candles and news from Finnhub.
4. The raw news array is passed through the filtering pipeline: source whitelisting removes articles from non-major outlets, then a company-relevance check removes articles where the company name or ticker does not appear in the headline.
5. For each article that survives filtering, stock-service POSTs the headline to sentiment-service and receives a label and confidence score. This fan-out is done concurrently using goroutines.
6. Enriched and filtered news is stored in the cache alongside the candle data. The response also includes the raw pre-filter count so the frontend can display a transparency indicator.
7. stock-service responds to Angular with both candles and enriched news in a single combined payload.
8. Angular renders the price chart and maps news markers onto the timeline.

> **Note:** Sentiment scoring happens at ingestion time (step 5), not at query time. Once an article has been scored and cached, subsequent requests for the same symbol and date range are served entirely from memory with no external calls.

---

## 5. Service Specifications

### 5.1 stock-service (Go)

The stock service is the central backend. It owns all external API communication, caching, and orchestration.

#### REST API (consumed by Angular)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/candles` | Returns OHLCV candle array. Query params: `symbol`, `from` (ISO date), `to` (ISO date), `resolution` (D/W/M, default D) |
| `GET` | `/api/news` | Returns news articles with sentiment. Query params: `symbol`, `from`, `to` |
| `GET` | `/api/search` | Ticker symbol search. Query param: `q` |
| `GET` | `/healthz` | Health check endpoint for Docker |

#### Response shapes

```json
// GET /api/candles
{ "symbol": "AAPL", "resolution": "D",
  "candles": [{ "t": 1704067200, "o": 185.2, "h": 188.4, "l": 184.1, "c": 187.0, "v": 52341200 }] }

// GET /api/news
{ "symbol": "AAPL",
  "totalFetched": 934,
  "totalFiltered": 83,
  "articles": [{ "id": 123456, "headline": "Apple beats earnings estimates",
    "source": "Reuters", "url": "...", "publishedAt": "2024-02-02T21:00:00Z",
    "sentiment": { "label": "positive", "score": 0.91 } }] }
```

#### News filtering pipeline

Before any article is scored, the raw list returned by Finnhub is passed through two sequential filters. Both are pure in-memory string operations and complete in under a millisecond regardless of article volume.

**Pass 1 — Source whitelist**

Retain only articles whose `source` field matches an entry in the whitelist below. All other articles are discarded before any further processing. Comparison is case-insensitive and checks for substring containment so minor source name variants (e.g. "Reuters UK") are captured.

| Whitelisted source | Category |
|---|---|
| Reuters | Wire service |
| Associated Press | Wire service |
| Bloomberg | Financial wire |
| Dow Jones | Financial wire |
| The Wall Street Journal | Financial newspaper |
| Financial Times | Financial newspaper |
| CNBC | Financial broadcast |
| MarketWatch | Financial news |
| Seeking Alpha | Financial analysis |
| Barron's | Financial magazine |
| Investor's Business Daily | Financial newspaper |
| Yahoo Finance | Aggregator (high editorial volume) |

> **Note:** This list should be kept in a config file or environment variable, not hardcoded, so it can be adjusted without recompiling.

**Pass 2 — Company relevance check**

After whitelisting, retain only articles where the company name or ticker symbol appears in the headline string. This removes market-roundup and sector-wide articles that Finnhub associates with a ticker but are not specifically about that company. The check is a case-insensitive substring match against both the ticker (e.g. `AAPL`) and the canonical company name retrieved from Finnhub's `/stock/profile2` endpoint (e.g. `Apple`).

```go
// Example: ticker = "AAPL", companyName = "Apple Inc."
// keywords = ["AAPL", "Apple"]

func isCompanyRelevant(headline string, keywords []string) bool {
    h := strings.ToLower(headline)
    for _, kw := range keywords {
        if strings.Contains(h, strings.ToLower(kw)) {
            return true
        }
    }
    return false
}
```

#### In-memory cache

Use `github.com/patrickmn/go-cache`. Cache keys are constructed as `symbol:resolution:from:to` for candles and `symbol:from:to` for news.

| Cache key type | TTL | Rationale |
|---|---|---|
| Candle data | 1 hour | Historical OHLC doesn't change; today's candle may update intraday |
| News articles + sentiment | 30 mins | New articles may appear; sentiment scores never change once computed |
| Ticker search results | 24 hours | Ticker metadata is stable |
| Company profile (name) | 24 hours | Used for relevance filter; stable |

> **Note:** Cache is lost on container restart. A future improvement would be replacing the in-memory cache with Redis without changing the service interface.

#### Dependencies

- `github.com/patrickmn/go-cache` — in-memory TTL cache
- `golang.org/x/time/rate` — token-bucket rate limiter
- `github.com/rs/cors` — CORS middleware for Angular dev server

> **Note:** The stock service makes one additional Finnhub call per unique symbol to `/stock/profile2` to retrieve the canonical company name for the relevance filter. This result is cached with a 24-hour TTL.

---

### 5.2 sentiment-service (Python)

A stateless FastAPI microservice. Loads `ProsusAI/finbert` at startup and keeps it in memory. Accepts a headline string, returns a sentiment label and confidence score.

#### Endpoint

```
POST /sentiment
Content-Type: application/json

// Request
{ "text": "Apple reports record quarterly revenue" }

// Response
{ "label": "positive", "score": 0.94 }

// Labels: "positive" | "negative" | "neutral"
// Score:  confidence float in range [0.0, 1.0]
```

#### Implementation notes

- Model loads once at application startup via `transformers.pipeline("text-classification", model="ProsusAI/finbert")`.
- No GPU required. CPU inference on a news headline takes 100–300 ms, which is acceptable since scoring runs asynchronously during ingestion.
- The service is intentionally stateless. It holds no cache and no persistent state. All caching of scores lives in the stock service.
- Expose a `GET /healthz` endpoint that confirms the model is loaded and returns 200.

#### Dependencies

- `fastapi` + `uvicorn[standard]`
- `transformers` + `torch` (CPU build; use `torch==2.x+cpu` for a smaller image)

> **Note:** The first startup will download the ProsusAI/finbert model weights (~400 MB). Pre-download the model in the Dockerfile using a `RUN python -c "from transformers import pipeline; pipeline(...)"` layer so the weights are baked into the image and startup is instant.

---

### 5.3 frontend (Angular + TypeScript)

#### Key components

| Component | Responsibility |
|---|---|
| `SearchBarComponent` | Ticker autocomplete; calls `/api/search` and populates a dropdown |
| `DateRangePickerComponent` | Controls the chart window; emits date-range change events |
| `StockChartComponent` | Renders TradingView Lightweight Charts candlestick chart with volume histogram |
| `NewsMarkerComponent` | Overlays news event pins on the chart timeline |
| `ArticlePanelComponent` | Side panel shown on marker click; displays headline, source, date, and sentiment badge |
| `SentimentFilterComponent` | Toggles positive / negative / neutral marker visibility |
| `FilterCountComponent` | Displays "X articles shown, Y filtered" transparency indicator using `totalFetched` and `totalFiltered` from the API response |
| `AppService` | HTTP client wrapper; makes typed calls to stock-service endpoints |

#### Chart integration

TradingView Lightweight Charts is a vanilla JS library. Integrate it into Angular via a wrapper component that calls `createChart()` inside `ngAfterViewInit()` on a native `ElementRef`. The chart instance is stored as a component property and updated via Angular's change detection when new data arrives.

#### News markers on the chart

Lightweight Charts supports custom price markers via the `series.setMarkers()` API. Map each news article to a marker object positioned at its publication timestamp, with colour indicating sentiment (green = positive, red = negative, grey = neutral). Clicking a marker emits an Angular event that opens the `ArticlePanelComponent`.

#### nginx proxy configuration

In production Docker mode, the Angular app is served as static files by nginx. Add a `proxy_pass` rule so that requests to `/api/*` are forwarded to `http://stock-service:8080`, avoiding CORS issues and keeping the stock service off the public internet.

```nginx
# nginx.conf (relevant block)
location /api/ {
    proxy_pass http://stock-service:8080;
}
```

---

## 6. Docker Compose

```yaml
services:

  sentiment-service:
    build: ./sentiment-service
    # Not exposed to host — internal only

  stock-service:
    build: ./stock-service
    depends_on:
      sentiment-service:
        condition: service_healthy
    environment:
      - FINNHUB_API_KEY=${FINNHUB_API_KEY}
      - SENTIMENT_SERVICE_URL=http://sentiment-service:8000
    # Not exposed to host — fronted by nginx

  frontend:
    build: ./frontend
    ports:
      - "4200:80"
    depends_on:
      - stock-service
```

A single `FINNHUB_API_KEY` environment variable, supplied via a `.env` file at the project root, is the only external secret required. Do not commit this file; add it to `.gitignore`.

---

## 7. Repository Structure

```
stock-news-sentiment-analysis/
├── docker-compose.yml
├── .env.example             # FINNHUB_API_KEY=your_key_here
├── .gitignore
├── README.md
│
├── stock-service/           # Go
│   ├── Dockerfile
│   ├── go.mod
│   ├── main.go
│   ├── handlers/
│   ├── cache/
│   └── finnhub/             # Finnhub API client
│
├── sentiment-service/       # Python
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
│
└── frontend/                # Angular
    ├── Dockerfile
    ├── nginx.conf
    ├── angular.json
    ├── src/
    │   ├── app/
    │   │   ├── components/
    │   │   └── services/
    │   └── environments/
    └── package.json
```

---

## 8. Suggested Build Order

1. **sentiment-service** — standalone FastAPI app, fully testable with `curl` before any other service exists.
2. **stock-service (Finnhub candles only)** — fetch and return candle data, confirm rate limiting and caching work.
3. **stock-service (news + filtering + sentiment orchestration)** — add news fetching, run filtering pipeline, call sentiment-service, return enriched articles.
4. **frontend (chart only)** — render the candlestick chart from a hardcoded or mocked API response.
5. **frontend (news markers + panel)** — connect real API, overlay markers, implement click-to-open article panel.
6. **Docker Compose integration** — wire all three services together, test full flow end-to-end.
7. **README** — document setup, architecture diagram, known limitations, and future work.

---

## 9. Known Limitations & Future Work

| Limitation | Notes / Future Fix |
|---|---|
| In-memory cache lost on restart | Replace go-cache with Redis; the service interface does not need to change |
| US-market stocks only (free tier) | Finnhub paid plan unlocks global exchanges |
| Daily resolution only (MVP scope) | Finnhub supports intraday (1min, 5min); extend the `resolution` parameter |
| No user authentication | Add JWT auth if multi-user or watchlist persistence is needed |
| Sentiment model not fine-tuned | ProsusAI/finbert works well out of the box; fine-tuning on a labeled news dataset would improve neutral detection |
| No volume visualisation | Lightweight Charts supports histogram series; straightforward to add |

---

*End of specification.*
