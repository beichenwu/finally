# FinAlly Documentation Review — 2026-09-07

## Summary

`planning/PLAN.md` is a mature, internally-consistent specification: the Decision Log
shows most earlier ambiguities were already resolved and folded into Sections 1–12.
The remaining problems cluster in three areas: (1) the live contract between the
already-built market-data module and the plan (SSE payload shape, timestamp format,
opening price, `MASSIVE_API_KEY`) has drifted and is not yet reconciled; (2) several
request/response shapes and enum sets that a coding agent needs are described in prose
but never fully specified (`/api/watchlist`, `/api/portfolio`, error `code` values,
`/api/chat/history`); and (3) the "active ticker set" for streaming/valuation is
under-defined once a position exists for a ticker that is not on the watchlist. None
of these are architectural; they are gaps a focused editing pass can close before
implementation starts.

## Findings

### 1. The "active ticker set" for streaming and valuation is undefined for held-but-unwatched tickers
- **Location:** §6 (SSE Streaming — "the set of tickers is the user's watchlist"), §8 (`DELETE /api/watchlist/{ticker}` — "the position keeps its streamed price"), §10 (Technical Notes — "Portfolio total value is computed client-side: Σ(position qty × latest streamed price) + cash").
- **Issue:** These three statements cannot all hold. If the stream only carries watchlist tickers, then a ticker you still hold after removing it from the watchlist has no streamed price, so the client cannot value it and total value / unrealized P&L silently go wrong. `POST /api/portfolio/trade` also accepts any known-universe ticker (§8 lists only "unknown ticker" as a rejection), so you can open a position in a symbol that was never on the watchlist.
- **Why it matters:** This is the core client-side valuation invariant. Getting it wrong breaks the header total, the positions table, the heatmap weights, and the P&L chart's live extension — exactly the things the E2E "Buy/Sell shares" scenarios assert.
- **Suggested resolution:** State explicitly that the simulator's active set and the SSE stream carry `watchlist ∪ {tickers with a non-zero position}`. On `DELETE /api/watchlist/{ticker}`, only call `source.remove_ticker` when no open position remains (the archived `MARKET_DATA_DESIGN.md` §11 already works this out — pull that rule into PLAN so an agent building from PLAN alone does not miss it). On `POST /api/portfolio/trade` for a symbol not on the watchlist, decide and document whether it is auto-added to the watchlist (and whether it counts against the cap) or merely tracked as a position-only ticker.

### 2. SSE payload contract in PLAN does not match the built module
- **Location:** §6 ("The server emits an event for a ticker **only when its price changes**"; "Each SSE event contains ticker, price, previous price, opening price, timestamp, and change direction") vs. `planning/MARKET_DATA_SUMMARY.md` and `planning/archive/MARKET_DATA_DESIGN.md` §2/§9.
- **Issue:** The built stream emits a **single batched JSON object keyed by ticker** containing **all** tracked tickers whenever the cache version changes (i.e. when *any* ticker moves), as an unnamed `data:` event. Its per-ticker fields are `ticker, price, previous_price, timestamp, change, change_percent, direction` — there is **no `opening_price`**, and `change`/`change_percent` are measured tick-over-tick, not against the day's open. PLAN describes per-ticker, on-change events with an opening price. An agent implementing from PLAN would build a different wire format than the one that already exists and that the frontend spec (§10) targets.
- **Why it matters:** The frontend spec depends on this exact shape (daily change %, sparkline accumulation, flash animation). A mismatch here forces rework on both sides.
- **Suggested resolution:** Pick one wire format and write it out verbatim in §6 — event name (or "unnamed message event"), whether it is batched or per-ticker, and the full key list with types and units. Add `opening_price` to `PriceUpdate` and the payload (the Decision Log Follow-up already flags this), and clarify that `change`/`change_percent` in the payload are tick-over-tick while the UI's "daily change %" is computed frontend-side from `price` and `opening_price`.

