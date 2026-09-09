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
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
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
│  SQLite database (bind-mounted)                 │
│  Background task: market data simulator          │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, bind-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (any free model), with structured outputs for trade execution
- **Market data**: built-in geometric Brownian motion simulator, running as an in-process background task (no external market data provider)

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
│   ├── finally.sh            # start | stop | restart the Docker container (macOS/Linux)
│   └── finally.ps1           # start | stop | restart the Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Bind-mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is bind-mounted into the container. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts. Being a bind mount (not a named volume), the file is directly visible and inspectable on the host.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml` — the only compose file in the project). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains one wrapper script per platform (`finally.sh` / `finally.ps1`) taking a `start`/`stop`/`restart` subcommand.

---

## 5. Environment Variables

```bash
# OpenRouter API key for LLM chat functionality.
# If absent or empty, the backend automatically runs in mock LLM mode.
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: force deterministic mock LLM responses even when a key is present (testing)
LLM_MOCK=false
```

### Behavior

- Market data always comes from the built-in simulator — there is no external provider and no env var to configure it.
- If `OPENROUTER_API_KEY` is absent or empty → mock LLM mode is enabled automatically (no error).
- If `LLM_MOCK=true` → mock LLM mode is forced on regardless of the key (for E2E tests and CI).
- Otherwise → the backend calls OpenRouter with the configured key.
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`).

---

## 6. Market Data

### Simulator (the only source)

- Generates prices using geometric Brownian motion (GBM) with per-ticker drift and volatility
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies, no API key
- Defines a fixed **known universe** of tickers it has seed prices and GBM parameters for (in `backend/app/market/seed_prices.py`). The watchlist can only hold tickers from this universe; it must contain enough symbols beyond the 10 defaults for "add a ticker" to be demonstrable.

### Opening Price / Daily Change

- The simulator records each ticker's **opening price for the current US Eastern trading day** — the first price it produces on or after 09:30 ET, or the first price after process start if that is later.
- Daily change % shown in the UI is `(current − open) / open`.
- The opening price is exposed to clients (see SSE payload and `/api/watchlist`) so the frontend does not have to infer a baseline.

### Shared Price Cache

- The simulator background task writes to an in-memory price cache
- The cache holds the latest price, previous price, opening price, and timestamp for each ticker
- SSE streams and REST endpoints read from this cache
- The cache is source-agnostic, so an alternative data source could be added later without touching downstream code

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The server emits an event for a ticker **only when its price changes**, checking the cache ~every 500ms — in the single-user model the set of tickers is the user's watchlist
- Each SSE event contains ticker, price, previous price, opening price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Conventions

- All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration. **No endpoint accepts a user id**; every query filters on the constant `user_id = "default"`.
- The database is opened in **WAL mode** and transactions are kept short — the simulator, the snapshot writer, and request handlers all touch the same file.
- All timestamps are **US Eastern time** (`America/New_York`), ISO 8601.
- Share quantities are **whole integers** — fractional shares are not supported anywhere (trade bar, REST endpoint, or LLM-supplied trades). Quantity must be a positive integer ≥ 1.

### Schema

**users_profile** — User state (cash balance, realized P&L)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `realized_pnl` REAL (default: `0.0`) — cumulative realized gain/loss; on each sell, `realized_pnl += (sell_price − avg_cost) × quantity`
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
- `quantity` INTEGER
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`
- On a **buy**, `avg_cost` becomes the quantity-weighted average of the existing lot and the new fill. On a **sell**, `avg_cost` is unchanged and `quantity` decreases. When `quantity` reaches 0 the row is **deleted**.

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` INTEGER
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Written on each trade execution, and lazily appended when `GET /api/portfolio/history` is requested if more than 30s have elapsed since the last snapshot. **No dedicated background timer.**
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON; `null` for user messages) — the outcome of every action the assistant attempted:
  ```json
  {
    "trades": [
      {"ticker": "AAPL", "side": "buy", "quantity": 10, "price": 190.12, "status": "filled"},
      {"ticker": "TSLA", "side": "buy", "quantity": 5, "status": "rejected", "reason": "Insufficient cash"}
    ],
    "watchlist_changes": [
      {"ticker": "PYPL", "action": "add", "status": "applied"},
      {"ticker": "XYZ", "action": "add", "status": "rejected", "reason": "Unknown ticker"}
    ]
  }
  ```
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`, `realized_pnl=0.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX (the watchlist cap is also 10, so the default watchlist starts full)

---

## 8. API Endpoints

### Error Contract

Every error response uses a single envelope and an appropriate status code:

```json
{ "error": { "code": "INSUFFICIENT_CASH", "message": "Need $2,140.00, have $1,905.33." } }
```

- `400` — business-rule / validation failure (insufficient cash, insufficient shares, unknown ticker, watchlist full, non-positive quantity)
- `422` — malformed request body (FastAPI default)
- `404` — removing a ticker not on the watchlist

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, realized P&L, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}`. On success returns the **updated portfolio** (same shape as `GET /api/portfolio`) so the UI updates from one payload |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart); lazily appends a snapshot if >30s since the last |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}`. Symbol is upper-cased and trimmed. Rejected (`400`) if outside the simulator's known universe, already present, or the watchlist already holds **10** tickers (the cap) |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker. Allowed even if an open position exists for it — the position keeps its streamed price |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message; returns `{message, actions, portfolio}` — the assistant's reply, per-action outcomes, and the resulting portfolio state |
| GET | `/api/chat/history` | Full stored conversation (for repopulating the transcript on page reload) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

Make LLM calls with LiteLLM via OpenRouter, using any free model of your choice — do not hardcode a specific model. Request structured output to interpret the results, and fall back to prompt-based JSON with a tolerant parser if the chosen model does not support `response_format`/JSON-schema.

There is an `OPENROUTER_API_KEY` in the `.env` file in the project root. If it is missing or empty, run in mock mode (see below).

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the **last 20 messages** of conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and per-action outcomes in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — inference is fast enough that a loading indicator is sufficient)

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
- `trades` (optional): Array of trades to auto-execute. `quantity` is a positive integer. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications; `action` is `"add"` or `"remove"`

**Tool scope:** trades and watchlist changes are the assistant's *entire* capability surface. It cannot set the cash balance, reset the portfolio, or take any other action.

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

Actions are applied **in array order**. Each is validated independently: the ones that pass are applied, the ones that fail are skipped, and every item's outcome (`status` + `reason`) is returned in the chat response and stored in `chat_messages.actions` so the assistant can explain what happened.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true` **or `OPENROUTER_API_KEY` is missing/empty**, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

