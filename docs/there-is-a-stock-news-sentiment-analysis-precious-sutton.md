# Stock News Sentiment Analysis — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a 3-microservice stock-news-sentiment visualization system deployable via `docker compose up`, with an Angular SPA, a Go API/orchestration service, and a Python ML inference service.

**Architecture:** Angular 17 SPA (nginx :80 → host :4200) calls Go stock-service (:8080, Docker-internal) which fans out to Python sentiment-service (:8000, Docker-internal only). No persistent database — all state in in-memory TTL cache. Go stock-service owns all Finnhub API communication, news filtering, sentiment fan-out, and caching.

**Tech Stack:** Angular 17, TypeScript, TradingView Lightweight Charts v4 | Go 1.22, net/http, go-cache, golang.org/x/time/rate, github.com/rs/cors | Python 3.12, FastAPI, uvicorn, transformers==4.44.2, torch (CPU) | Docker Compose 3.8 | Finnhub API (free tier, 60 req/min)

---

## Architectural Decisions (spec overrides)

| Decision | Choice |
|----------|--------|
| Sentiment-service unavailable | stock-service returns HTTP 502 |
| Concurrent sentiment goroutines | Semaphore, max 20 |
| Testing scope | Unit tests only |
| Source whitelist config | `NEWS_SOURCE_WHITELIST` env var, comma-delimited |

---

## Execution Model

**Tasks 1 and 2 run in parallel** (independent — API contracts are fully defined in spec).
**Task 3 runs after Tasks 1 & 2.**
**Task 4 runs last.**

| Task | Service | Agent (`.claude/agents/`) |
|------|---------|---------------------------|
| 1 | `sentiment-service/` | `sentiment-analysis-engineer` |
| 2 | `stock-service/` | `go-backend-architect` |
| 3 | `frontend/` | `angular-frontend-dev` |
| 4 | `docker-compose.yml` + `.env.example` | `cicd-docker-engineer` |

---

## File Map

```
stock-news-sentiment-analysis/
├── sentiment-service/
│   ├── model.py
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── tests/
│       ├── __init__.py
│       └── test_main.py
├── stock-service/
│   ├── go.mod
│   ├── main.go
│   ├── config/config.go
│   ├── cache/cache.go
│   ├── finnhub/client.go
│   ├── sentiment/client.go
│   ├── filter/
│   │   ├── filter.go
│   │   └── filter_test.go
│   ├── handlers/
│   │   ├── candles.go + candles_test.go
│   │   ├── news.go + news_test.go
│   │   └── search.go
│   └── Dockerfile
├── frontend/
│   ├── src/app/
│   │   ├── models/candle.model.ts
│   │   ├── models/news.model.ts
│   │   ├── services/app.service.ts + spec
│   │   └── components/
│   │       ├── search-bar/
│   │       ├── date-range-picker/
│   │       ├── stock-chart/
│   │       ├── news-marker/
│   │       ├── article-panel/
│   │       ├── sentiment-filter/
│   │       └── filter-count/
│   ├── nginx.conf
│   └── Dockerfile
├── docker-compose.yml
└── .env.example
```

---

## Task 1: sentiment-service (Agent: `sentiment-analysis-engineer`)

**Dispatch brief for agent:** Build the Python 3.12 FastAPI sentiment microservice at `sentiment-service/`. Use `superpowers:test-driven-development`. Expose `POST /sentiment` (body: `{"text": string}`, response: `{"label": string, "score": float}`) and `GET /healthz`. Load ProsusAI/finbert once at startup. Pre-download model in Dockerfile. No caching — stock-service handles all caching.

**Files:**
- Create: `sentiment-service/model.py`
- Create: `sentiment-service/main.py`
- Create: `sentiment-service/requirements.txt`
- Create: `sentiment-service/Dockerfile`
- Create: `sentiment-service/tests/__init__.py`
- Create: `sentiment-service/tests/test_main.py`

---

- [ ] **Step 1.1: Create requirements.txt**

```
fastapi==0.115.0
uvicorn==0.30.6
transformers==4.44.2
torch==2.4.1+cpu
httpx==0.27.2
pytest==8.3.3
```

```bash
cd sentiment-service && pip install -r requirements.txt
```

- [ ] **Step 1.2: Write failing tests**

`sentiment-service/tests/test_main.py`:
```python
import pytest
from fastapi.testclient import TestClient


@pytest.fixture(scope="session")
def client() -> TestClient:
    from main import app
    return TestClient(app)


def test_sentiment_returns_label_and_score(client: TestClient) -> None:
    r = client.post("/sentiment", json={"text": "Apple reports record quarterly revenue"})
    assert r.status_code == 200
    data = r.json()
    assert data["label"] in ("positive", "negative", "neutral")
    assert 0.0 <= data["score"] <= 1.0


def test_sentiment_missing_text_is_422(client: TestClient) -> None:
    r = client.post("/sentiment", json={})
    assert r.status_code == 422


def test_healthz(client: TestClient) -> None:
    r = client.get("/healthz")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}
```

Run: `cd sentiment-service && pytest tests/ -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'main'`

- [ ] **Step 1.3: Implement model.py**

`sentiment-service/model.py`:
```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

_MODEL_NAME = "ProsusAI/finbert"


class SentimentModel:
    def __init__(self) -> None:
        self.tokenizer = AutoTokenizer.from_pretrained(_MODEL_NAME)
        self.model = AutoModelForSequenceClassification.from_pretrained(_MODEL_NAME)
        self.model.eval()

    def predict(self, text: str) -> dict[str, str | float]:
        inputs = self.tokenizer(
            text, return_tensors="pt", truncation=True, max_length=512, padding=True
        )
        with torch.no_grad():
            logits = self.model(**inputs).logits
        probs = torch.softmax(logits, dim=-1).squeeze()
        idx = int(probs.argmax().item())
        label: str = self.model.config.id2label[idx]
        return {"label": label, "score": round(float(probs[idx].item()), 4)}
```

- [ ] **Step 1.4: Implement main.py**

`sentiment-service/main.py`:
```python
from fastapi import FastAPI
from pydantic import BaseModel
from model import SentimentModel

app = FastAPI(title="sentiment-service")
_model = SentimentModel()


class SentimentRequest(BaseModel):
    text: str


class SentimentResponse(BaseModel):
    label: str
    score: float


@app.post("/sentiment", response_model=SentimentResponse)
def score(req: SentimentRequest) -> SentimentResponse:
    result = _model.predict(req.text)
    return SentimentResponse(label=str(result["label"]), score=float(result["score"]))


@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}
```

- [ ] **Step 1.5: Run tests — expect PASS**

```bash
cd sentiment-service && pytest tests/ -v
```

