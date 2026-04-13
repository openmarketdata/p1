# Architecture Document: Sports Arbitrage Dashboard

## 1. Overview

This application compares **Polymarket sports prediction-market probabilities** with **DraftKings sportsbook betting odds** to identify arbitrage and hedging opportunities. It surfaces overlapping events side-by-side, computes the optimal DraftKings bet size to break even on a short-sale of 10 YES contracts on Polymarket, and ranks opportunities by residual profit. The entire data pipeline refreshes **6 times per minute** (every 10 seconds).

---

## 2. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              WEB BROWSER                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    React / Next.js Frontend                           │  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────┐ │  │
│  │  │  Polymarket   │  │  DraftKings  │  │    Merged Opportunity       │ │  │
│  │  │  Panel        │  │  Panel       │  │    Table (ranked by         │ │  │
│  │  │              │  │              │  │    residual, color-coded)   │ │  │
│  │  └──────┬───────┘  └──────┬───────┘  └─────────────┬───────────────┘ │  │
│  │         │                 │                         │                 │  │
│  │         └─────────┬───────┘                         │                 │  │
│  │                   ▼                                 │                 │  │
│  │         Polling Timer (every 10s)                   │                 │  │
│  └───────────────────┬─────────────────────────────────┘                 │  │
│                      │                                                    │
└──────────────────────┼────────────────────────────────────────────────────┘
                       │  HTTP / WebSocket
                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         BACKEND API SERVER (Node.js / Fastify)               │
│                                                                              │
│  ┌─────────────────────────┐      ┌────────────────────────────────────┐    │
│  │  Polymarket Data Module  │      │  DraftKings / Odds Data Module     │    │
│  │                          │      │                                    │    │
│  │  • Gamma API poller      │      │  Primary: DK unofficial endpoints  │    │
│  │  • CLOB orderbook reader │      │  Fallback: The Odds API aggregator │    │
│  │  • WebSocket subscriber  │      │  • Poller (10s interval)           │    │
│  │  • Sports API metadata   │      │  • Normalizer                      │    │
│  └────────────┬─────────────┘      └──────────────┬─────────────────────┘    │
│               │                                   │                          │
│               ▼                                   ▼                          │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    In-Memory Cache (Redis or Map)                     │    │
│  │                                                                      │    │
│  │  polymarket_events: { eventId → { slug, teams, P_yes, P_no, ... } }  │    │
│  │  draftkings_odds:   { eventKey → { teams, decimalOdds, american } }  │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│               │                                                              │
│               ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                      Event Matching Engine                            │    │
│  │                                                                      │    │
│  │  1. Normalize team/event names (fuzzy match)                          │    │
│  │  2. Match by sport + teams + date                                     │    │
│  │  3. Produce merged overlay: PM prob + DK odds + hedge calculation     │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│               │                                                              │
│               ▼                                                              │
│         GET /api/opportunities   (JSON)                                      │
│         WS  /ws/opportunities    (push on every cycle)                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       EXTERNAL DATA SOURCES                                  │
│                                                                              │
│  ┌─────────────────────────┐      ┌────────────────────────────────────┐    │
│  │  Polymarket APIs         │      │  DraftKings Data Sources           │    │
│  │                          │      │                                    │    │
│  │  Gamma (market data):    │      │  Primary — DK Unofficial REST:     │    │
│  │  gamma-api.polymarket.com│      │  api.draftkings.com                │    │
│  │                          │      │                                    │    │
│  │  CLOB (orderbook/trade): │      │  Fallback — The Odds API:          │    │
│  │  clob.polymarket.com     │      │  api.the-odds-api.com/v4           │    │
│  │                          │      │                                    │    │
│  │  Sports (US):            │      │                                    │    │
│  │  docs.polymarket.us      │      │                                    │    │
│  └─────────────────────────┘      └────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Module 1 — Polymarket Data Service

### 3.1 Purpose

Continuously fetch all **sports** prediction-market data from Polymarket, including event metadata, current YES/NO prices, and orderbook depth.

### 3.2 Data Sources & Endpoints

| API Layer | Base URL | Key Endpoints | Auth |
|-----------|----------|---------------|------|
| **Gamma API** (discovery) | `https://gamma-api.polymarket.com` | `GET /events?tag=sports` — list all sports events | None |
| | | `GET /events/{id}` — single event with markets | None |
| | | `GET /markets?eventId={id}` — markets for event | None |
| | | `GET /markets/{id}` — single market detail | None |
| **CLOB API** (orderbook) | `https://clob.polymarket.com` | `GET /order-book/{token_id}` — bids/asks | None (read) |
| | | `GET /trades?market={id}` — recent trades | None |
| **Sports API** (US) | `https://gateway.polymarket.us` | `GET /v2/sports/{slug}/events` — by sport | None |
| | | `GET /v2/leagues/{slug}/events` — by league (nfl, nba…) | None |
| **WebSocket** | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | Market channel — live price updates | None |