### 3. Timestamp format is contradictory
- **Location:** §7 Conventions ("All timestamps are **US Eastern time** (`America/New_York`), ISO 8601") vs. §6 SSE payload ("timestamp") and the built `PriceUpdate.timestamp` (Unix epoch seconds, `float`).
- **Issue:** The plan states a single global timestamp convention, but the market layer uses Unix seconds and the DB uses ISO strings. The SSE "timestamp" field's format is never pinned down. There is also a latent DST ambiguity: storing ISO strings in `America/New_York` **without an explicit UTC offset** makes the fall-back hour (01:00–01:59 ET, twice a year) ambiguous and can corrupt ordering of `trades` / `portfolio_snapshots` / `chat_messages`.
- **Why it matters:** Ordering and "last N" queries (`chat_messages`, snapshots) depend on unambiguous, sortable timestamps; the frontend charts key points by time.
- **Suggested resolution:** Say that price-stream timestamps are Unix epoch seconds (UTC), and that DB-persisted timestamps are ISO 8601 **with an explicit offset** (`2026-09-07T14:32:05-04:00`) rendered in `America/New_York`. Alternatively, store UTC everywhere and convert for display only.

### 4. Opening-price / trading-day rules are under-specified
- **Location:** §6 Opening Price / Daily Change ("the first price it produces on or after 09:30 ET, or the first price after process start if that is later").
- **Issue:** The simulator is a 24/7 in-process task with no concept of market hours. Unspecified: (a) what "the current US Eastern trading day" means on a weekend or US market holiday; (b) when the opening price rolls over to the next day and whether prices "freeze" outside 09:30–16:00 ET; (c) what "daily change %" shows before the first 09:30 boundary of a session (pre-market). The unit test bullet "opening-price capture works" (§12) cannot be written against this as stated.
- **Why it matters:** Daily change % appears on every watchlist row; ambiguous rules mean two agents implement it differently and the test is unfalsifiable.
- **Suggested resolution:** Either (preferred, see Simplification 1) drop the trading-day concept and use a session baseline, or fully specify: open = first price with an ET timestamp ≥ 09:30 on a weekday that is not in a hardcoded US-market-holiday list; it rolls over at the next such instant; before the first open of the process's lifetime, daily change displays `0.00%` (baseline = process-start price). State whether the sim keeps ticking outside market hours (recommended: yes, continuously).

### 5. The known-universe ticker list is never enumerated, and the built code contradicts the restriction
- **Location:** §6 ("Defines a fixed **known universe**... it must contain enough symbols beyond the 10 defaults"), §8 (`POST /api/watchlist` rejects symbols "outside the simulator's known universe"), §12 E2E ("add a known-universe ticker back to 10").
- **Issue:** PLAN never lists the universe, so "add a known-universe ticker" has no canonical symbol for the E2E test to use. The built `seed_prices.py` only contains the 10 defaults, and `GBMSimulator._add_ticker_internal` assigns unknown tickers a **random $50–300 price** (`archive/MARKET_DATA_DESIGN.md` §6.1, `MARKET_SIMULATOR.md`) — directly contradicting a fixed, validated universe. The Decision Log Follow-up notes the expansion is still owed.
- **Why it matters:** Watchlist-add validation, the E2E watchlist scenario, and the simulator's seed data all depend on a concrete list.
- **Suggested resolution:** Enumerate the full universe (e.g. 25–30 symbols) with seed prices and GBM params in `backend/app/market/seed_prices.py`, and reference the count (or the list) in PLAN §6. Remove the random-price fallback path so an out-of-universe ticker is impossible to instantiate.

### 6. `/api/watchlist` and `/api/portfolio` response shapes are not specified
- **Location:** §8 Watchlist / Portfolio tables; §6 ("The opening price is exposed to clients (see SSE payload and `/api/watchlist`)"); §10 (watchlist row and positions table field lists).
- **Issue:** `/api/watchlist` is described only as "current watchlist tickers with latest prices," but §6 says it also carries the opening price, and §10 needs previous price / change to render a row before the first SSE tick. `/api/portfolio` lists the values it returns in prose ("positions, cash balance, realized P&L, total value, unrealized P&L") but gives no JSON structure and no per-position field list (does each position include `current_price`, `unrealized_pnl`, `pct_change`, or must the client derive them?).
- **Why it matters:** `/api/portfolio` is the sole initial-load source for portfolio state (§10) and the shape returned by the mutating endpoints must match it exactly. Frontend and backend agents need the literal schema.
- **Suggested resolution:** Add a concrete JSON example for each of `GET /api/watchlist`, `GET /api/portfolio` (and therefore the `POST /api/portfolio/trade` / `POST /api/chat` `portfolio` field), and `GET /api/portfolio/history`, including every field the corresponding §10 UI element renders.

