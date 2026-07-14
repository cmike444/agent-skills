---
name: tastytrade-api
description: >
  Reference and how-to for the tastytrade Open API — the REST/JSON + WebSocket brokerage
  API for trading equities, options, futures, futures options, and crypto, plus streaming
  market data, account data, balances, positions, transactions, margin, watchlists, and
  backtesting. Use this whenever the task involves the tastytrade (tastyworks) API: building
  or debugging an integration, authenticating with OAuth2, structuring or submitting an order
  (including spreads, iron condors, OTO/OCO/OTOCO brackets), looking up an option chain or
  streamer symbol, reading balances/positions/greeks/IV, connecting to DXLink or the account
  streamer, or resolving errors like 401/422/unconfirmed_user. Trigger on mentions of
  "tastytrade", "tastyworks", "api.tastyworks.com", "DXLink", "tasty API", tastytrade order
  JSON, or tastytrade symbology — even if the user doesn't name a specific endpoint.
---

# tastytrade Open API

The tastytrade Open API is a REST/JSON API (plus two WebSocket streamers) for the tastytrade
brokerage. It covers account data, trading, market data, and real-time streaming across
equities, equity options, futures, futures options, and cryptocurrency.

This file is the map. It holds everything you need for simple lookups and points you to a
`reference/` file when you need endpoint-level detail. Read the relevant reference file before
writing integration code against that area — the request/response shapes have many small
gotchas (dasherized vs. camelCase keys, streamer vs. OCC symbols, string-encoded decimals)
that are easy to get wrong from memory.

## Environments

| Environment | Base URL (REST) | Account streamer (WS) |
|-------------|-----------------|------------------------|
| Production  | `https://api.tastyworks.com` | `wss://streamer.tastyworks.com` |
| Sandbox     | `https://api.cert.tastyworks.com` | `wss://streamer.cert.tastyworks.com` |

Credentials are **separate per environment**. Sandbox resets every 24 hours (trades/positions/
balances cleared; users/accounts kept), quotes are 15-minutes delayed, and a few services are
**not available in sandbox**: net-liq history, market metrics, and real-time (non-delayed)
market data. Hitting production with sandbox credentials (or vice versa) yields
`invalid_credentials`. See [reference/authentication.md](reference/authentication.md) for the
sandbox account/email-confirmation flow.

## REST conventions (read this before any request)

These apply to every REST call and are the most common source of silent failures:

- **User-Agent is mandatory.** Every request must send `User-Agent: <product>/<version>`
  (e.g. `my-client/1.0`). A missing/malformed User-Agent returns a **401** with an nginx HTML
  body even when the token is valid.
- **Headers:** `Content-Type: application/json` and `Accept: application/json`. Optionally
  `Accept-Version: YYYYMMDD` to pin an API version (see Versioning below).
- **Auth:** `Authorization: Bearer <access_token>` on every request (OAuth2 access tokens,
  15-minute lifetime). See [reference/authentication.md](reference/authentication.md).
- **Keys are dasherized** in request and response bodies: `underlying-symbol`, `time-in-force`.
  Two services break this rule and return **camelCase**: the REST **Market Data** endpoint
  (`/market-data/by-type`) and **Net-Liq History**. Don't assume kebab-case everywhere.
- **GET array params** use `key[]=a&key[]=b` (repeat the key). POST/PUT/PATCH/DELETE use a
  dasherized JSON body.
- **Response envelope:** payload is under `data`; list responses nest an `items` array inside
  `data`; a `context` echoes the request path. Errors come back as
  `{"error": {"code": "...", "message": "...", "errors": [...]}}`.

**Versioning:** API versions are dates (`20250904`). Send today's date in `Accept-Version` to
always target the latest; tastytrade routes to the nearest version at or before your target.
Deprecated versions live 9 months. An unknown version returns HTTP **406**.

## Error codes