### 3.3 Data Model — Polymarket Event

```typescript
interface PolymarketEvent {
  id: string;                  // Gamma event ID
  slug: string;                // URL-friendly name
  title: string;               // e.g. "Will the Lakers win Game 7?"
  sport: string;               // "basketball", "football", etc.
  league: string;              // "nba", "nfl", etc.
  startDate: string;           // ISO 8601
  homeTeam: string;
  awayTeam: string;
  markets: PolymarketMarket[];
}

interface PolymarketMarket {
  id: string;                  // Market ID
  question: string;            // Market question text
  outcomes: string[];          // ["Yes", "No"]
  outcomePrices: number[];     // [0.72, 0.28]  (YES price, NO price)
  tokenIds: string[];          // CLOB token IDs for YES/NO
  volume: number;              // Total traded volume in USDC
  liquidity: number;           // Current liquidity
  active: boolean;
}
```

### 3.4 Polling & Subscription Strategy

```
┌───────────────────────────────────────────────────────┐
│              Polymarket Data Service                    │
│                                                        │
│   On startup:                                          │
│   1. GET /events?tag=sports → full event list          │
│   2. For each event: GET /markets?eventId=X            │
│   3. Open WebSocket → subscribe to all token_ids       │
│                                                        │
│   Every 10 seconds (polling fallback):                 │
│   1. GET /events?tag=sports&active=true                │
│   2. For new/changed events: refresh market data       │
│   3. Update in-memory cache                            │
│                                                        │
│   WebSocket on-message:                                │
│   1. Parse price_change event                          │
│   2. Update cache entry for that token_id              │
│   3. Emit "polymarket-update" internal event           │
└───────────────────────────────────────────────────────┘
```

**Hybrid approach**: Use WebSocket for low-latency price updates; fall back to HTTP polling every 10 seconds to catch new events and ensure consistency. The WebSocket alone may miss new event creation.

### 3.5 Rate Limits & Resilience

- Gamma API: No published rate limit; implement **exponential backoff** on 429/5xx.
- CLOB API: Subject to rate limits per IP; batch requests where possible.
- WebSocket: Implement automatic reconnect with jitter on disconnect.
- Circuit breaker pattern: If 3 consecutive failures, skip cycle and alert.

---

## 4. Module 2 — DraftKings Odds Data Service

### 4.1 Purpose

Continuously fetch DraftKings sportsbook betting odds for all sports events. Uses a **tiered approach**: try the DraftKings unofficial endpoints first, fall back to The Odds API aggregator.

### 4.2 Primary Source — DraftKings Unofficial REST API

> ⚠️ **Disclaimer**: These endpoints are **unofficial** and reverse-engineered from the DraftKings web client. They may change without notice. Monitor for breakage and be aware of TOS considerations.

| Endpoint | Purpose | Example |
|----------|---------|---------|
| `GET /sites/US-DK/sports/v1/sports?format=json` | List all available sports | Returns sport IDs/names |
| `GET /sportscontent/dkusnj/v1/leagues/{leagueId}` | Events for a league | NFL, NBA, etc. |
| `GET /sportscontent/dkusnj/v1/events/{eventId}` | Single event with odds | Moneyline, spread, totals |
| `GET /sportscontent/dkusnj/v1/events/{eventId}/categories/{catId}` | Specific market category | Moneyline odds |

**Base URL**: `https://sportsbook-nash.draftkings.com` (or `https://api.draftkings.com`)

### 4.3 Fallback Source — The Odds API

When DraftKings unofficial endpoints are unavailable or return errors, fall back to **The Odds API** (`https://api.the-odds-api.com/v4`).

| Endpoint | Purpose | Parameters |
|----------|---------|------------|
| `GET /v4/sports` | List in-season sports | `apiKey` |
| `GET /v4/sports/{sport}/odds` | Odds for events in a sport | `regions=us&markets=h2h,spreads,totals&bookmakers=draftkings&oddsFormat=decimal&apiKey=...` |
| `GET /v4/sports/{sport}/events` | Event list | `apiKey` |
| `GET /v4/sports/{sport}/events/{eventId}/odds` | Single event odds | `bookmakers=draftkings&apiKey=...` |

