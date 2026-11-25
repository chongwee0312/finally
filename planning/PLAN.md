# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area (also built from SSE data since page load; there is no historical price endpoint)
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog, fractional shares supported
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend initializes the database on startup — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: SQLite file location. Default: <project root>/db/finally.db
# The Dockerfile sets DB_PATH=/app/db/finally.db
DB_PATH=
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- If `OPENROUTER_API_KEY` is missing and `LLM_MOCK` is not `true` → the app still starts; `/api/chat` returns a friendly assistant message saying the AI is unavailable
- **In Docker**: variables come from `--env-file .env`
- **Local dev** (`uv run` from `backend/`): the backend loads `<project root>/.env`, resolved from the backend's own file location (not the working directory). Already-set environment variables take precedence

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all tracked tickers (see below) on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- A newly added ticker has no price until the next poll (the simulator, by contrast, seeds a price immediately)

### Tracked Tickers

The market data source prices the **union of the watchlist and all held positions**, so a position can always be valued:

- On startup: `source.start(watchlist ∪ positions)`
- Removing a ticker from the watchlist calls `source.remove_ticker()` only if no position is held
- Selling a position to zero calls `source.remove_ticker()` only if the ticker is not on the watchlist
- The SSE stream therefore may include held tickers that are not on the watchlist; the frontend shows only watchlist tickers in the watchlist panel

### Ticker Validation

- Tickers are normalized to uppercase and trimmed
- Format check: 1-5 letters (`^[A-Z]{1,5}$`); otherwise rejected with 400
- Known limitation: the simulator invents a price for any well-formed ticker; under Massive, an unknown ticker simply never receives a price and the UI shows `—`

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, session open price, and timestamp for each ticker
- **Session open price** = the first price the cache records for a ticker since the backend started (simulator: seed price; Massive: first poll). It is the reference for the watchlist change %, giving one definition for both sources
- Prices reset to seed values when the container restarts. This is accepted: P&L on held positions will jump after a restart in simulator mode
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

**Change to the completed market data code:** add `session_open` to `PriceUpdate` (set by `PriceCache.update()` from the first price seen for the ticker) and add `session_open` and `session_change_percent` to `to_dict()`.

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Every ~500ms, if any price changed (cache version check), the server sends **one event containing all tracked tickers**, keyed by ticker:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.50, "previous_price": 190.42, "timestamp": 1764057600.12, "change": 0.08, "change_percent": 0.042, "direction": "up", "session_open": 190.00, "session_change_percent": 0.2632}, "GOOGL": {...}}
```

- `change` / `change_percent` / `direction` are tick-over-tick (drive the flash animation); `session_change_percent` drives the watchlist change % column
- The stream starts with `retry: 1000`; client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Initialization on Startup

The backend checks for the SQLite database (at `DB_PATH`) in the FastAPI lifespan startup, the same place the market data source and snapshot task start. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Connections and Transactions

- Enable WAL mode (`PRAGMA journal_mode=WAL`) at startup; the trade endpoint, chat endpoint and snapshot task all write
- Use a short-lived connection per operation (no shared global connection)
- A trade is one transaction: update cash, upsert/delete position, insert trade, insert snapshot

### Schema

All tables except `users_profile` include a `user_id` column defaulting to `"default"`. `users_profile` uses its `id` as the user key. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task (even with no positions, so the chart has data from the start), and immediately after each trade execution. Rows grow unbounded (~2,880/day); acceptable for a demo, and the history endpoint is capped.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades and watchlist changes attempted, each with status and any error; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

### Trade Rules

Applied identically to manual trades and LLM trades:

- `quantity` must be > 0; fractional shares supported; rounded to 4 dp on write
- `side` is `"buy"` or `"sell"`
- Fill price = current price in the cache; if the ticker has no cached price, reject (400, "No price available for X")
- **Buy**: requires `cash_balance >= quantity × price`. New `avg_cost` = weighted average: `(old_qty × old_avg + qty × price) / (old_qty + qty)`
- **Sell**: requires held quantity >= `quantity`. `avg_cost` is unchanged. If remaining quantity < 0.0001, delete the position row
- Realized P&L is not tracked separately; it is reflected in cash only
- Rounding on write: cash to 2 dp; prices and `avg_cost` to 4 dp. Calculations use full precision before rounding

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers (prices come from SSE) |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/chat` | Recent conversation history (restores the chat panel on page load) |
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

