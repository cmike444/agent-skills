# Watchlists & Backtesting

Two smaller feature areas. Both are **user-scoped**, not account-scoped.

## Watchlists

User watchlists are tied to the authenticated user. `PUT` is a **full replacement** — to add one
symbol you must resend all existing entries plus the new one.

### User watchlists
- `GET /watchlists` — all of the user's watchlists.
- `GET /watchlists/{name}` — one.
- `POST /watchlists` → 201. Body:
  ```json
  {
    "name": "AI Infrastructure",
    "group-name": "Thematic ETFs",
    "order-index": 9999,
    "watchlist-entries": [
      { "symbol": "VRT", "instrument-type": "Equity" },
      { "symbol": "DELL", "instrument-type": "Equity" }
    ]
  }
  ```
  `name` (unique) and `watchlist-entries` are required; each entry needs `symbol` (+ optional
  `instrument-type`). `group-name` and `order-index` (default 9999) are optional.
- `PUT /watchlists/{name}` — full replacement (same body as POST).
- `DELETE /watchlists/{name}` — delete.

### Public & pairs watchlists (read-only)
- `GET /public-watchlists` — tastytrade-curated lists; `counts-only=true` returns just names +
  counts.
- `GET /public-watchlists/{name}`.
- `GET /pairs-watchlists` · `GET /pairs-watchlists/{name}` — symbol pairs for pairs trading;
  each `PairsWatchlist` has `name`, `pairs-equations`, `order-index`.

`Watchlist` fields: `name`, `watchlist-entries`, `group-name`, `order-index`, `cms-id` (public).

Common use: build a portfolio/screen watchlist, then batch it through Market Data or Market
Metrics to find (e.g.) high-IVR candidates.

## Backtesting

Run historical options-strategy backtests. **User-scoped** and **asynchronous** — create returns
an id; poll for results. The swagger doesn't define full request/response schemas; consult
developer.tastytrade.com for exact strategy parameters.

| Endpoint | Purpose |
|----------|---------|
| `GET /available-dates` | Available historical date ranges per symbol — check before creating. |
| `GET /backtests` | IDs of the user's backtests. |
| `POST /backtests` | Create + start. Body = strategy config (underlying, date range, entry/exit rules, sizing). |
| `GET /backtests/{id}` | Full results incl. status + params. Poll this for completion. |
| `POST /backtests/{id}/cancel` | Cancel a running backtest. |
| `GET /backtests/{id}/logs` | Step-by-step execution logs (debugging). |
| `POST /simulate-trade` | One-off historical pricing for a hypothetical trade — no full backtest. |

Typical flow: `GET /available-dates` → `POST /backtests` → poll `GET /backtests/{id}` until done
(cancel if needed). Use `POST /simulate-trade` for a quick "what did this cost on date X?" lookup.