### 7. Error `code` values are not enumerated
- **Location:** §8 Error Contract (only `INSUFFICIENT_CASH` shown by example; conditions listed in prose).
- **Issue:** The frontend cannot branch on error codes reliably because the canonical set is not given. There is also no code reserved for the archived design's "price not yet available" case (`archive/MARKET_DATA_DESIGN.md` §13.2, which itself disagrees with §10's sketch on 400 vs 404).
- **Why it matters:** Consistent codes are part of the API contract and are referenced by the "the `{error: {code, message}}` envelope" unit test (§12).
- **Suggested resolution:** List the codes and their HTTP status, e.g. `INSUFFICIENT_CASH`, `INSUFFICIENT_SHARES`, `UNKNOWN_TICKER`, `INVALID_QUANTITY`, `WATCHLIST_FULL`, `DUPLICATE_TICKER` → 400; `TICKER_NOT_ON_WATCHLIST` → 404; and decide on `PRICE_UNAVAILABLE` (see Finding 12).

### 8. Ticker case/whitespace normalization is specified for only one endpoint
- **Location:** §8 (`POST /api/watchlist` — "Symbol is upper-cased and trimmed"); §9 mock contract regex `/buy (\d+) ([A-Z]+)/i`.
- **Issue:** `POST /api/portfolio/trade` and LLM/mock-supplied trades never state that the ticker is upper-cased/trimmed. The mock regex uses the `/i` flag, so `buy 5 aapl` matches and yields `ticker: "aapl"`, which then fails universe validation unless something normalizes it.
- **Why it matters:** Silent, inconsistent normalization produces "unknown ticker" rejections that look like bugs, and makes the mock contract order-dependent on input casing.
- **Suggested resolution:** State once, in §7 Conventions or §8, that every ticker received by any endpoint or by the LLM executor is `.strip().upper()`-ed before validation.

### 9. Realized P&L has no home in the UI
- **Location:** §8 (`GET /api/portfolio` returns realized P&L), §12 E2E ("realized P&L reflects the gain/loss") vs. §10 (Header lists only "portfolio total value... cash balance"; positions table lists only "unrealized P&L, % change").
- **Issue:** Realized P&L is stored, computed, and returned, and an E2E test asserts it is visible after a sell — but no §10 UI element displays it.
- **Why it matters:** The E2E scenario is not implementable against the current frontend spec.
- **Suggested resolution:** Add realized P&L (and ideally total P&L = realized + unrealized) to the header or to a portfolio-summary area in §10.

### 10. `portfolio_snapshots` behavior when the table is empty is undefined
- **Location:** §7 (`portfolio_snapshots` — "lazily appended when `GET /api/portfolio/history` is requested if more than 30s have elapsed since the last snapshot"), §12 E2E ("P&L chart has data points (backfilled from history)").
- **Issue:** "Since the last snapshot" is undefined when there are zero snapshots. On a fresh start with no trades, `GET /api/portfolio/history` may return an empty array, so the P&L chart has nothing to backfill and the E2E assertion fails.
- **Why it matters:** Directly breaks a listed E2E scenario on the default first-run state.
- **Suggested resolution:** Write one snapshot at DB seed/init time, and specify that `GET /api/portfolio/history` always appends a snapshot when the table is empty (and otherwise when >30s since the most recent).

### 11. `GET /api/chat/history` response shape and ordering are unspecified
- **Location:** §8 Chat table ("Full stored conversation"), §10 (AI chat panel — "the transcript is repopulated from `GET /api/chat/history`... Trade executions and watchlist changes shown inline as confirmations").
- **Issue:** No element shape (is it `{id, role, content, actions, created_at}`?), no ordering guarantee, no statement about whether `actions` is returned parsed or as a JSON string, and no limit/pagination even though a long session can accumulate thousands of rows.
- **Why it matters:** The frontend must reconstruct inline action confirmations from this payload; it needs the exact structure.
- **Suggested resolution:** Specify the array element shape (mirroring `chat_messages` minus `user_id`), ascending `created_at` order, `actions` returned as parsed JSON or `null`, and whether any cap applies.