### Response Contract

Errors use FastAPI's default shape: `{"detail": "Insufficient cash"}`. Status codes: 400 validation/business rule failure, 404 not found, 409 duplicate, 422 malformed request body (FastAPI default). The figures in the examples below are illustrative.

**`GET /api/portfolio`** → 200
```json
{
  "cash_balance": 8100.00,
  "total_value": 10050.30,
  "unrealized_pnl": 50.30,
  "positions": [
    {"ticker": "AAPL", "quantity": 10, "avg_cost": 190.00, "current_price": 195.03,
     "market_value": 1950.30, "unrealized_pnl": 50.30, "pnl_percent": 2.65}
  ]
}
```
`total_value` = cash + Σ market_value. `pnl_percent` = unrealized_pnl / (quantity × avg_cost) × 100.

**`POST /api/portfolio/trade`** `{"ticker": "AAPL", "quantity": 10, "side": "buy"}` → 200
```json
{
  "trade": {"id": "uuid", "ticker": "AAPL", "side": "buy", "quantity": 10, "price": 190.00, "executed_at": "2026-11-25T09:30:00Z"},
  "portfolio": { "...same shape as GET /api/portfolio..." }
}
```
400 on insufficient cash, insufficient shares, no price, quantity <= 0, or invalid ticker.

**`GET /api/portfolio/history?limit=1000`** → 200, oldest first; `limit` defaults to 1,000 (~8 hours at 30s)
```json
[{"total_value": 10000.00, "recorded_at": "2026-11-25T09:30:00Z"}]
```

**`GET /api/watchlist`** → 200 `{"tickers": ["AAPL", "GOOGL"]}`

**`POST /api/watchlist`** `{"ticker": "PYPL"}` → 201 `{"ticker": "PYPL"}`; 400 invalid format; 409 already on watchlist

**`DELETE /api/watchlist/{ticker}`** → 204; 404 if not on watchlist

**`POST /api/chat`** `{"message": "Buy 5 AAPL and watch PYPL"}` → 200
```json
{
  "message": "Added PYPL to your watchlist and bought 5 AAPL at $190.12.",
  "actions": {
    "watchlist_changes": [{"ticker": "PYPL", "action": "add", "status": "ok"}],
    "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 5, "price": 190.12, "status": "ok"}]
  }
}
```
A failed action has `"status": "error"` and an `"error"` string.

**`GET /api/chat?limit=50`** → 200, oldest first; `limit` defaults to 50
```json
[
  {"id": "uuid", "role": "user", "content": "Buy 5 AAPL", "actions": null, "created_at": "2026-11-25T09:30:00Z"},
  {"id": "uuid", "role": "assistant", "content": "Bought 5 AAPL at $190.12.",
   "actions": {"watchlist_changes": [], "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 5, "price": 190.12, "status": "ok"}]},
   "created_at": "2026-11-25T09:30:01Z"}
]
```
`actions` uses the same shape as the `POST /api/chat` response.

**`GET /api/health`** → 200 `{"status": "ok"}`

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 20 messages of conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

Execution order and semantics:

1. Apply `watchlist_changes` first, in order (so "add PYPL and buy 5" can work)
2. Then execute `trades` in order, each independently — one failure does not stop the others (partial success allowed)
3. Each action goes through the same rules as the manual endpoints (§7 Trade Rules, §6 Ticker Validation)
4. Under Massive, a ticker added in the same response has no price yet, so a trade on it fails with "No price available"

If an action fails, its error is recorded in `actions` and returned in the chat response (see §8). The assistant `message` is the LLM's text as generated; the frontend shows each action's status inline, so failures are visible even if the message does not mention them.