| Code | Meaning |
|------|---------|
| 400 | Invalid request — missing/invalid params in the body |
| 401 | Missing/expired/invalid token, **or** a bad User-Agent header, or wrong-environment login |
| 403 | Authenticated but not authorized for this resource (e.g. another customer's account) |
| 404 | Endpoint or resource not found |
| 422 | Unprocessable — the action is invalid in context; **order rejections come back as 422** with an `errors` array |
| 406 | Requested `Accept-Version` no longer exists |
| 429 | Rate limited — too many requests too fast |
| 500 | tastytrade server error; returns a support identifier |

Repeated **failed logins block your IP for ~8 hours** (requests time out). Contact
api.support@tastytrade.com to unblock.

## Symbology (quick reference)

Correct symbol formatting is the single biggest trip-up. Full detail + how to look each one up
is in [reference/instruments.md](reference/instruments.md).

| Instrument | Format | Example |
|-----------|--------|---------|
| Equity | Ticker | `AAPL`, `BRK/A` |
| Equity Option | OCC: 6-char space-padded root + `YYMMDD` + `C`/`P` + strike×1000 as 8 digits | `AAPL  230818C00197500` |
| Future | `/` + product code + month code + year digit(s) | `/ESZ2` |
| Future Option | `./` + future contract + space + option root + `YYMMDD` + `C`/`P` + strike | `./ESZ2 EW4U5 221104C91` |
| Cryptocurrency | Pair with `/` | `BTC/USD` |

Month codes: F=Jan G=Feb H=Mar J=Apr K=May M=Jun N=Jul Q=Aug U=Sep V=Oct X=Nov Z=Dec.

**Streamer symbols are different.** DXLink streaming and `get_options_greeks`/quotes use a
`streamer-symbol` (e.g. `.AAPL250117C230`, `/6AM23:XCME`), **not** the OCC/tastytrade symbol.
Fetch it from the instrument or option-chain (`/nested`) response — see
[reference/streaming.md](reference/streaming.md).

## Core mental model: orders, legs, positions

- An **order** trades and manages positions. It has ≤ 4 **legs**; each leg has a `symbol`,
  `quantity`, `instrument-type`, and an `action`.
- `action` = direction + effect: `Buy to Open`, `Sell to Open`, `Buy to Close`,
  `Sell to Close` (futures also accept bare `Buy`/`Sell`). **Open** increases a position;
  **Close** decreases it.
- A **position** has a `symbol`, positive `quantity`, and `direction` (Long/Short). You cannot
  be simultaneously long and short the same symbol; selling through zero flips the direction.
- Leg limits by instrument: Equity / Future / Crypto = **1 leg**; Equity Option / Future
  Option = up to **4 legs**.
- **Always dry-run first.** `POST .../orders/dry-run` runs the exact same body through every
  validation and returns buying-power effect + estimated fees without routing. Check its
  `errors` array (blocking) vs `warnings` (informational).

Full order construction, order types, TIF, complex/bracket orders, and the status lifecycle are
in [reference/orders.md](reference/orders.md).

## Session bootstrap (do this every session)

1. Get an access token via OAuth2 (`POST /oauth/token`, `grant_type=refresh_token`).
2. `GET /customers/me/accounts` to discover account number(s) — use the literal `me` shortcut.
3. Use those account numbers in all `/accounts/{account_number}/...` calls.

For streaming quotes: `GET /api-quote-tokens` → connect to the returned DXLink URL.

## Reference files

Read the file matching the task before writing code against that area.

| File | Covers |
|------|--------|
| [reference/authentication.md](reference/authentication.md) | OAuth2 app creation, personal grant, access/refresh tokens, authorization-code flow, scopes, 2FA, sandbox signup |
| [reference/orders.md](reference/orders.md) | Order JSON (types, TIF, price/effect, legs, actions, advanced/conditional), dry-run, submit, cancel, replace/edit, complex orders (OTO/OCO/OTOCO/PAIRS), order status flow, worked examples |
| [reference/instruments.md](reference/instruments.md) | Instruments (equity/option/future/future-option/crypto/warrant), option chains (full/compact/nested), full symbology, quantity precision |
| [reference/accounts.md](reference/accounts.md) | Customers & accounts, trading status, balances (71 fields), positions, transactions, net-liq history, margin requirements & dry-run, risk/position limits |
| [reference/market-data.md](reference/market-data.md) | REST market data (`by-type`, camelCase, 100-symbol cap), market metrics (IV rank/percentile, liquidity, dividends, earnings), symbol search, market sessions/holidays, quote alerts |
| [reference/streaming.md](reference/streaming.md) | DXLink market-data streaming (token, protocol, events, candles) and the account streamer (connect, heartbeats, order/fill notifications) |
| [reference/watchlists-backtesting.md](reference/watchlists-backtesting.md) | User/public/pairs watchlists, backtesting & simulate-trade |

## Full endpoint index

Every endpoint, grouped. `{acct}` = account number, `{cust}` = customer id (or `me`).

**Auth** — `POST /oauth/token` · `POST /confirmation` (email confirm)

**Accounts & Customers** — `GET /customers/{cust}` · `GET /customers/{cust}/accounts` ·
`GET /customers/{cust}/accounts/{acct}` · `GET /api-quote-tokens`

**Account Status** — `GET /accounts/{acct}/trading-status`

**Balances & Positions** — `GET /accounts/{acct}/balances` · `GET /accounts/{acct}/balances/{currency}` ·
`GET /accounts/{acct}/balance-snapshots` · `GET /accounts/{acct}/positions`

**Net-Liq History** — `GET /accounts/{acct}/net-liq/history`

**Orders** — `GET /accounts/{acct}/orders` · `GET /accounts/{acct}/orders/live` ·
`GET /accounts/{acct}/orders/{id}` · `POST /accounts/{acct}/orders` ·
`POST /accounts/{acct}/orders/dry-run` · `DELETE /accounts/{acct}/orders/{id}` ·
`PUT /accounts/{acct}/orders/{id}` (replace) · `PATCH /accounts/{acct}/orders/{id}` (edit) ·
`POST /accounts/{acct}/orders/{id}/dry-run` · `GET /customers/{cust}/orders` ·
`GET /customers/{cust}/orders/live`

**Complex Orders** — `GET /accounts/{acct}/complex-orders` · `GET /accounts/{acct}/complex-orders/live` ·
`GET /accounts/{acct}/complex-orders/{id}` · `POST /accounts/{acct}/complex-orders` ·
`POST /accounts/{acct}/complex-orders/dry-run` · `PATCH /accounts/{acct}/complex-orders/{id}` (PAIRS) ·
`DELETE /accounts/{acct}/complex-orders/{id}` · `POST /accounts/{acct}/complex-orders/{id}/dry-run`

**Transactions** — `GET /accounts/{acct}/transactions` · `GET /accounts/{acct}/transactions/{id}` ·
`GET /accounts/{acct}/transactions/total-fees`

**Instruments** — `GET /instruments/equities/{symbol}` · `GET /instruments/equities/active` ·
`GET /instruments/equity-options/{symbol}` · `GET /option-chains/{symbol}` ·
`GET /option-chains/{symbol}/compact` · `GET /option-chains/{symbol}/nested` ·
`GET /instruments/futures` · `GET /instruments/futures/{symbol}` · `GET /instruments/future-products` ·
`GET /instruments/future-products/{exchange}/{code}` · `GET /instruments/future-options/{symbol}` ·
`GET /futures-option-chains/{symbol}` · `GET /futures-option-chains/{symbol}/nested` ·
`GET /instruments/future-option-products` · `GET /instruments/future-option-products/{root}` ·
`GET /instruments/future-option-products/{exchange}/{root}` · `GET /instruments/cryptocurrencies` ·
`GET /instruments/cryptocurrencies/{symbol}` · `GET /instruments/warrants` ·
`GET /instruments/warrants/{symbol}` · `GET /instruments/quantity-decimal-precisions`

**Symbol Search** — `GET /symbols/search/{symbol}`

**Market Data (REST)** — `GET /market-data/by-type`

**Market Metrics** — `GET /market-metrics` · `GET /market-metrics/historic-corporate-events/dividends/{symbol}` ·
`GET /market-metrics/historic-corporate-events/earnings-reports/{symbol}`

**Market Sessions** — `GET /market-time/sessions` · `GET /market-time/sessions/current` ·
`GET /market-time/equities/sessions/current` · `.../next` · `.../previous` ·
`GET /market-time/equities/holidays` · `GET /market-time/futures/sessions/current` ·
`GET /market-time/futures/sessions/current/{collection}` · `.../next/{collection}` ·
`.../previous/{collection}` · `GET /market-time/futures/holidays/{collection}`

**Margin & Risk** — `GET /margin/accounts/{acct}/requirements` · `POST /margin/accounts/{acct}/dry-run` ·
`GET /accounts/{acct}/position-limit` ·
`GET /accounts/{acct}/margin-requirements/{underlying}/effective` ·
`GET /margin-requirements-public-configuration` (no auth) · `GET /span/rows`

**Watchlists** — `GET /watchlists` · `GET /watchlists/{name}` · `POST /watchlists` ·
`PUT /watchlists/{name}` · `DELETE /watchlists/{name}` · `GET /public-watchlists` ·
`GET /public-watchlists/{name}` · `GET /pairs-watchlists` · `GET /pairs-watchlists/{name}`

**Quote Alerts** — `GET /quote-alerts` · `POST /quote-alerts` · `DELETE /quote-alerts/{id}`

**Backtesting** — `GET /available-dates` · `GET /backtests` · `POST /backtests` ·
`GET /backtests/{id}` · `POST /backtests/{id}/cancel` · `GET /backtests/{id}/logs` ·
`POST /simulate-trade`

**Streaming (WebSocket, not REST)** — DXLink market data (URL from `/api-quote-tokens`) ·
Account streamer (`wss://streamer[.cert].tastyworks.com`). See
[reference/streaming.md](reference/streaming.md).

## High-value gotchas

- **Don't poll `GET /orders/live`** for order updates — it degrades the platform and can get you
  throttled. Use the **account streamer** for real-time status/fills instead.
- **Multi-leg `price` is the net price** of all legs combined, and **option `price` is per-share**
  (per contract = price × 100). Option `quantity` is **contracts**, not shares.
- **Order marked `Filled` may briefly lack some leg fills** — multi-leg fills publish sequentially;
  re-fetch after a moment for the complete set.
- **Position/quantity decimals are string-encoded** for precision — parse accordingly.
- **REST market data caps at 100 symbols** combined and uses **singular** param names
  (`equity`, not `equities`; `equity-option`, not `equity-options`).
- **Prohibited without the user doing it themselves:** this skill is documentation only. Placing
  live orders, moving money, or changing account settings on a real account are actions the user
  must authorize and perform per the assistant's action rules — dry-run and read endpoints are safe.