**Key filter**: Always pass `bookmakers=draftkings` to receive only DraftKings lines. Request `oddsFormat=decimal` for uniform calculation.

### 4.4 Data Model — DraftKings Odds

```typescript
interface DraftKingsEvent {
  id: string;                  // Source event ID
  source: "draftkings" | "the-odds-api";  // Which source provided data
  sport: string;               // Normalized: "basketball", "football"
  league: string;              // "nba", "nfl"
  homeTeam: string;
  awayTeam: string;
  startTime: string;           // ISO 8601
  markets: DraftKingsMarket[];
}

interface DraftKingsMarket {
  type: "h2h" | "spreads" | "totals";  // Market type
  outcomes: DraftKingsOutcome[];
}

interface DraftKingsOutcome {
  name: string;                // Team name or "Over"/"Under"
  americanOdds: number;        // e.g. -150, +130
  decimalOdds: number;         // e.g. 1.667, 2.300
  impliedProbability: number;  // Derived: 1 / decimalOdds
}
```

### 4.5 Odds Format Conversion

```
American → Decimal:
  If americanOdds > 0:  decimalOdds = (americanOdds / 100) + 1
  If americanOdds < 0:  decimalOdds = (100 / |americanOdds|) + 1

Decimal → Implied Probability:
  impliedProbability = 1 / decimalOdds

Examples:
  -150 → decimal 1.667 → implied 60.0%
  +130 → decimal 2.300 → implied 43.5%
```

### 4.6 Polling Strategy

```
┌──────────────────────────────────────────────────────┐
│            DraftKings Odds Data Service                │
│                                                       │
│   Every 10 seconds:                                   │
│   1. Try DK unofficial endpoint for each sport        │
│      └─ On success: normalize → update cache          │
│      └─ On failure (HTTP error / timeout / parse err) │
│         │                                             │
│         ▼                                             │
│   2. Fallback to The Odds API                         │
│      └─ GET /v4/sports/{sport}/odds?bookmakers=dk     │
│      └─ Normalize → update cache                      │
│                                                       │
│   Source tracking:                                     │
│   • Tag each record with source ("draftkings" or      │
│     "the-odds-api") for transparency                  │
│   • Log source switches for monitoring                │
└──────────────────────────────────────────────────────┘
```

### 4.7 The Odds API Budget

- Free tier: 500 requests/month. At 6 req/min fallback = 8,640/day → **requires paid plan** if used as primary.
- Recommendation: Use DK unofficial as primary; The Odds API only on failure. Budget for paid plan ($20–$80/month) if DK endpoints become unreliable.

---

## 5. Module 3 — Web Frontend

### 5.1 Purpose

Display matched Polymarket × DraftKings opportunities in a ranked, color-coded table, refreshed every 10 seconds.

### 5.2 Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Framework | **Next.js 14+** (App Router) | SSR for initial load, React for interactivity |
| Styling | **Tailwind CSS** | Rapid styling, conditional classes for colors |
| State | **React Query (TanStack Query)** | Built-in polling, caching, stale management |
| Real-time | **WebSocket** (or SSE fallback) | Push updates from backend |
| Table | **TanStack Table** | Sorting, virtualization for large datasets |

### 5.3 Data Flow — Frontend

```
┌─────────────────────────────────────────────────────┐
│                   Frontend Data Flow                  │
│                                                      │
│   useQuery("/api/opportunities", {                   │
│     refetchInterval: 10_000   // 10 seconds          │
│   })                                                 │
│       │                                              │
│       ▼                                              │
│   WebSocket connection to /ws/opportunities          │
│   (if push available, disable polling)               │
│       │                                              │
│       ▼                                              │
│   Merge response into state                          │
│       │                                              │
│       ▼                                              │
│   Sort by residual DESC                              │
│       │                                              │
│       ▼                                              │
│   Render <OpportunityTable>                          │
│     • Row color: green if residual > 0               │
│     • Row color: red if residual < 0                 │
│     • Columns: see §5.4                              │
└─────────────────────────────────────────────────────┘
```

### 5.4 Opportunity Table — Column Specification