Expected:
```
tests/test_main.py::test_sentiment_returns_label_and_score PASSED
tests/test_main.py::test_sentiment_missing_text_is_422 PASSED
tests/test_main.py::test_healthz PASSED
3 passed
```

- [ ] **Step 1.6: Commit**

```bash
git add sentiment-service/
git commit -m "feat: add sentiment-service (FastAPI + FinBERT)"
```

- [ ] **Step 1.7: Create Dockerfile**

`sentiment-service/Dockerfile`:
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Pre-download model at build time so startup is instant
RUN python -c "\
from transformers import AutoTokenizer, AutoModelForSequenceClassification; \
AutoTokenizer.from_pretrained('ProsusAI/finbert'); \
AutoModelForSequenceClassification.from_pretrained('ProsusAI/finbert')"

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 1.8: Build and smoke-test**

```bash
cd sentiment-service
docker build -t sentiment-service:local .
docker run --rm -d -p 8000:8000 --name sent-test sentiment-service:local
# wait ~5s for startup
curl -s -X POST http://localhost:8000/sentiment \
  -H "Content-Type: application/json" \
  -d '{"text":"Apple beats earnings"}'
curl -s http://localhost:8000/healthz
docker stop sent-test
```

Expected first curl: `{"label":"positive","score":0.XXXX}`
Expected second curl: `{"status":"ok"}`

- [ ] **Step 1.9: Commit**

```bash
git add sentiment-service/Dockerfile
git commit -m "feat: add sentiment-service Dockerfile with pre-downloaded FinBERT"
```

---

## Task 2: stock-service (Agent: `go-backend-architect`)

**Dispatch brief for agent:** Build the Go stock-service microservice at `stock-service/`. Use `superpowers:test-driven-development`. This is the central orchestrator: it fetches from Finnhub, filters news, fans out to sentiment-service, caches results, and serves the Angular frontend. Full spec below.

**Environment variables:**

| Variable | Default | Notes |
|----------|---------|-------|
| `FINNHUB_API_KEY` | (required) | Finnhub auth token |
| `SENTIMENT_SERVICE_URL` | `http://localhost:8000` | Internal Docker URL |
| `NEWS_SOURCE_WHITELIST` | 12 built-in sources | Comma-delimited override |
| `PORT` | `8080` | Listen port |

**Finnhub endpoints used (base: `https://finnhub.io/api/v1`, auth via `?token=` param):**

| Endpoint | Params | Notes |
|----------|--------|-------|
| `GET /stock/candle` | symbol, resolution, from (unix), to (unix) | Returns `{c,h,l,o,t,v,s}` arrays |
| `GET /company-news` | symbol, from (YYYY-MM-DD), to (YYYY-MM-DD) | Returns `[]{id,headline,source,url,datetime}` |
| `GET /search` | q | Returns `{count, result:[]}` |
| `GET /stock/profile2` | symbol | Returns `{name,...}` |

**Cache TTLs:**

| Key pattern | TTL |
|-------------|-----|
| `candles:{symbol}:{resolution}:{from}:{to}` | 1 hour |
| `news:{symbol}:{from}:{to}` | 30 minutes |
| `search:{q}` | 24 hours |
| `profile:{symbol}` | 24 hours |

**Files:**
- Create: `stock-service/go.mod`
- Create: `stock-service/main.go`
- Create: `stock-service/config/config.go`
- Create: `stock-service/cache/cache.go`
- Create: `stock-service/finnhub/client.go`
- Create: `stock-service/sentiment/client.go`
- Create: `stock-service/filter/filter.go`
- Create: `stock-service/filter/filter_test.go`
- Create: `stock-service/handlers/candles.go`
- Create: `stock-service/handlers/candles_test.go`
- Create: `stock-service/handlers/news.go`
- Create: `stock-service/handlers/news_test.go`
- Create: `stock-service/handlers/search.go`
- Create: `stock-service/Dockerfile`

---

- [ ] **Step 2.1: Initialize Go module and fetch dependencies**

```bash
mkdir stock-service && cd stock-service
go mod init stock-news-sentiment/stock-service
go get github.com/patrickmn/go-cache
go get golang.org/x/time/rate
go get github.com/rs/cors
```

- [ ] **Step 2.2: Write failing filter tests**

`stock-service/filter/filter_test.go`:
```go
package filter_test

import (
	"testing"

	"stock-news-sentiment/stock-service/filter"
)

func TestBySource_keepsWhitelisted(t *testing.T) {
	arts := []filter.Article{{Source: "Reuters", Headline: "x"}, {Source: "Unknown", Headline: "y"}}
	got := filter.BySource(arts, []string{"Reuters"})
	if len(got) != 1 || got[0].Source != "Reuters" {
		t.Fatalf("expected 1 Reuters article, got %v", got)
	}
}

func TestBySource_caseInsensitive(t *testing.T) {
	arts := []filter.Article{{Source: "reuters", Headline: "x"}}
	if got := filter.BySource(arts, []string{"Reuters"}); len(got) != 1 {
		t.Fatal("expected case-insensitive match")
	}
}

func TestByRelevance_matchesTicker(t *testing.T) {
	arts := []filter.Article{
		{Source: "Reuters", Headline: "AAPL surges on iPhone sales"},
		{Source: "Reuters", Headline: "Oil prices drop"},
	}
	if got := filter.ByRelevance(arts, []string{"AAPL", "Apple"}); len(got) != 1 {
		t.Fatalf("expected 1 relevant article, got %v", got)
	}
}

func TestByRelevance_matchesCompanyName(t *testing.T) {
	arts := []filter.Article{{Source: "Reuters", Headline: "apple reports record revenue"}}
	if got := filter.ByRelevance(arts, []string{"AAPL", "Apple Inc"}); len(got) != 1 {
		t.Fatal("expected company name match")
	}
}
```

Run: `go test ./filter/...`
Expected: FAIL — `cannot find package`

- [ ] **Step 2.3: Implement filter/filter.go**

`stock-service/filter/filter.go`:
```go
package filter

import "strings"

type Article struct {
	ID        int64
	Headline  string
	Source    string
	URL       string
	Published string
}

func BySource(articles []Article, whitelist []string) []Article {
	set := make(map[string]struct{}, len(whitelist))
	for _, s := range whitelist {
		set[strings.ToLower(strings.TrimSpace(s))] = struct{}{}
	}
	out := make([]Article, 0, len(articles))
	for _, a := range articles {
		if _, ok := set[strings.ToLower(strings.TrimSpace(a.Source))]; ok {
			out = append(out, a)
		}
	}
	return out
}

func ByRelevance(articles []Article, keywords []string) []Article {
	lower := make([]string, len(keywords))
	for i, k := range keywords {
		lower[i] = strings.ToLower(k)
	}
	out := make([]Article, 0, len(articles))
	for _, a := range articles {
		h := strings.ToLower(a.Headline)
		for _, kw := range lower {
			if strings.Contains(h, kw) {
				out = append(out, a)
				break
			}
		}
	}
	return out
}
```