### 12. Trade against a valid ticker with no cached price is unhandled in PLAN
- **Location:** §8 (`POST /api/portfolio/trade`), §6 (trade execution reads from the cache).
- **Issue:** If a universe ticker was just added and the simulator has not produced a tick yet, `price_cache.get_price` returns `None`. PLAN says nothing; the archived design is self-contradictory (400 in §13.2, 404 in §10).
- **Why it matters:** This is a real race on "add ticker then immediately buy," which the demo and tests will hit.
- **Suggested resolution:** Require the simulator to seed the cache with the ticker's seed price synchronously inside `add_ticker` (the archived `SimulatorDataSource.add_ticker` already does this — state it as a requirement), and define a `PRICE_UNAVAILABLE` → 400 error for the residual case.

### 13. `MASSIVE_API_KEY` still drives source selection in the built code
- **Location:** §5 ("Market data always comes from the built-in simulator — there is no external provider and no env var to configure it"), §6, Decision Log E23/E24 and Follow-up vs. `MARKET_DATA_SUMMARY.md` and `archive/MARKET_DATA_DESIGN.md` §8 (`factory.py` returns `MassiveDataSource` when `MASSIVE_API_KEY` is set).
- **Issue:** The factory's behavior is a live contradiction of §5. The Decision Log Follow-up acknowledges the cleanup is owed, but until it is done, `MARKET_DATA_SUMMARY.md` (which CLAUDE.md points agents to as the market-data source of truth) still documents a Polygon/Massive path.
- **Why it matters:** An agent consulting `MARKET_DATA_SUMMARY.md` will believe real market data is a supported mode.
- **Suggested resolution:** Prioritize the cleanup pass (remove `massive_client.py`, the factory branch, `test_massive.py`, the `massive` dependency) and rewrite `MARKET_DATA_SUMMARY.md` to describe the simulator as the only source. Until then, add a one-line "superseded" banner to `MARKET_DATA_SUMMARY.md`.

### 14. `trades` table has no reader
- **Location:** §7 (`trades` — "Trade history (append-only log)"); §8 (no endpoint exposes it); §10 (no trade-history / blotter UI element).
- **Issue:** Trades are written but never read anywhere in the spec. Filled trades are also recorded (with price and status) in `chat_messages.actions` for LLM-driven ones, but manual trades have no surfaced history at all.
- **Why it matters:** Either the plan is missing a feature (a trade blotter, common in a "Bloomberg-like" terminal) or the table is dead weight.
- **Suggested resolution:** Decide: add `GET /api/trades` plus a positions-adjacent "recent trades" panel in §10, or drop the `trades` table and note that trade history is out of scope.

### 15. Connection status indicator has three states but only one transition is testable
- **Location:** §2 / §10 (dot: green = connected, yellow = reconnecting, red = disconnected), §12 E2E ("SSE resilience: disconnect and verify reconnection").
- **Issue:** Nothing defines what drives the yellow ("reconnecting") vs. red ("disconnected") states given `EventSource` only exposes `onopen`/`onerror` and a `readyState`. The E2E test only checks that reconnection happens, not that the indicator reflects each state.
- **Why it matters:** "yellow = reconnecting" is not directly observable from the `EventSource` API without a heuristic (e.g. `readyState === CONNECTING` after an error → yellow; N seconds without an open → red).
- **Suggested resolution:** Define the mapping from `EventSource` state/events to the three dot colors, and add a frontend unit test asserting each mapping.

### 16. Chat-driven watchlist removal cannot be tested in mock mode
- **Location:** §9 (`watchlist_changes.action` ∈ {`add`, `remove`}; mock contract table has `buy`, `sell`, `watch` patterns only), §12 (E2E runs with `LLM_MOCK=true`).
- **Issue:** There is no mock pattern that produces a `{"action": "remove"}` watchlist change, so the `remove` branch of chat action handling has no deterministic E2E path.
- **Why it matters:** Half of the watchlist-management capability the AI advertises is untested end-to-end.
- **Suggested resolution:** Add an `/unwatch ([A-Z]+)/i` (or `/(?:unwatch|remove) ([A-Z]+)/i`) row to the mock contract and a corresponding E2E assertion.