| # | Column | Source | Description |
|---|--------|--------|-------------|
| 1 | **Event** | Matched | Sport icon + "Team A vs Team B" |
| 2 | **League** | Matched | NBA, NFL, MLB, etc. |
| 3 | **Start Time** | Matched | Localized event start |
| 4 | **PM YES Price** | Polymarket | Current YES price (e.g. $0.72) |
| 5 | **PM Probability** | Polymarket | = YES price × 100% (e.g. 72.0%) |
| 6 | **DK Odds** | DraftKings | American format (e.g. -150) |
| 7 | **DK Implied Prob** | DraftKings | = 1 / decimalOdds × 100% (e.g. 60.0%) |
| 8 | **Prob Δ** | Calculated | PM prob − DK implied prob |
| 9 | **Hedge Bet ($)** | Calculated | DraftKings bet to breakeven (see §6) |
| 10 | **Short-Sale Proceeds ($)** | Calculated | 10 × P_yes |
| 11 | **Residual ($)** | Calculated | Proceeds − Hedge Bet (see §6) |
| 12 | **Source** | Backend | "DK Direct" or "Odds API" badge |

### 5.5 Color Coding & Highlighting

```css
/* Residual cell and row highlighting */
.residual-positive {
  @apply bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400;
  /* Green background for positive residuals */
}

.residual-negative {
  @apply bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400;
  /* Red background for negative residuals */
}
```

Entire row receives a subtle tint; the **Residual** column cell receives a stronger color.

### 5.6 Frontend Component Tree

```
<App>
  <Header />                      // Logo, refresh indicator, connection status
  <DashboardPage>
    <FilterBar />                  // Sport/league filter, search, auto-refresh toggle
    <OpportunityTable>             // Main ranked table
      <TableHeader />              // Sortable column headers
      <TableBody>
        <OpportunityRow />         // One per matched event, colored by residual
          <ResidualCell />         // Strongly colored green/red
      </TableBody>
    </OpportunityTable>
    <RefreshTimer />               // Countdown to next refresh (10s)
    <StatusFooter />               // Data source health, last update timestamp
  </DashboardPage>
</App>
```

---

## 6. Core Calculation — Hedge & Residual Math

### 6.1 The Strategy

The user **short-sells 10 YES contracts** on Polymarket and **hedges** by placing a bet on DraftKings for the same outcome (event occurs). The goal is to find how much to bet on DraftKings to break even if the event occurs, and measure the residual profit if it doesn't.

### 6.2 Variables

| Symbol | Definition |
|--------|-----------|
| `P_yes` | Current Polymarket YES price (0 to 1, e.g. 0.72) |
| `N` | Number of YES contracts short-sold = **10** |
| `D` | DraftKings decimal odds for the same outcome (e.g. 1.667) |
| `B` | Amount to bet on DraftKings (in $) — **solve for this** |

### 6.3 Scenario Analysis

**Short-selling 10 YES contracts at P_yes:**
- You sell 10 YES contracts at the current YES price.
- Proceeds received immediately: `N × P_yes`

**If the event OCCURS (YES wins):**
- Polymarket: Each YES contract settles at $1.00. You owe $1 per contract.
- Polymarket P&L: `N × P_yes − N × $1.00 = −N × (1 − P_yes)`
- DraftKings: Your bet pays out `B × D` (total return), profit = `B × (D − 1)`
- **Combined P&L** = `B × (D − 1) − N × (1 − P_yes)`

**If the event DOES NOT OCCUR (NO wins):**
- Polymarket: YES contracts settle at $0. You owe nothing.
- Polymarket P&L: `+N × P_yes` (you keep the proceeds)
- DraftKings: You lose your bet.
- **Combined P&L** = `N × P_yes − B`

### 6.4 Breakeven Condition

Set the combined P&L to zero when the **event occurs**:

```
B × (D − 1) − N × (1 − P_yes) = 0

B = N × (1 − P_yes) / (D − 1)
```

### 6.5 Residual Calculation

The **residual** is the profit when the event **does not occur**, after subtracting the hedge bet:

```
Residual = N × P_yes − B
         = N × P_yes − N × (1 − P_yes) / (D − 1)
         = N × [ P_yes − (1 − P_yes) / (D − 1) ]
```

### 6.6 Worked Example

| Parameter | Value |
|-----------|-------|
| Event | Lakers vs Celtics — "Will Lakers win?" |
| P_yes (Polymarket) | $0.72 |
| DK American odds (Lakers win) | -150 |
| DK Decimal odds | 1.667 |
| N (contracts) | 10 |

**Step 1 — Short-sale proceeds:**
```
Proceeds = 10 × $0.72 = $7.20
```

**Step 2 — Implied probabilities:**
```
Polymarket implied probability = 72.0%
DraftKings implied probability = 1/1.667 = 60.0%
Probability delta = +12.0% (PM thinks it's MORE likely)
```