Run: `go test ./filter/... -v`
Expected: 4 tests PASS.

- [ ] **Step 2.4: Commit filter**

```bash
git add stock-service/
git commit -m "feat: add stock-service Go module and filter pipeline"
```

- [ ] **Step 2.5: Implement config/config.go**

`stock-service/config/config.go`:
```go
package config

import (
	"os"
	"strings"
)

var defaultSources = []string{
	"Reuters", "Associated Press", "Bloomberg", "Dow Jones",
	"The Wall Street Journal", "Financial Times", "CNBC",
	"MarketWatch", "Seeking Alpha", "Barron's",
	"Investor's Business Daily", "Yahoo Finance",
}

type Config struct {
	Port                string
	FinnhubAPIKey       string
	SentimentServiceURL string
	SourceWhitelist     []string
}

func Load() Config {
	sources := defaultSources
	if raw := os.Getenv("NEWS_SOURCE_WHITELIST"); raw != "" {
		parts := strings.Split(raw, ",")
		sources = make([]string, 0, len(parts))
		for _, p := range parts {
			if s := strings.TrimSpace(p); s != "" {
				sources = append(sources, s)
			}
		}
	}
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	sentURL := os.Getenv("SENTIMENT_SERVICE_URL")
	if sentURL == "" {
		sentURL = "http://localhost:8000"
	}
	return Config{
		Port:                port,
		FinnhubAPIKey:       os.Getenv("FINNHUB_API_KEY"),
		SentimentServiceURL: sentURL,
		SourceWhitelist:     sources,
	}
}
```

- [ ] **Step 2.6: Implement cache/cache.go**

`stock-service/cache/cache.go`:
```go
package cache

import (
	"time"

	gocache "github.com/patrickmn/go-cache"
)

const (
	TTLCandles = 1 * time.Hour
	TTLNews    = 30 * time.Minute
	TTLSearch  = 24 * time.Hour
	TTLProfile = 24 * time.Hour
)

type Cache struct {
	c *gocache.Cache
}

func New() *Cache {
	return &Cache{c: gocache.New(gocache.NoExpiration, 10*time.Minute)}
}

func (c *Cache) Set(key string, value any, ttl time.Duration) {
	c.c.Set(key, value, ttl)
}

func (c *Cache) Get(key string) (any, bool) {
	return c.c.Get(key)
}
```

- [ ] **Step 2.7: Implement finnhub/client.go**

`stock-service/finnhub/client.go`:
```go
package finnhub

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"time"

	"golang.org/x/time/rate"
)

const baseURL = "https://finnhub.io/api/v1"

type Client struct {
	apiKey  string
	http    *http.Client
	limiter *rate.Limiter
}

func NewClient(apiKey string) *Client {
	return &Client{
		apiKey:  apiKey,
		http:    &http.Client{Timeout: 15 * time.Second},
		limiter: rate.NewLimiter(rate.Every(time.Minute/60), 1),
	}
}

func (c *Client) get(ctx context.Context, path string, params url.Values, dest any) error {
	if err := c.limiter.Wait(ctx); err != nil {
		return err
	}
	params.Set("token", c.apiKey)
	req, err := http.NewRequestWithContext(ctx, http.MethodGet,
		fmt.Sprintf("%s%s?%s", baseURL, path, params.Encode()), nil)
	if err != nil {
		return err
	}
	resp, err := c.http.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("finnhub: status %d for %s", resp.StatusCode, path)
	}
	return json.NewDecoder(resp.Body).Decode(dest)
}

type CandleResponse struct {
	C      []float64 `json:"c"`
	H      []float64 `json:"h"`
	L      []float64 `json:"l"`
	O      []float64 `json:"o"`
	T      []int64   `json:"t"`
	V      []float64 `json:"v"`
	Status string    `json:"s"`
}

func (c *Client) Candles(ctx context.Context, symbol, resolution string, from, to int64) (*CandleResponse, error) {
	p := url.Values{
		"symbol": {symbol}, "resolution": {resolution},
		"from": {fmt.Sprintf("%d", from)}, "to": {fmt.Sprintf("%d", to)},
	}
	var r CandleResponse
	return &r, c.get(ctx, "/stock/candle", p, &r)
}

type NewsArticle struct {
	ID       int64  `json:"id"`
	Headline string `json:"headline"`
	Source   string `json:"source"`
	URL      string `json:"url"`
	Datetime int64  `json:"datetime"`
}

func (c *Client) CompanyNews(ctx context.Context, symbol, from, to string) ([]NewsArticle, error) {
	p := url.Values{"symbol": {symbol}, "from": {from}, "to": {to}}
	var r []NewsArticle
	return r, c.get(ctx, "/company-news", p, &r)
}

type SearchResult struct {
	Description   string `json:"description"`
	DisplaySymbol string `json:"displaySymbol"`
	Symbol        string `json:"symbol"`
	Type          string `json:"type"`
}

type SearchResponse struct {
	Count  int            `json:"count"`
	Result []SearchResult `json:"result"`
}

func (c *Client) Search(ctx context.Context, q string) (*SearchResponse, error) {
	p := url.Values{"q": {q}}
	var r SearchResponse
	return &r, c.get(ctx, "/search", p, &r)
}

type CompanyProfile struct {
	Name string `json:"name"`
}

func (c *Client) Profile(ctx context.Context, symbol string) (*CompanyProfile, error) {
	p := url.Values{"symbol": {symbol}}
	var r CompanyProfile
	return &r, c.get(ctx, "/stock/profile2", p, &r)
}
```

- [ ] **Step 2.8: Implement sentiment/client.go**

`stock-service/sentiment/client.go`:
```go
package sentiment

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

type Client struct {
	baseURL string
	http    *http.Client
}

type Response struct {
	Label string  `json:"label"`
	Score float64 `json:"score"`
}

func NewClient(baseURL string) *Client {
	return &Client{baseURL: baseURL, http: &http.Client{Timeout: 30 * time.Second}}
}

func (c *Client) Score(ctx context.Context, text string) (*Response, error) {
	body, _ := json.Marshal(map[string]string{"text": text})
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.baseURL+"/sentiment", bytes.NewReader(body))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")
	resp, err := c.http.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("sentiment service: status %d", resp.StatusCode)
	}
	var r Response
	return &r, json.NewDecoder(resp.Body).Decode(&r)
}
```

- [ ] **Step 2.9: Write failing handler tests**