### 17. Additional Testing Strategy gaps
- **Location:** §12.
- **Issue:** No coverage is listed for: (a) opening price → daily-change-% computation on the SSE payload / frontend; (b) the `watchlist ∪ positions` active-set behavior (Finding 1); (c) `DELETE /api/watchlist/{ticker}` returning 404 for a ticker not on the list; (d) `GET /api/portfolio`'s computed `total_value`/`unrealized_pnl` matching the client-side formula in §10; (e) the prompt-based-JSON fallback / tolerant parser path (§9); (f) the "last 20 messages" context truncation (§9); (g) heatmap rectangle *sizing* by weight (only color is asserted).
- **Why it matters:** Each of these is a behavior the plan explicitly specifies; untested, they will drift.
- **Suggested resolution:** Add one unit or E2E case per item.

### 18. `.env` handling: placeholder key defeats auto-mock; file-vs-env-var wording is loose
- **Location:** §5 (mock mode when `OPENROUTER_API_KEY` "is absent or empty"; example shows `OPENROUTER_API_KEY=your-openrouter-api-key-here` and `LLM_MOCK=false`; "The backend reads `.env` from the project root"), §11 (`docker run ... --env-file .env`).
- **Issue:** (a) The committed `.env.example` value is a non-empty placeholder, so a user who copies it verbatim gets **live** mode with an invalid key and a failing first chat, instead of mock mode. (b) `--env-file` injects environment variables; it does not place a `.env` file in the container, so "reads `.env` from the project root" is misleading for the Docker path.
- **Why it matters:** First-run experience for a student without an OpenRouter key should be zero-friction mock mode; the current example breaks that.
- **Suggested resolution:** Ship `.env.example` with an empty `OPENROUTER_API_KEY=` (and `LLM_MOCK=false`), or treat a recognized placeholder string as empty. Reword §5 to "the backend reads configuration from environment variables; the wrapper scripts pass them via `--env-file .env`, and local dev may use a `.env` file."

### 19. Minor: `/api/health` response and semantics unspecified
- **Location:** §8 System table, §11 (health check "for Docker/deployment").
- **Issue:** No response body, status semantics, or statement of what it checks (process only? DB reachable? simulator task alive?). No `HEALTHCHECK` is mentioned in the Dockerfile description.
- **Why it matters:** A health check that only proves the process is up will not catch a dead simulator task or a locked DB.
- **Suggested resolution:** Specify `200 {"status": "ok"}` when the DB is reachable and the market-data task is running; `503` otherwise. Note whether the Dockerfile includes a `HEALTHCHECK`.

## Simplification Opportunities

### 1. Replace "opening price of the US Eastern trading day" with a session baseline
- **Location:** §6 Opening Price / Daily Change, §10, Decision Log A2.
- **Opportunity:** The simulator is not a real market and has no trading day. Capturing the 09:30-ET open requires timezone handling, a market-holiday calendar, a daily rollover, and pre-market special-casing (Finding 4) — all for a cosmetic percentage. Defining the baseline as "first price observed after process start" (persisted so it survives SSE reconnects) removes every one of those mechanisms. Relabel the UI metric "Change" (session-to-date) instead of "Daily change." A2 chose the trading-day open; it is worth reopening because the cost/benefit is poor for a demo.
- **Trade-off:** The percentage resets whenever the container restarts rather than at 09:30 ET. For a single-user demo that is acceptable and arguably clearer.

### 2. Drop the 30-second lazy snapshot append
- **Location:** §7 `portfolio_snapshots`, §8 `GET /api/portfolio/history`.
- **Opportunity:** The P&L chart is already "extended live" client-side from the price stream (§10), so persisted snapshots only need to exist at points the client cannot reconstruct after a reload — i.e. at trade times. Write a snapshot on each trade plus one at seed/init, and make `GET /api/portfolio/history` a pure read. This removes clock-dependent side effects from a GET handler (which also simplifies testing) and resolves Finding 10.
- **Trade-off:** With very few trades the persisted curve is sparse between reloads; the live client-side extension covers the recent segment, so the visible chart is still continuous during a session.