**Step 3 — Breakeven hedge bet:**
```
B = 10 × (1 − 0.72) / (1.667 − 1)
  = 10 × 0.28 / 0.667
  = $4.20
```

**Step 4 — Residual:**
```
Residual = $7.20 − $4.20 = $3.00 ✅ (positive → green)
```

**Verification — If event occurs:**
```
PM P&L  = -10 × (1 − 0.72) = -$2.80
DK P&L  = $4.20 × (1.667 − 1) = +$2.80
Net     = $0.00 ✓ (breakeven)
```

**Verification — If event does not occur:**
```
PM P&L  = +$7.20
DK P&L  = -$4.20
Net     = +$3.00 ✓ (matches residual)
```

### 6.7 When Is Residual Positive?

```
Residual > 0
⟺ P_yes > (1 − P_yes) / (D − 1)
⟺ P_yes × (D − 1) > 1 − P_yes
⟺ P_yes × D − P_yes > 1 − P_yes
⟺ P_yes × D > 1
⟺ P_yes > 1/D
⟺ Polymarket probability > DraftKings implied probability
```

**Insight**: The residual is positive exactly when Polymarket assigns a **higher** probability to the event than DraftKings does. This means you're selling overpriced YES contracts and cheaply hedging on DraftKings.

---

## 7. Event Matching Engine

### 7.1 Challenge

Polymarket and DraftKings use different event naming, IDs, and structures. The matching engine must reliably pair the same real-world sporting events across both sources.

### 7.2 Matching Algorithm

```
┌──────────────────────────────────────────────────────────┐
│                  Event Matching Pipeline                   │
│                                                           │
│  Step 1: Normalize                                        │
│  ├─ Lowercase all team/event names                        │
│  ├─ Strip suffixes ("FC", "City", etc.)                   │
│  ├─ Apply alias map (e.g. "LAL" → "lakers")              │
│  ├─ Normalize league names ("NBA" → "nba")                │
│  └─ Parse dates to UTC                                    │
│                                                           │
│  Step 2: Index                                            │
│  ├─ Build map: (sport, homeTeam, awayTeam, date) → event  │
│  └─ For both Polymarket and DraftKings event sets          │
│                                                           │
│  Step 3: Match                                            │
│  ├─ Exact key match first                                 │
│  ├─ Fuzzy match (Levenshtein ≤ 2) on team names           │
│  ├─ Date tolerance: ±12 hours (timezone differences)       │
│  └─ Produce matched pairs                                 │
│                                                           │
│  Step 4: Validate                                         │
│  ├─ Confirm sport matches                                 │
│  ├─ Confirm at least one team name matches                │
│  └─ Flag low-confidence matches for review                │
└──────────────────────────────────────────────────────────┘
```

### 7.3 Team Name Alias Map (example)

```json
{
  "lal": "los angeles lakers",
  "lakers": "los angeles lakers",
  "bos": "boston celtics",
  "celtics": "boston celtics",
  "kc": "kansas city chiefs",
  "chiefs": "kansas city chiefs",
  "nyy": "new york yankees",
  "yankees": "new york yankees"
}
```

This alias map is extensible and should be maintained as a configuration file.

---

## 8. Backend API Server

### 8.1 Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Runtime | **Node.js 20+** | Async I/O, npm ecosystem |
| Framework | **Fastify** | High performance, schema validation |
| Scheduler | **node-cron** or **setInterval** | 10-second polling cycle |
| Cache | **In-memory Map** (or Redis for multi-instance) | Low-latency reads |
| WebSocket | **@fastify/websocket** | Push merged data to frontend |
| HTTP client | **undici** (built-in) or **axios** | API requests to PM/DK |
| Fuzzy matching | **fuse.js** | Team name fuzzy search |

### 8.2 API Endpoints

```
GET  /api/health
     → { status: "ok", polymarketConnected: true, draftkingsSource: "direct" }

GET  /api/opportunities
     → OpportunityRow[]   (full merged, calculated table, sorted by residual DESC)

GET  /api/polymarket/events
     → PolymarketEvent[]  (raw Polymarket sports events)

GET  /api/draftkings/events
     → DraftKingsEvent[]  (raw DraftKings odds data)

WS   /ws/opportunities
     → Pushes OpportunityRow[] on every data refresh cycle (every 10s)
```

### 8.3 Response Schema — OpportunityRow