`stock-service/handlers/candles_test.go`:
```go
package handlers_test

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/handlers"
)

func TestCandlesHandler_missingSymbol(t *testing.T) {
	h := handlers.NewCandlesHandler(nil, cache.New())
	req := httptest.NewRequest(http.MethodGet, "/api/candles?from=2024-01-01&to=2024-01-31", nil)
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)
	if w.Code != http.StatusBadRequest {
		t.Fatalf("expected 400, got %d", w.Code)
	}
}

func TestCandlesHandler_missingFrom(t *testing.T) {
	h := handlers.NewCandlesHandler(nil, cache.New())
	req := httptest.NewRequest(http.MethodGet, "/api/candles?symbol=AAPL&to=2024-01-31", nil)
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)
	if w.Code != http.StatusBadRequest {
		t.Fatalf("expected 400, got %d", w.Code)
	}
}
```

`stock-service/handlers/news_test.go`:
```go
package handlers_test

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/handlers"
)

func TestNewsHandler_missingSymbol(t *testing.T) {
	h := handlers.NewNewsHandler(nil, nil, cache.New(), []string{"Reuters"})
	req := httptest.NewRequest(http.MethodGet, "/api/news?from=2024-01-01&to=2024-01-31", nil)
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)
	if w.Code != http.StatusBadRequest {
		t.Fatalf("expected 400, got %d", w.Code)
	}
}

func TestNewsHandler_missingTo(t *testing.T) {
	h := handlers.NewNewsHandler(nil, nil, cache.New(), []string{"Reuters"})
	req := httptest.NewRequest(http.MethodGet, "/api/news?symbol=AAPL&from=2024-01-01", nil)
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)
	if w.Code != http.StatusBadRequest {
		t.Fatalf("expected 400, got %d", w.Code)
	}
}
```

Run: `go test ./handlers/...`
Expected: FAIL — `cannot find package`

- [ ] **Step 2.10: Implement handlers/candles.go**

`stock-service/handlers/candles.go`:
```go
package handlers

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/finnhub"
)

type Candle struct {
	T int64   `json:"t"`
	O float64 `json:"o"`
	H float64 `json:"h"`
	L float64 `json:"l"`
	C float64 `json:"c"`
	V float64 `json:"v"`
}

type CandlesResponse struct {
	Symbol     string   `json:"symbol"`
	Resolution string   `json:"resolution"`
	Candles    []Candle `json:"candles"`
}

type CandlesHandler struct {
	finnhub *finnhub.Client
	cache   *cache.Cache
}

func NewCandlesHandler(f *finnhub.Client, c *cache.Cache) *CandlesHandler {
	return &CandlesHandler{finnhub: f, cache: c}
}

func (h *CandlesHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	symbol, from, to := q.Get("symbol"), q.Get("from"), q.Get("to")
	resolution := q.Get("resolution")
	if resolution == "" {
		resolution = "D"
	}
	if symbol == "" || from == "" || to == "" {
		http.Error(w, "symbol, from, and to are required", http.StatusBadRequest)
		return
	}

	key := fmt.Sprintf("candles:%s:%s:%s:%s", symbol, resolution, from, to)
	if v, ok := h.cache.Get(key); ok {
		writeJSON(w, v)
		return
	}

	fromT, err := time.Parse("2006-01-02", from)
	if err != nil {
		http.Error(w, "invalid from date (YYYY-MM-DD)", http.StatusBadRequest)
		return
	}
	toT, err := time.Parse("2006-01-02", to)
	if err != nil {
		http.Error(w, "invalid to date (YYYY-MM-DD)", http.StatusBadRequest)
		return
	}

	raw, err := h.finnhub.Candles(r.Context(), symbol, resolution, fromT.Unix(), toT.Unix())
	if err != nil {
		http.Error(w, "upstream error", http.StatusBadGateway)
		return
	}

	candles := make([]Candle, len(raw.T))
	for i := range raw.T {
		candles[i] = Candle{T: raw.T[i], O: raw.O[i], H: raw.H[i], L: raw.L[i], C: raw.C[i], V: raw.V[i]}
	}
	resp := CandlesResponse{Symbol: symbol, Resolution: resolution, Candles: candles}
	h.cache.Set(key, resp, cache.TTLCandles)
	writeJSON(w, resp)
}

func writeJSON(w http.ResponseWriter, v any) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(v)
}
```

- [ ] **Step 2.11: Implement handlers/search.go**

`stock-service/handlers/search.go`:
```go
package handlers

import (
	"fmt"
	"net/http"

	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/finnhub"
)

type SearchHandler struct {
	finnhub *finnhub.Client
	cache   *cache.Cache
}

func NewSearchHandler(f *finnhub.Client, c *cache.Cache) *SearchHandler {
	return &SearchHandler{finnhub: f, cache: c}
}

func (h *SearchHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query().Get("q")
	if q == "" {
		http.Error(w, "q is required", http.StatusBadRequest)
		return
	}
	key := fmt.Sprintf("search:%s", q)
	if v, ok := h.cache.Get(key); ok {
		writeJSON(w, v)
		return
	}
	result, err := h.finnhub.Search(r.Context(), q)
	if err != nil {
		http.Error(w, "upstream error", http.StatusBadGateway)
		return
	}
	h.cache.Set(key, result, cache.TTLSearch)
	writeJSON(w, result)
}
```

- [ ] **Step 2.12: Implement handlers/news.go**