### Failure Handling

On LLM timeout, API error, missing `OPENROUTER_API_KEY`, or a response that fails schema validation:
- Execute no actions
- Return 200 with a friendly assistant message (e.g. "Sorry, I couldn't reach the AI service. Please try again.") and empty `actions`
- Store both the user message and this assistant message in `chat_messages`

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. The mock output then goes through the normal execution path. Rules (case-insensitive, checked on the user message):

| Message contains | Mock response |
|---|---|
| `buy` | trades: `[{"ticker": "AAPL", "side": "buy", "quantity": 1}]` |
| `sell` | trades: `[{"ticker": "AAPL", "side": "sell", "quantity": 1}]` |
| `watch` | watchlist_changes: `[{"ticker": "PYPL", "action": "add"}]` |
| anything else | message only: "This is a mock response from FinAlly." |

Rules combine (e.g. "watch and buy" returns both). This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), change % since session start (`session_change_percent`, labelled "Chg %", not "daily"), and a sparkline mini-chart (accumulated from SSE since page load). Shows `—` for a ticker with no price yet
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here. Uses the same per-ticker price buffer as the sparklines (data since page load)
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history (loaded from `GET /api/chat` on page load), loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Live Values

- Positions and cash come from `GET /api/portfolio`, fetched on load, after each manual trade (the trade response includes the portfolio), and after any chat response with actions
- Total value, market value and unrealized P&L update live on the client: cash + Σ quantity × latest SSE price

### Empty States

- **No positions**: heatmap and positions table show a short placeholder ("No positions yet")
- **P&L chart**: always has data, as snapshots are recorded even with cash only (flat line at $10,000)
- **Empty watchlist**: show a prompt to add a ticker

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Charting: **Lightweight Charts** (canvas) for the main chart, sparklines and P&L line chart; **Recharts** `Treemap` for the portfolio heatmap only
- Static export limits (`output: 'export'`): no Next.js API routes, server actions or middleware; set `images.unoptimized: true`. Use the current Next.js major version
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node current LTS, slim (Node 20 is end-of-life; pin the exact LTS major in the Dockerfile)
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - ENV DB_PATH=/app/db/finally.db
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, weighted-average cost, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, fractional quantities, selling to zero deletes the position, no cached price)
- Tracked tickers: removing a held ticker from the watchlist keeps it priced; selling to zero stops pricing a non-watchlist ticker
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses and API failures, watchlist-before-trades ordering, partial success, mock-mode rules
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline; after a page reload the conversation is still shown
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Decisions (2026-11-25)

Review questions are resolved and folded into the sections above:

| Topic | Decision | Section |
|---|---|---|
| SSE payload | One event per tick with all tickers keyed by symbol | §6 |
| Change % reference | Session open price (first price since backend start), both sources | §6, §10 |
| Restart price reset | Accepted; documented | §6 |
| Tracked tickers | Watchlist ∪ positions | §6 |
| Ticker validation | Uppercase, `^[A-Z]{1,5}$` only | §6 |
| Trade rules & precision | Fractional shares, weighted avg cost, 4 dp qty/price, 2 dp cash | §7 |
| DB init / concurrency | Startup lifespan, WAL, per-operation connections, one transaction per trade | §7 |
| API contract | JSON examples, FastAPI `{"detail": ...}` errors | §8 |
| `GET /api/watchlist` | Tickers only; prices via SSE | §8 |
| LLM execution | Watchlist changes first, trades in order, partial success; last 20 messages | §9 |
| LLM failures & mock | Friendly message, no actions; keyword-based mock rules | §9 |
| Chat history on reload | `GET /api/chat` (last 50, oldest first) restores the panel | §8, §10 |
| Portfolio history cap | `limit` default 1,000 (~8 hours) confirmed | §8 |
| Live values & empty states | Client-side totals from SSE; placeholders defined | §10 |
| Charts | Lightweight Charts + Recharts Treemap | §10 |
| Versions & paths | Node current LTS; `DB_PATH`; `.env` resolution | §5, §11 |