```typescript
interface OpportunityRow {
  // Identification
  matchId: string;                  // Deterministic hash of matched event
  sport: string;
  league: string;
  homeTeam: string;
  awayTeam: string;
  eventStartTime: string;           // ISO 8601

  // Polymarket data
  pmMarketId: string;
  pmYesPrice: number;               // e.g. 0.72
  pmProbability: number;            // e.g. 0.72 (= pmYesPrice)
  pmVolume: number;
  pmLiquidity: number;

  // DraftKings data
  dkEventId: string;
  dkAmericanOdds: number;           // e.g. -150
  dkDecimalOdds: number;            // e.g. 1.667
  dkImpliedProbability: number;     // e.g. 0.600
  dkSource: "draftkings" | "the-odds-api";

  // Calculated fields
  probabilityDelta: number;         // pmProbability - dkImpliedProbability
  shortSaleProceeds: number;        // 10 * pmYesPrice
  hedgeBet: number;                 // 10 * (1 - pmYesPrice) / (dkDecimalOdds - 1)
  residual: number;                 // shortSaleProceeds - hedgeBet

  // Metadata
  matchConfidence: number;          // 0-1, how confident the matching is
  lastUpdated: string;              // ISO 8601
}
```

### 8.4 Server Orchestration Loop

```
┌──────────────────────────────────────────────────────┐
│              Main Orchestration Loop                   │
│              (runs every 10 seconds)                   │
│                                                       │
│   1. PARALLEL fetch:                                  │
│      ├─ Polymarket Data Service → PM events           │
│      └─ DraftKings Data Service → DK events           │
│                                                       │
│   2. Event Matching Engine                            │
│      └─ Produce matched pairs                         │
│                                                       │
│   3. For each matched pair:                           │
│      ├─ Calculate implied probabilities               │
│      ├─ Calculate hedge bet: B = 10*(1-P)/(D-1)      │
│      ├─ Calculate residual: 10*P - B                  │
│      └─ Build OpportunityRow                          │
│                                                       │
│   4. Sort by residual DESC                            │
│                                                       │
│   5. Update cache                                     │
│                                                       │
│   6. Push to all WebSocket subscribers                │
│                                                       │
│   Total budget per cycle: < 8 seconds                 │
│   (leave 2s headroom before next cycle)               │
└──────────────────────────────────────────────────────┘
```

---

## 9. Refresh Strategy — 6 Times Per Minute

### 9.1 Timing

```
Refresh interval = 60s / 6 = 10 seconds per cycle
```

### 9.2 Implementation

```typescript
// Backend: orchestration timer
const REFRESH_INTERVAL_MS = 10_000; // 10 seconds

let isRunning = false;

setInterval(async () => {
  if (isRunning) {
    logger.warn("Previous cycle still running, skipping");
    return;
  }
  isRunning = true;
  try {
    const [pmEvents, dkEvents] = await Promise.all([
      polymarketService.fetchSportsEvents(),
      draftkingsService.fetchOdds(),
    ]);
    const matched = matchingEngine.match(pmEvents, dkEvents);
    const opportunities = calculator.compute(matched);
    opportunities.sort((a, b) => b.residual - a.residual);
    cache.set("opportunities", opportunities);
    wsServer.broadcast(opportunities);
  } catch (err) {
    logger.error("Cycle failed", err);
  } finally {
    isRunning = false;
  }
}, REFRESH_INTERVAL_MS);
```

```typescript
// Frontend: polling with React Query
const { data: opportunities } = useQuery({
  queryKey: ["opportunities"],
  queryFn: () => fetch("/api/opportunities").then(r => r.json()),
  refetchInterval: 10_000,  // 10 seconds, aligned with backend
  staleTime: 8_000,
});
```

### 9.3 Timing Budget Per Cycle

| Phase | Target Duration | Notes |
|-------|----------------|-------|
| Polymarket API calls | ≤ 3s | Parallel, cached connections |
| DraftKings API calls | ≤ 3s | With fallback switch ≤ 5s |
| Matching + Calculation | ≤ 0.5s | In-memory, O(n×m) with indexing |
| WebSocket broadcast | ≤ 0.1s | Serialization + push |
| **Total** | **≤ 7s** | **3s headroom before next cycle** |

---

## 10. Project Structure