`stock-service/handlers/news.go`:
```go
package handlers

import (
	"fmt"
	"net/http"
	"sync"
	"time"

	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/filter"
	"stock-news-sentiment/stock-service/finnhub"
	"stock-news-sentiment/stock-service/sentiment"
)

type Sentiment struct {
	Label string  `json:"label"`
	Score float64 `json:"score"`
}

type EnrichedArticle struct {
	ID          int64     `json:"id"`
	Headline    string    `json:"headline"`
	Source      string    `json:"source"`
	URL         string    `json:"url"`
	PublishedAt string    `json:"publishedAt"`
	Sentiment   Sentiment `json:"sentiment"`
}

type NewsResponse struct {
	Symbol        string            `json:"symbol"`
	TotalFetched  int               `json:"totalFetched"`
	TotalFiltered int               `json:"totalFiltered"`
	Articles      []EnrichedArticle `json:"articles"`
}

type NewsHandler struct {
	finnhub   *finnhub.Client
	sentiment *sentiment.Client
	cache     *cache.Cache
	whitelist []string
}

func NewNewsHandler(f *finnhub.Client, s *sentiment.Client, c *cache.Cache, whitelist []string) *NewsHandler {
	return &NewsHandler{finnhub: f, sentiment: s, cache: c, whitelist: whitelist}
}

func (h *NewsHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	symbol, from, to := q.Get("symbol"), q.Get("from"), q.Get("to")
	if symbol == "" || from == "" || to == "" {
		http.Error(w, "symbol, from, and to are required", http.StatusBadRequest)
		return
	}

	key := fmt.Sprintf("news:%s:%s:%s", symbol, from, to)
	if v, ok := h.cache.Get(key); ok {
		writeJSON(w, v)
		return
	}

	// Company profile for relevance keywords
	profileKey := fmt.Sprintf("profile:%s", symbol)
	keywords := []string{symbol}
	if v, ok := h.cache.Get(profileKey); ok {
		if name, _ := v.(string); name != "" {
			keywords = append(keywords, name)
		}
	} else if profile, err := h.finnhub.Profile(r.Context(), symbol); err == nil && profile.Name != "" {
		h.cache.Set(profileKey, profile.Name, cache.TTLProfile)
		keywords = append(keywords, profile.Name)
	}

	raw, err := h.finnhub.CompanyNews(r.Context(), symbol, from, to)
	if err != nil {
		http.Error(w, "upstream error", http.StatusBadGateway)
		return
	}
	totalFetched := len(raw)

	// Convert and apply two-pass filter
	arts := make([]filter.Article, len(raw))
	for i, a := range raw {
		arts[i] = filter.Article{
			ID:        a.ID,
			Headline:  a.Headline,
			Source:    a.Source,
			URL:       a.URL,
			Published: time.Unix(a.Datetime, 0).UTC().Format(time.RFC3339),
		}
	}
	filtered := filter.ByRelevance(filter.BySource(arts, h.whitelist), keywords)

	// Concurrent sentiment scoring — semaphore capped at 20
	sem := make(chan struct{}, 20)
	enriched := make([]EnrichedArticle, len(filtered))
	var wg sync.WaitGroup
	var mu sync.Mutex
	var sentErr error

	for i, a := range filtered {
		wg.Add(1)
		go func(idx int, art filter.Article) {
			defer wg.Done()
			sem <- struct{}{}
			defer func() { <-sem }()

			result, err := h.sentiment.Score(r.Context(), art.Headline)
			mu.Lock()
			defer mu.Unlock()
			if err != nil {
				sentErr = err
				return
			}
			enriched[idx] = EnrichedArticle{
				ID:          art.ID,
				Headline:    art.Headline,
				Source:      art.Source,
				URL:         art.URL,
				PublishedAt: art.Published,
				Sentiment:   Sentiment{Label: result.Label, Score: result.Score},
			}
		}(i, a)
	}
	wg.Wait()

	if sentErr != nil {
		http.Error(w, "sentiment service unavailable", http.StatusBadGateway)
		return
	}

	resp := NewsResponse{
		Symbol:        symbol,
		TotalFetched:  totalFetched,
		TotalFiltered: len(filtered),
		Articles:      enriched,
	}
	h.cache.Set(key, resp, cache.TTLNews)
	writeJSON(w, resp)
}
```

- [ ] **Step 2.13: Implement main.go**

`stock-service/main.go`:
```go
package main

import (
	"fmt"
	"net/http"

	"github.com/rs/cors"
	"stock-news-sentiment/stock-service/cache"
	"stock-news-sentiment/stock-service/config"
	"stock-news-sentiment/stock-service/finnhub"
	"stock-news-sentiment/stock-service/handlers"
	"stock-news-sentiment/stock-service/sentiment"
)

func main() {
	cfg := config.Load()
	c := cache.New()
	finn := finnhub.NewClient(cfg.FinnhubAPIKey)
	sent := sentiment.NewClient(cfg.SentimentServiceURL)

	mux := http.NewServeMux()
	mux.Handle("/api/candles", handlers.NewCandlesHandler(finn, c))
	mux.Handle("/api/news", handlers.NewNewsHandler(finn, sent, c, cfg.SourceWhitelist))
	mux.Handle("/api/search", handlers.NewSearchHandler(finn, c))
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, `{"status":"ok"}`)
	})

	fmt.Printf("stock-service listening on :%s\n", cfg.Port)
	http.ListenAndServe(":"+cfg.Port, cors.Default().Handler(mux))
}
```

- [ ] **Step 2.14: Run all tests and verify build**

```bash
cd stock-service
go test ./... -v
go build ./...
```

Expected: all tests PASS, build succeeds with no errors.

- [ ] **Step 2.15: Commit**

```bash
git add stock-service/
git commit -m "feat: add stock-service with candles, news, search, filter, and cache"
```

- [ ] **Step 2.16: Create Dockerfile**

`stock-service/Dockerfile`:
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o stock-service .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/stock-service .
EXPOSE 8080
CMD ["./stock-service"]
```

- [ ] **Step 2.17: Build Docker image**

```bash
cd stock-service && docker build -t stock-service:local .
```

Expected: Build succeeds.

- [ ] **Step 2.18: Commit**

```bash
git add stock-service/Dockerfile
git commit -m "feat: add stock-service multi-stage Dockerfile"
```

---

## Task 3: Frontend (Agent: `angular-frontend-dev`)

**Dispatch brief for agent:** Build the Angular 17 SPA at `frontend/`. Use `superpowers:test-driven-development`. Install TradingView Lightweight Charts v4. The app calls `/api/candles`, `/api/news`, `/api/search` — these are proxied to stock-service via nginx in production. Use standalone components throughout (no NgModules). Full spec below.

**Backend API contracts:**
```
GET /api/candles?symbol=AAPL&from=2024-01-01&to=2024-01-31&resolution=D
→ { symbol: string, resolution: string, candles: [{t,o,h,l,c,v}] }

GET /api/news?symbol=AAPL&from=2024-01-01&to=2024-01-31
→ { symbol, totalFetched, totalFiltered, articles: [{id, headline, source, url, publishedAt, sentiment:{label,score}}] }

GET /api/search?q=apple
→ { count, result: [{description, displaySymbol, symbol, type}] }
```

**Sentiment labels:** `'positive' | 'negative' | 'neutral'`
**Sentiment colors:** positive=#22c55e, negative=#ef4444, neutral=#9ca3af

**Files:**
- Create: `frontend/` (via `ng new`)
- Create: `frontend/src/app/models/candle.model.ts`
- Create: `frontend/src/app/models/news.model.ts`
- Create: `frontend/src/app/services/app.service.ts`
- Create: `frontend/src/app/services/app.service.spec.ts`
- Create: `frontend/src/app/components/search-bar/search-bar.component.ts` + spec
- Create: `frontend/src/app/components/date-range-picker/date-range-picker.component.ts` + spec
- Create: `frontend/src/app/components/stock-chart/stock-chart.component.ts` + spec
- Create: `frontend/src/app/components/news-marker/news-marker.component.ts` + spec
- Create: `frontend/src/app/components/article-panel/article-panel.component.ts` + spec
- Create: `frontend/src/app/components/sentiment-filter/sentiment-filter.component.ts` + spec
- Create: `frontend/src/app/components/filter-count/filter-count.component.ts` + spec
- Create: `frontend/src/app/app.component.ts`
- Create: `frontend/nginx.conf`
- Create: `frontend/Dockerfile`

---

- [ ] **Step 3.1: Scaffold Angular project and install dependencies**

```bash
npx @angular/cli@17 new frontend --routing=false --style=scss --strict --standalone
cd frontend
npm install lightweight-charts@4
```

- [ ] **Step 3.2: Create TypeScript models**

`frontend/src/app/models/candle.model.ts`:
```typescript
export interface Candle {
  t: number;
  o: number;
  h: number;
  l: number;
  c: number;
  v: number;
}