### 3. Have `GET /api/portfolio` return only stored state; let the client compute valuation
- **Location:** §8 (`GET /api/portfolio` returns `total value` and `unrealized P&L`), §10 (client computes total value from the stream anyway).
- **Opportunity:** The client already owns the one true valuation path after load. Returning server-computed `total_value` / `unrealized_pnl` creates a second path that can disagree (rounding, cache vs. stream timing) and a test obligation (Finding 17d). Return only `cash_balance`, `realized_pnl`, and positions (`ticker, quantity, avg_cost`); the client derives the rest. The server still computes `total_value` internally for snapshots, but that value never needs to match the client tick-for-tick.
- **Trade-off:** The UI shows `$0` unrealized P&L for a few hundred ms after load until the first SSE frame arrives. Acceptable, and can be masked with a skeleton state.

### 4. Consider a single `/api/bootstrap` instead of separate initial-load GETs
- **Location:** §8 (`GET /api/portfolio`, `GET /api/watchlist`, `GET /api/portfolio/history`, `GET /api/chat/history`).
- **Opportunity:** On load the frontend calls four GETs to hydrate. A single `GET /api/bootstrap` returning `{portfolio, watchlist, history, chat}` cuts round-trips and gives one place to define the initial-state contract (addresses Finding 6). The individual endpoints can remain for the mutating flows that already return `portfolio`.
- **Trade-off:** One more endpoint to document; slightly less RESTful. Net simpler for the client.

### 5. Drop `previous_price` (and possibly `change`/`change_percent`) from the SSE payload
- **Location:** §6 SSE event fields; built payload keys `previous_price, change, change_percent, direction`.
- **Opportunity:** The client necessarily retains the last price it received per ticker, so `previous_price` is redundant, and `change`/`direction` are one subtraction away. Sending `ticker, price, opening_price, timestamp` is enough for the flash animation and daily-change math. Smaller payload, fewer fields to keep consistent.
- **Trade-off:** The very first frame after connect has no prior price client-side, so the first flash/direction is suppressed until the second frame — a non-issue visually.

### 6. Reduce the default watchlist to 8 tickers
- **Location:** §7 Default Seed Data (10 defaults, cap 10), §12 E2E watchlist scenario.
- **Opportunity:** With the default list full at the cap, the manual "add a ticker" flow only works after removing one, and the E2E script has to remove-then-add. Seeding 8 of the 10 lets "add a ticker" work on a fresh start and simplifies the demo and the test.
- **Trade-off:** Two fewer tickers visible on first launch; the cap and universe still exercise the rejection paths.

### 7. Collapse `direction` into a client-side derivation
- **Location:** §6, built `PriceUpdate.direction`.
- **Opportunity:** Related to Simplification 5 — `direction` is fully determined by `price` vs. the client's last price. Keeping it server-side is a second encoding of the same fact. If the batched-snapshot format is kept, `direction` computed against the *cache's* previous tick can even disagree with what the client last rendered (if the client missed a frame). Deriving it client-side is more consistent.
- **Trade-off:** Minimal client code; removes a field.

## Open Questions

1. Should buying a known-universe ticker that is not on the watchlist auto-add it to the watchlist, and if so, does a position-only ticker count against the 10-slot cap? (Finding 1/2.)
2. Does the simulator model market hours at all, or tick continuously 24/7? If it models hours, do prices freeze outside 09:30–16:00 ET and on weekends/holidays? (Finding 4.)
3. What is the canonical known-universe list and its seed prices / GBM parameters beyond the current 10? (Finding 5.)
4. Is a trade-history / blotter view intended for this build, or is `trades` write-only for now? (Finding 14.)
5. Should persisted timestamps be UTC or `America/New_York`, and must the stored ISO strings carry an explicit offset to disambiguate the DST fall-back hour? (Finding 3.)
6. "Any free OpenRouter model" — is there a preferred model plus an ordered fallback list if the first choice is unavailable or rate-limited at runtime, or is a single hardcoded-at-config-time choice acceptable? (§9, Decision Log D13.)
7. Is transcript growth bounded? A long-running single session could accumulate thousands of `chat_messages` rows that `GET /api/chat/history` returns in full on every reload. (Finding 11.)
8. On first launch with an empty portfolio, what should the heatmap, positions table, P&L chart, and main chart render before any position/trade/selection exists? (No empty-state guidance in §10.)
9. Which watchlist ticker (if any) is selected in the main chart area on initial load? (§2 "Click a ticker"; §10 does not name a default.)