```
p1/
├── ARCHITECTURE.md                  # This document
├── package.json                     # Root monorepo config
├── packages/
│   ├── backend/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── index.ts             # Fastify server bootstrap
│   │   │   ├── config.ts            # Environment config
│   │   │   ├── services/
│   │   │   │   ├── polymarket.ts    # Polymarket Data Service
│   │   │   │   ├── draftkings.ts    # DraftKings Data Service (primary)
│   │   │   │   ├── oddsApi.ts       # The Odds API fallback
│   │   │   │   └── cache.ts         # In-memory cache
│   │   │   ├── engine/
│   │   │   │   ├── matcher.ts       # Event Matching Engine
│   │   │   │   ├── calculator.ts    # Hedge/residual calculations
│   │   │   │   └── normalizer.ts    # Team name normalization
│   │   │   ├── routes/
│   │   │   │   ├── opportunities.ts # GET /api/opportunities
│   │   │   │   ├── polymarket.ts    # GET /api/polymarket/events
│   │   │   │   ├── draftkings.ts    # GET /api/draftkings/events
│   │   │   │   └── health.ts       # GET /api/health
│   │   │   ├── ws/
│   │   │   │   └── opportunities.ts # WebSocket handler
│   │   │   ├── orchestrator.ts      # 10s cycle manager
│   │   │   └── types.ts            # Shared TypeScript interfaces
│   │   └── tests/
│   │       ├── matcher.test.ts
│   │       ├── calculator.test.ts
│   │       └── normalizer.test.ts
│   └── frontend/
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.js
│       ├── tailwind.config.js
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx          # Dashboard page
│       │   │   └── globals.css
│       │   ├── components/
│       │   │   ├── OpportunityTable.tsx
│       │   │   ├── OpportunityRow.tsx
│       │   │   ├── ResidualCell.tsx
│       │   │   ├── FilterBar.tsx
│       │   │   ├── RefreshTimer.tsx
│       │   │   ├── StatusFooter.tsx
│       │   │   └── Header.tsx
│       │   ├── hooks/
│       │   │   ├── useOpportunities.ts  # React Query + WS
│       │   │   └── useWebSocket.ts
│       │   ├── lib/
│       │   │   ├── api.ts            # API client
│       │   │   └── format.ts         # Number/odds formatters
│       │   └── types.ts             # Shared frontend types
│       └── tests/
│           └── OpportunityTable.test.tsx
├── docker-compose.yml               # Local dev environment
└── .env.example                     # Configuration template
```

---

## 11. Configuration & Environment

```bash
# .env.example

# Server
PORT=3001
NODE_ENV=development

# Polymarket
POLYMARKET_GAMMA_URL=https://gamma-api.polymarket.com
POLYMARKET_CLOB_URL=https://clob.polymarket.com
POLYMARKET_WS_URL=wss://ws-subscriptions-clob.polymarket.com/ws/market
POLYMARKET_US_URL=https://gateway.polymarket.us

# DraftKings (unofficial)
DRAFTKINGS_API_URL=https://sportsbook-nash.draftkings.com
DRAFTKINGS_ENABLED=true

# The Odds API (fallback)
ODDS_API_KEY=your_api_key_here
ODDS_API_URL=https://api.the-odds-api.com/v4
ODDS_API_ENABLED=true

# Orchestration
REFRESH_INTERVAL_MS=10000
SHORT_SELL_CONTRACTS=10

# Frontend
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_WS_URL=ws://localhost:3001/ws/opportunities
NEXT_PUBLIC_REFRESH_INTERVAL_MS=10000
```

---

## 12. Deployment Architecture

```
┌───────────────────────────────────────────────────────────┐
│                    Production Deployment                    │
│                                                            │
│   ┌─────────────────────────────────────────────────────┐ │
│   │  CDN / Vercel Edge                                   │ │
│   │  └─ Next.js Frontend (static + SSR)                  │ │
│   └──────────────────────┬──────────────────────────────┘ │
│                          │                                 │
│   ┌──────────────────────▼──────────────────────────────┐ │
│   │  Load Balancer / Reverse Proxy (nginx / Caddy)       │ │
│   └──────────────────────┬──────────────────────────────┘ │
│                          │                                 │
│   ┌──────────────────────▼──────────────────────────────┐ │
│   │  Backend Container(s)                                │ │
│   │  ├─ Fastify API server                               │ │
│   │  ├─ Polymarket Service (WebSocket + poller)          │ │
│   │  ├─ DraftKings Service (poller + fallback)           │ │
│   │  ├─ Matching Engine                                  │ │
│   │  └─ WebSocket server for frontend push               │ │
│   └──────────────────────┬──────────────────────────────┘ │
│                          │                                 │
│   ┌──────────────────────▼──────────────────────────────┐ │
│   │  Redis (optional, for multi-instance cache sharing)  │ │
│   └─────────────────────────────────────────────────────┘ │
│                                                            │
│   Hosting options:                                         │
│   • Railway / Render / Fly.io (containers)                 │
│   • Vercel (frontend) + separate backend host              │
│   • Self-hosted VPS with Docker Compose                    │
└───────────────────────────────────────────────────────────┘
```