export interface CandlesResponse {
  symbol: string;
  resolution: string;
  candles: Candle[];
}
```

`frontend/src/app/models/news.model.ts`:
```typescript
export type SentimentLabel = 'positive' | 'negative' | 'neutral';

export interface Sentiment {
  label: SentimentLabel;
  score: number;
}

export interface Article {
  id: number;
  headline: string;
  source: string;
  url: string;
  publishedAt: string;
  sentiment: Sentiment;
}

export interface NewsResponse {
  symbol: string;
  totalFetched: number;
  totalFiltered: number;
  articles: Article[];
}

export interface SearchResult {
  description: string;
  displaySymbol: string;
  symbol: string;
  type: string;
}

export interface SearchResponse {
  count: number;
  result: SearchResult[];
}
```

- [ ] **Step 3.3: Write failing AppService tests**

`frontend/src/app/services/app.service.spec.ts`:
```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { AppService } from './app.service';

describe('AppService', () => {
  let service: AppService;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ imports: [HttpClientTestingModule], providers: [AppService] });
    service = TestBed.inject(AppService);
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());

  it('getCandles passes symbol, from, to, resolution=D', () => {
    service.getCandles('AAPL', '2024-01-01', '2024-01-31').subscribe();
    const req = http.expectOne(r => r.url === '/api/candles');
    expect(req.request.params.get('symbol')).toBe('AAPL');
    expect(req.request.params.get('resolution')).toBe('D');
    req.flush({ symbol: 'AAPL', resolution: 'D', candles: [] });
  });

  it('getNews passes symbol, from, to', () => {
    service.getNews('AAPL', '2024-01-01', '2024-01-31').subscribe();
    const req = http.expectOne(r => r.url === '/api/news');
    expect(req.request.params.get('symbol')).toBe('AAPL');
    req.flush({ symbol: 'AAPL', totalFetched: 0, totalFiltered: 0, articles: [] });
  });

  it('searchTickers passes q', () => {
    service.searchTickers('apple').subscribe();
    const req = http.expectOne(r => r.url === '/api/search');
    expect(req.request.params.get('q')).toBe('apple');
    req.flush({ count: 0, result: [] });
  });
});
```

Run: `ng test --watch=false`
Expected: FAIL — `AppService not found`

- [ ] **Step 3.4: Implement AppService**

`frontend/src/app/services/app.service.ts`:
```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { CandlesResponse } from '../models/candle.model';
import { NewsResponse, SearchResponse } from '../models/news.model';

@Injectable({ providedIn: 'root' })
export class AppService {
  constructor(private http: HttpClient) {}

  getCandles(symbol: string, from: string, to: string, resolution = 'D'): Observable<CandlesResponse> {
    return this.http.get<CandlesResponse>('/api/candles', {
      params: new HttpParams().set('symbol', symbol).set('from', from).set('to', to).set('resolution', resolution),
    });
  }

  getNews(symbol: string, from: string, to: string): Observable<NewsResponse> {
    return this.http.get<NewsResponse>('/api/news', {
      params: new HttpParams().set('symbol', symbol).set('from', from).set('to', to),
    });
  }

  searchTickers(q: string): Observable<SearchResponse> {
    return this.http.get<SearchResponse>('/api/search', {
      params: new HttpParams().set('q', q),
    });
  }
}
```

Run: `ng test --watch=false`
Expected: 3 tests PASS.

- [ ] **Step 3.5: Implement StockChartComponent**

`frontend/src/app/components/stock-chart/stock-chart.component.ts`:
```typescript
import {
  Component, ElementRef, Input, OnChanges, OnDestroy,
  AfterViewInit, SimpleChanges, ViewChild,
} from '@angular/core';
import { createChart, IChartApi, ISeriesApi, CandlestickData, Time } from 'lightweight-charts';
import { Candle } from '../../models/candle.model';

@Component({
  selector: 'app-stock-chart',
  standalone: true,
  template: `<div #chartEl style="width:100%;height:500px;"></div>`,
})
export class StockChartComponent implements AfterViewInit, OnChanges, OnDestroy {
  @ViewChild('chartEl') chartEl!: ElementRef<HTMLDivElement>;
  @Input() candles: Candle[] = [];

  private chart!: IChartApi;
  private series!: ISeriesApi<'Candlestick'>;

  ngAfterViewInit(): void {
    this.chart = createChart(this.chartEl.nativeElement, { height: 500 });
    this.series = this.chart.addCandlestickSeries();
    this.render();
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (this.series && changes['candles']) this.render();
  }

  ngOnDestroy(): void { this.chart?.remove(); }

  private render(): void {
    this.series.setData(
      this.candles.map(c => ({ time: c.t as Time, open: c.o, high: c.h, low: c.l, close: c.c }))
    );
    this.chart.timeScale().fitContent();
  }
}
```

- [ ] **Step 3.6: Implement SearchBarComponent**

`frontend/src/app/components/search-bar/search-bar.component.ts`:
```typescript
import { Component, EventEmitter, Output } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Subject } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';
import { AppService } from '../../services/app.service';
import { SearchResult } from '../../models/news.model';