**Mock contract** — the mock maps the user's message to a fixed structured response:

| Input pattern | Mock response |
|---|---|
| `/buy (\d+) ([A-Z]+)/i` | `{"message": "Bought {n} {ticker}.", "trades": [{"ticker": "{ticker}", "side": "buy", "quantity": {n}}]}` |
| `/sell (\d+) ([A-Z]+)/i` | symmetric sell |
| `/watch ([A-Z]+)/i` | `{"message": "Added {ticker}.", "watchlist_changes": [{"ticker": "{ticker}", "action": "add"}]}` |
| anything else | `{"message": "<fixed analysis string>", "trades": [], "watchlist_changes": []}` |

The mocked actions still go through real validation and execution, so a mocked `buy` that exceeds cash is rejected exactly like a live one.

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (`(price − open) / open`, using the opening price from the SSE payload), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker. It shares the sparkline's data — the price series accumulated from the SSE stream since page load, rendered larger — with no historical-price endpoint. It fills in progressively and starts empty on each reload. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart of total portfolio value over time. Backfilled on load from `GET /api/portfolio/history` (`portfolio_snapshots`), then extended live.
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field (whole shares), buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. On load, the transcript is repopulated from `GET /api/chat/history`. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations (from the response's per-action outcomes).
- **Header** — portfolio total value, connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- **Portfolio total value is computed client-side**: `Σ(position qty × latest streamed price) + cash`. The frontend already holds positions and the price stream, so there is no polling loop and no server push of portfolio state. `GET /api/portfolio` is used only for the initial load; mutating endpoints (`/api/portfolio/trade`, `/api/chat`) return the updated portfolio in their responses.
- Use **Lightweight Charts** for all three time-series charts (sparkline, main chart, P&L). Use a small dedicated treemap library (e.g. Nivo or visx) for the heatmap, since Lightweight Charts has no treemap.
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- Display prices and P&L to 2 decimal places
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm ci && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Database Persistence — Bind Mount

The SQLite database persists via a bind mount of the project's `db/` directory:

```bash
docker run -v "$(pwd)/db:/app/db" -p 8000:8000 --env-file .env finally
```

The backend writes `finally.db` into `/app/db`, which is the host's `db/` directory — so the file is directly inspectable on the host.

### Wrapper Scripts

**`scripts/finally.sh <start|stop|restart>`** (macOS/Linux) and **`scripts/finally.ps1 <start|stop|restart>`** (Windows PowerShell):

- `start` — builds the image if needed (or on `--build`), runs the container with the bind mount, port mapping, and `.env` file, prints the URL, optionally opens the browser
- `stop` — stops and removes the running container; leaves `db/finally.db` in place
- `restart` — `stop` then `start`

All subcommands are idempotent — safe to run repeatedly.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

**Access control:** the app has no authentication by design. A public cloud deployment exposes `/api/chat` — and the OpenRouter spend behind it — to anyone with the URL. Any cloud deployment must sit behind network or platform-level access restrictions.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, opening-price capture works, unknown tickers are rejected
- Portfolio: trade execution logic, weighted-average `avg_cost`, position deleted at qty 0, realized-P&L accumulation, integer-only quantity, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, per-action partial-failure outcomes, mock-contract patterns, trade validation within chat flow
- API routes: correct status codes, the `{error: {code, message}}` envelope, response shapes, mutating endpoints return updated portfolio

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
- Fresh start: default watchlist appears (10 tickers), $10k balance shown, prices are streaming
- Watchlist: remove a ticker (down to 9), add a known-universe ticker back to 10; adding an 11th or an unknown symbol is rejected
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears at qty 0, realized P&L reflects the gain/loss
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points (backfilled from history)
- AI chat (mocked): send `buy 5 AAPL`, receive the mock reply, inline trade confirmation appears, cash drops; reload and confirm the transcript repopulates from `/api/chat/history`
- SSE resilience: disconnect and verify reconnection

---

## 13. Decision Log

_Documentation review of 2026-09-07. The questions raised in review have been resolved and folded into Sections 1–12; this log records the decisions and the follow-up work they create._

### Resolved

| # | Decision | Applied in |
|---|----------|------------|
| A1 | The main chart has no historical endpoint — it renders the same SSE-accumulated series as the sparkline, larger; empty on reload | §2, §10 |
| A2 | Daily change % is measured against the **opening price of the US Eastern trading day**, captured by the simulator and carried in the SSE payload | §6, §10 |
| A3 | The P&L chart is backfilled on load from `GET /api/portfolio/history` (`portfolio_snapshots`) | §10 |
| B4 | Watchlist adds are **restricted to the simulator's known universe**; unknown symbols are rejected (`400`) | §6, §8 |
| B5 | The watchlist is capped at **10 tickers**; the default watchlist starts full | §7, §8 |
| B6 | A ticker may be removed from the watchlist even while a position is held | §8 |
| C7 | `avg_cost` = quantity-weighted average on buys; unchanged on sells | §7 |
| C8 | A position row is **deleted** when its quantity reaches 0 | §7 |
| C9 | **Fractional shares are not supported** — quantity is a positive integer everywhere | §7, §9, §10 |
| C10 | **Realized P&L** is tracked (`users_profile.realized_pnl`) and returned by `GET /api/portfolio` | §7, §8 |
| C11 | Standard error envelope `{ "error": { "code", "message" } }`; mutating endpoints return the updated portfolio | §8 |
| D12 | New `GET /api/chat/history`; the transcript repopulates from it on reload | §8, §10 |
| D13 | Any free OpenRouter model — no hardcoded model id; prompt-based-JSON fallback permitted | §3, §9 |
| D14 | LLM context window = last **20 messages** | §9 |
| D15 | Multi-action responses apply in array order; passing actions execute, failing ones are skipped, per-item outcomes are returned | §9 |
| D16 | `watchlist_changes.action` ∈ { `add`, `remove` } | §9 |
| D17 | `chat_messages.actions` JSON shape specified (per-action `status` + `reason`) | §7 |
| D18 | Mock-mode contract specified (regex → fixed-response table) | §9 |
| D19 | The assistant's capability surface is trades + watchlist changes only | §9 |
| E20 | Persistence is a **bind mount** of `db/`, not a named volume | §3, §4, §11 |
| E21 | **Lightweight Charts** for all time-series charts; a small treemap lib (Nivo/visx) for the heatmap | §10 |
| E22 | SSE emits **on price change**, checked ~every 500ms | §6 |
| E23 / E24 | **Massive / Polygon real-data source removed entirely** — the GBM simulator is the only market data source; no `MASSIVE_API_KEY`, no `cerebras` skill | §3, §5, §6, §12 |
| E25 | Timestamps stored in **US Eastern** time; prices / P&L display to 2 dp | §7, §10 |
| E26 | SQLite opened in **WAL mode**, transactions kept short | §7 |
| E27 | Cloud deployments must sit behind access control (no auth by design) | §11 |
| F | Portfolio total value computed client-side; mock mode auto-enabled when no API key; no dedicated 30s snapshot task; root `docker-compose.yml` dropped; start/stop scripts collapsed to `finally.sh` / `finally.ps1`; `user_id` hardcoded to `"default"`; `npm ci` in the Docker build | §4, §5, §7, §9, §10, §11 |

_Review simplification #1 (make the main chart literally the sparkline's data path) was declined — the main chart stays a distinct component, sharing the data but not the rendering._

### Follow-up (outside this document)

- **Removing Massive is not yet reflected elsewhere.** `planning/MARKET_DATA_SUMMARY.md`, `planning/archive/MASSIVE_API.md`, and the backend (`app/market/massive_client.py`, `factory.py`, `tests/market/test_massive.py`, the `massive` dependency in `pyproject.toml`) still carry the Massive integration. Remove them in a dedicated cleanup pass.
- **Opening-price capture is new** relative to the built market-data module, whose `PriceUpdate` is `ticker, price, previous_price, timestamp, change, direction`. The simulator needs a small extension to record each ticker's daily open and expose it on `PriceUpdate` and the SSE payload.
- **Expand the known universe** in `backend/app/market/seed_prices.py` beyond the 10 default tickers so that "add a ticker" is demonstrable within the 10-slot cap.