---

## 13. Error Handling & Resilience

| Scenario | Handling |
|----------|----------|
| Polymarket API down | Return cached data; show "stale" badge on frontend with timestamp |
| DraftKings unofficial API breaks | Auto-switch to The Odds API; log alert |
| The Odds API rate limit exceeded | Queue requests; degrade to longer polling interval |
| WebSocket disconnect (PM) | Auto-reconnect with exponential backoff (1s, 2s, 4s, max 30s) |
| Event match confidence < 0.7 | Exclude from main table; show in "low confidence" section |
| Cycle exceeds 10s | Skip next cycle; log warning; monitor for persistent slowness |
| Frontend loses WS connection | Fall back to HTTP polling; show reconnection indicator |

---

## 14. Monitoring & Observability

| Metric | Tool | Alert Threshold |
|--------|------|-----------------|
| Cycle duration | Prometheus / custom logging | > 8 seconds |
| Source failover events | Structured logs | Any occurrence |
| Matched event count | Dashboard metric | Drop > 50% from previous |
| WebSocket client count | Server metric | Informational |
| API error rate (PM/DK) | Error counter | > 10% per 5 min |
| The Odds API usage | Request counter | > 80% of monthly quota |

---

## 15. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| API keys in code | Use environment variables, never commit to repo |
| The Odds API key exposure | Backend-only; never send to frontend |
| DraftKings TOS | Monitor unofficial API status; be prepared to go aggregator-only |
| Rate limiting own API | Apply rate limits on `/api/*` to prevent abuse |
| CORS | Configure to allow only frontend origin |
| WebSocket auth | Token-based auth if deployed publicly |

---

## 16. Future Enhancements

1. **Historical tracking**: Store opportunity snapshots in PostgreSQL for backtesting.
2. **Alerts**: Push notifications (email/Slack/Telegram) when residual exceeds a threshold.
3. **Multi-outcome markets**: Support spread and totals markets beyond moneyline.
4. **Configurable contract count**: Let user change the 10-contract default via UI slider.
5. **Multiple sportsbooks**: Add FanDuel, BetMGM columns for comparison.
6. **Auto-execution**: Integrate Polymarket CLOB trading SDK for automated short-selling.
7. **Mobile-responsive**: Optimize table for mobile with collapsible columns.
8. **Dark/light mode**: Theme toggle (Tailwind dark mode support).

---

## 17. Glossary

| Term | Definition |
|------|-----------|
| **YES contract** | A Polymarket binary contract that pays $1 if the event occurs, $0 otherwise |
| **Short-sell** | Selling YES contracts you don't own (equivalent to buying NO contracts) |
| **Implied probability** | The probability of an outcome as implied by betting odds: `1 / decimalOdds` |
| **Decimal odds** | European odds format: total payout per $1 wagered (includes stake) |
| **American odds** | US odds format: -150 means bet $150 to win $100; +130 means bet $100 to win $130 |
| **Hedge bet** | A bet placed to offset risk from another position |
| **Residual** | Net profit from the combined PM short + DK hedge when the event does NOT occur |
| **CLOB** | Central Limit Order Book — Polymarket's order matching system |
| **Gamma API** | Polymarket's public API for market discovery and metadata |

---

## 18. Appendix — API Reference Quick Sheet

### Polymarket

```bash
# List all sports events
curl "https://gamma-api.polymarket.com/events?tag=sports&active=true&limit=100"

# Get markets for an event
curl "https://gamma-api.polymarket.com/markets?eventId=EVENT_ID"

# Get orderbook for a market outcome
curl "https://clob.polymarket.com/order-book/TOKEN_ID"

# Sports events by league (US)
curl "https://gateway.polymarket.us/v2/leagues/nba/events"
```

### DraftKings (Unofficial)

```bash
# List sports
curl "https://api.draftkings.com/sites/US-DK/sports/v1/sports?format=json"

# Events for a league
curl "https://sportsbook-nash.draftkings.com/sportscontent/dkusnj/v1/leagues/42648"
```

### The Odds API (Fallback)

```bash
# List sports
curl "https://api.the-odds-api.com/v4/sports?apiKey=API_KEY"

# DraftKings odds for NBA
curl "https://api.the-odds-api.com/v4/sports/basketball_nba/odds?regions=us&markets=h2h&bookmakers=draftkings&oddsFormat=decimal&apiKey=API_KEY"
```