@Component({
  selector: 'app-search-bar',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div>
      <input type="text" [(ngModel)]="query" (ngModelChange)="input$.next($event)"
             placeholder="Search ticker (e.g. AAPL)" />
      <ul *ngIf="results.length">
        <li *ngFor="let r of results" (click)="select(r)">
          <strong>{{ r.symbol }}</strong> — {{ r.description }}
        </li>
      </ul>
    </div>
  `,
})
export class SearchBarComponent {
  @Output() symbolSelected = new EventEmitter<string>();
  query = '';
  results: SearchResult[] = [];
  readonly input$ = new Subject<string>();

  constructor(svc: AppService) {
    this.input$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => svc.searchTickers(q)),
    ).subscribe(r => { this.results = r.result.slice(0, 8); });
  }

  select(r: SearchResult): void {
    this.query = r.symbol;
    this.results = [];
    this.symbolSelected.emit(r.symbol);
  }
}
```

- [ ] **Step 3.7: Implement DateRangePickerComponent**

`frontend/src/app/components/date-range-picker/date-range-picker.component.ts`:
```typescript
import { Component, EventEmitter, OnInit, Output } from '@angular/core';
import { FormsModule } from '@angular/forms';

export interface DateRange { from: string; to: string; }

@Component({
  selector: 'app-date-range-picker',
  standalone: true,
  imports: [FormsModule],
  template: `
    <label>From <input type="date" [(ngModel)]="from" (change)="emit()" /></label>
    <label>To <input type="date" [(ngModel)]="to" (change)="emit()" /></label>
  `,
})
export class DateRangePickerComponent implements OnInit {
  @Output() rangeChanged = new EventEmitter<DateRange>();
  from = '';
  to = '';

  ngOnInit(): void {
    const now = new Date();
    this.to = now.toISOString().split('T')[0];
    const ago = new Date(now);
    ago.setMonth(ago.getMonth() - 6);
    this.from = ago.toISOString().split('T')[0];
  }

  emit(): void { this.rangeChanged.emit({ from: this.from, to: this.to }); }
}
```

- [ ] **Step 3.8: Implement NewsMarkerComponent**

`frontend/src/app/components/news-marker/news-marker.component.ts`:
```typescript
import { Component, EventEmitter, Input, Output } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Article, SentimentLabel } from '../../models/news.model';

const COLOR: Record<SentimentLabel, string> = {
  positive: '#22c55e', negative: '#ef4444', neutral: '#9ca3af',
};

@Component({
  selector: 'app-news-marker',
  standalone: true,
  imports: [CommonModule],
  styles: [`
    .marker { display:inline-block; width:12px; height:12px; border-radius:50%;
              cursor:pointer; margin:0 2px; }
  `],
  template: `
    <div>
      <span *ngFor="let a of articles" class="marker"
            [style.background]="color(a.sentiment.label)"
            [title]="a.headline"
            (click)="markerClicked.emit(a)"></span>
    </div>
  `,
})
export class NewsMarkerComponent {
  @Input() articles: Article[] = [];
  @Output() markerClicked = new EventEmitter<Article>();
  color(label: SentimentLabel): string { return COLOR[label]; }
}
```

- [ ] **Step 3.9: Implement ArticlePanelComponent**

`frontend/src/app/components/article-panel/article-panel.component.ts`:
```typescript
import { Component, Input } from '@angular/core';
import { CommonModule, DatePipe, DecimalPipe } from '@angular/common';
import { Article } from '../../models/news.model';

@Component({
  selector: 'app-article-panel',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="article">
      <span [class]="'badge ' + article.sentiment.label">
        {{ article.sentiment.label }} ({{ article.sentiment.score | number:'1.2-2' }})
      </span>
      <h3><a [href]="article.url" target="_blank" rel="noopener">{{ article.headline }}</a></h3>
      <p>{{ article.source }} · {{ article.publishedAt | date:'mediumDate' }}</p>
    </div>
  `,
})
export class ArticlePanelComponent {
  @Input() article: Article | null = null;
}
```

- [ ] **Step 3.10: Implement SentimentFilterComponent**

`frontend/src/app/components/sentiment-filter/sentiment-filter.component.ts`:
```typescript
import { Component, EventEmitter, Output } from '@angular/core';
import { CommonModule } from '@angular/common';
import { SentimentLabel } from '../../models/news.model';

@Component({
  selector: 'app-sentiment-filter',
  standalone: true,
  imports: [CommonModule],
  template: `
    <label *ngFor="let l of labels">
      <input type="checkbox" [checked]="active.has(l)" (change)="toggle(l)" /> {{ l }}
    </label>
  `,
})
export class SentimentFilterComponent {
  @Output() filterChanged = new EventEmitter<Set<SentimentLabel>>();
  labels: SentimentLabel[] = ['positive', 'negative', 'neutral'];
  active = new Set<SentimentLabel>(['positive', 'negative', 'neutral']);

  toggle(l: SentimentLabel): void {
    this.active.has(l) ? this.active.delete(l) : this.active.add(l);
    this.filterChanged.emit(new Set(this.active));
  }
}
```

- [ ] **Step 3.11: Implement FilterCountComponent**

`frontend/src/app/components/filter-count/filter-count.component.ts`:
```typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-filter-count',
  standalone: true,
  template: `<p>{{ shown }} shown, {{ hidden }} hidden ({{ totalFetched }} total fetched from API)</p>`,
})
export class FilterCountComponent {
  @Input() shown = 0;
  @Input() hidden = 0;
  @Input() totalFetched = 0;
}
```

- [ ] **Step 3.12: Implement AppComponent**

`frontend/src/app/app.component.ts`:
```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { forkJoin } from 'rxjs';
import { AppService } from './services/app.service';
import { Candle } from './models/candle.model';
import { Article, SentimentLabel } from './models/news.model';
import { SearchBarComponent } from './components/search-bar/search-bar.component';
import { DateRangePickerComponent, DateRange } from './components/date-range-picker/date-range-picker.component';
import { StockChartComponent } from './components/stock-chart/stock-chart.component';
import { NewsMarkerComponent } from './components/news-marker/news-marker.component';
import { ArticlePanelComponent } from './components/article-panel/article-panel.component';
import { SentimentFilterComponent } from './components/sentiment-filter/sentiment-filter.component';
import { FilterCountComponent } from './components/filter-count/filter-count.component';
import { HttpClientModule } from '@angular/common/http';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [
    CommonModule, HttpClientModule,
    SearchBarComponent, DateRangePickerComponent, StockChartComponent,
    NewsMarkerComponent, ArticlePanelComponent, SentimentFilterComponent, FilterCountComponent,
  ],
  template: `
    <div class="app">
      <header>
        <app-search-bar (symbolSelected)="onSymbol($event)" />
        <app-date-range-picker (rangeChanged)="onRange($event)" />
      </header>
      <app-stock-chart [candles]="candles" />
      <app-news-marker [articles]="visible" (markerClicked)="selected = $event" />
      <app-sentiment-filter (filterChanged)="onFilter($event)" />
      <app-filter-count [shown]="visible.length"
                        [hidden]="articles.length - visible.length"
                        [totalFetched]="totalFetched" />
      <app-article-panel [article]="selected" />
    </div>
  `,
})
export class AppComponent implements OnInit {
  symbol = 'AAPL';
  from = '';
  to = '';
  candles: Candle[] = [];
  articles: Article[] = [];
  visible: Article[] = [];
  selected: Article | null = null;
  totalFetched = 0;
  private activeFilters = new Set<SentimentLabel>(['positive', 'negative', 'neutral']);

  constructor(private svc: AppService) {}

  ngOnInit(): void {
    const now = new Date();
    this.to = now.toISOString().split('T')[0];
    const ago = new Date(now); ago.setMonth(ago.getMonth() - 6);
    this.from = ago.toISOString().split('T')[0];
    this.load();
  }

  onSymbol(s: string): void { this.symbol = s; this.load(); }
  onRange(r: DateRange): void { this.from = r.from; this.to = r.to; this.load(); }
  onFilter(f: Set<SentimentLabel>): void { this.activeFilters = f; this.applyFilter(); }

  private load(): void {
    forkJoin({
      candles: this.svc.getCandles(this.symbol, this.from, this.to),
      news: this.svc.getNews(this.symbol, this.from, this.to),
    }).subscribe(({ candles, news }) => {
      this.candles = candles.candles;
      this.articles = news.articles;
      this.totalFetched = news.totalFetched;
      this.applyFilter();
    });
  }

  private applyFilter(): void {
    this.visible = this.articles.filter(a => this.activeFilters.has(a.sentiment.label));
  }
}
```

- [ ] **Step 3.13: Run all Angular tests**

```bash
cd frontend && ng test --watch=false
```

Expected: All tests PASS (AppService spec + any component specs).

- [ ] **Step 3.14: Commit**

```bash
git add frontend/src/
git commit -m "feat: add Angular frontend components, service, and models"
```

- [ ] **Step 3.15: Create nginx.conf**

`frontend/nginx.conf`:
```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://stock-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- [ ] **Step 3.16: Create Dockerfile**

`frontend/Dockerfile`:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist/frontend/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

- [ ] **Step 3.17: Build Docker image**

```bash
cd frontend && docker build -t frontend:local .
```

Expected: Build succeeds.

- [ ] **Step 3.18: Commit**

```bash
git add frontend/nginx.conf frontend/Dockerfile
git commit -m "feat: add frontend nginx config and Dockerfile"
```

---

## Task 4: Docker Compose (Agent: `cicd-docker-engineer`)

**Dispatch brief for agent:** Wire the 3 microservices with Docker Compose for the stock-news-sentiment-analysis project. sentiment-service must be healthy before stock-service starts. Only port 4200 is exposed to the host. Single Docker bridge network. FINNHUB_API_KEY comes from `.env`.

**Files:**
- Create: `docker-compose.yml`
- Create: `.env.example`

---

- [ ] **Step 4.1: Create .env.example**

`.env.example`:
```dotenv
FINNHUB_API_KEY=your_finnhub_key_here
# Optional — defaults shown:
# NEWS_SOURCE_WHITELIST=Reuters,Associated Press,Bloomberg,Dow Jones,The Wall Street Journal,Financial Times,CNBC,MarketWatch,Seeking Alpha,Barron's,Investor's Business Daily,Yahoo Finance
```

- [ ] **Step 4.2: Create docker-compose.yml**

`docker-compose.yml`:
```yaml
version: '3.8'

services:
  sentiment-service:
    build: ./sentiment-service
    networks: [backend]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  stock-service:
    build: ./stock-service
    networks: [backend]
    depends_on:
      sentiment-service:
        condition: service_healthy
    environment:
      - FINNHUB_API_KEY=${FINNHUB_API_KEY}
      - SENTIMENT_SERVICE_URL=http://sentiment-service:8000
      - NEWS_SOURCE_WHITELIST=${NEWS_SOURCE_WHITELIST:-}

  frontend:
    build: ./frontend
    ports:
      - "4200:80"
    networks: [backend]
    depends_on:
      - stock-service

networks:
  backend:
    driver: bridge
```

- [ ] **Step 4.3: Smoke test end-to-end**

```bash
cp .env.example .env
# Set FINNHUB_API_KEY in .env to a real key
docker compose up --build -d
docker compose ps
```

Expected: All 3 containers `running`; `sentiment-service` shows `healthy`.

```bash
curl -s http://localhost:4200/api/candles?symbol=AAPL&from=2024-01-01&to=2024-01-31
```

Expected: `{"symbol":"AAPL","resolution":"D","candles":[...]}`

```bash
curl -s "http://localhost:4200/api/news?symbol=AAPL&from=2024-01-01&to=2024-01-31" | head -c 500
```

Expected: JSON with `articles` array, each article has `sentiment.label` and `sentiment.score`.

- [ ] **Step 4.4: Commit**

```bash
git add docker-compose.yml .env.example
git commit -m "feat: add Docker Compose for single-command deployment"
```

---

## Verification Checklist

Run after all 4 tasks complete:

1. `docker compose up --build` — no build errors, all 3 containers start
2. `docker compose ps` — all show `running`; `sentiment-service` shows `healthy`
3. Open `http://localhost:4200` — Angular app loads, AAPL chart visible
4. Type "Microsoft" in search bar — autocomplete shows MSFT
5. Select MSFT — chart and news update
6. Change date range — both candles and news refresh
7. News markers appear on the timeline, color-coded (green/red/grey)
8. Click a marker — ArticlePanel shows headline, source, date, sentiment badge
9. Uncheck "positive" in sentiment filter — green markers disappear; count updates
10. `docker compose down` — all containers stop cleanly

---

## Spec Coverage Checklist

| Spec Requirement | Task | Location |
|-----------------|------|----------|
| `GET /api/candles` | 2 | `handlers/candles.go` |
| `GET /api/news` with two-pass filter | 2 | `handlers/news.go`, `filter/filter.go` |
| `GET /api/search` | 2 | `handlers/search.go` |
| `/healthz` on both services | 1, 2 | `main.py`, `main.go` |
| Source whitelist (pass 1) | 2 | `filter.BySource` |
| Company relevance (pass 2) | 2 | `filter.ByRelevance` |
| Concurrent sentiment — max 20 goroutines | 2 | `handlers/news.go` semaphore |
| Sentiment-service unavailable → HTTP 502 | 2 | `handlers/news.go` |
| go-cache with correct TTLs | 2 | `cache/cache.go` |
| Finnhub rate limit 60 req/min | 2 | `finnhub/client.go` token bucket |
| `NEWS_SOURCE_WHITELIST` env var | 2 | `config/config.go` |
| FinBERT inference | 1 | `model.py` |
| Model pre-downloaded in image | 1 | `Dockerfile` |
| TradingView candlestick chart | 3 | `StockChartComponent` |
| News markers color-coded by sentiment | 3 | `NewsMarkerComponent` |
| Click marker → ArticlePanel | 3 | `AppComponent` + `ArticlePanelComponent` |
| Sentiment toggle filter | 3 | `SentimentFilterComponent` |
| "X shown, Y filtered" transparency | 3 | `FilterCountComponent` |
| Ticker search autocomplete | 3 | `SearchBarComponent` |
| nginx proxies `/api/` → stock-service | 3 | `nginx.conf` |
| Docker Compose single-command | 4 | `docker-compose.yml` |
| `depends_on` with health check | 4 | `docker-compose.yml` |
| Only port 4200 exposed | 4 | `docker-compose.yml` |
| `.env.example` | 4 | `.env.example` |
