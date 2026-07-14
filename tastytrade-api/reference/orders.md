# Orders

Everything for structuring, submitting, and managing orders. Endpoints are under
`/accounts/{account_number}/`. **Always dry-run before submitting live.**

## Table of contents
- [Order JSON structure](#order-json-structure)
- [Order types](#order-types)
- [Time in force](#time-in-force-tif)
- [Price & price-effect](#price--price-effect)
- [Legs: action, instrument-type, quantity, symbol](#legs)
- [Advanced & conditional instructions](#advanced--conditional-instructions)
- [Endpoints: dry-run, submit, cancel, replace, edit](#endpoints)
- [Complex orders (OTO / OCO / OTOCO / PAIRS)](#complex-orders)
- [Order status lifecycle](#order-status-lifecycle)
- [Response objects](#response-objects)
- [Worked examples](#worked-examples)

## Order JSON structure

An order = order-level attributes + a `legs` array. Minimal shape:

```json
{
  "time-in-force": "Day",
  "order-type": "Limit",
  "price": "1.09",
  "price-effect": "Debit",
  "legs": [ { "action": "...", "symbol": "...", "quantity": 1, "instrument-type": "..." } ]
}
```

Order-level fields:

| Field | Required | Notes |
|-------|----------|-------|
| `order-type` | Yes | See [Order types](#order-types) |
| `time-in-force` | Yes | See [TIF](#time-in-force-tif) |
| `price` | Yes for Limit / Stop Limit | Net price of all legs; **per-share for options** |
| `price-effect` | With `price` | `Credit` or `Debit` |
| `stop-trigger` | Yes for Stop / Stop Limit | Trigger price |
| `value` | Yes for Notional Market | Dollar amount to buy/sell |
| `value-effect` | With `value` | `Credit` or `Debit` |
| `gtc-date` | Yes for GTD | `yyyy-mm-dd` (misnamed — it's the GTD expiry) |
| `legs` | Yes | 1–4 leg objects |
| `rules` | No | Conditional execution (see below) |
| `advanced-instructions` | No | e.g. strict position-effect validation |
| `source` | No | Free-form origin tag |
| `automated-source` | No | `true` for algorithmic orders (affects handling/reporting) |
| `external-identifier` | No | Your own order id |

## Order types

| Type | Rules |
|------|-------|
| `Limit` | Must include `price` + `price-effect`. |
| `Market` | No `price`. **Single leg only.** TIF must not be GTC. Can't open while market closed. |
| `Marketable Limit` | Limit priced to fill immediately. |
| `Stop` | Market order triggered at `stop-trigger`. No `price`. (Value "Stop" = "Stop Market".) Opening stops can't be GTC. |
| `Stop Limit` | Limit triggered at `stop-trigger`. Include both `price`/`price-effect` and `stop-trigger`. |
| `Notional Market` | Buy a **dollar amount**. Single leg, **no `quantity`**, must include `value` + `value-effect`. Crypto and fractional-eligible equities only; can't open while market closed. |

## Time in force (TIF)

| TIF | Behavior |
|-----|----------|
| `Day` | Regular hours; expires at close (3:00pm CT equities, 4:00pm CT futures). |
| `GTC` | Works until filled or canceled. |
| `GTD` | Works until filled or `gtc-date`. Requires `gtc-date`. |
| `Ext` | Extended session (equities only, 6:00am–7:00pm CT). **Single-leg equity only.** |
| `GTC Ext` | Extended session, no expiry. Single-leg equity only. |
| `Ext Overnight` | 24-hour session (7pm–7pm CT). Single-leg equity only. |
| `GTC Ext Overnight` | 24-hour session, no expiry. Single-leg equity only. |
| `IOC` | Immediate-or-cancel. |

Session hours — Equities/options: regular 8:30am–3:00pm CT, extended 6:00am–7:00pm CT, 24-hour
7pm–7pm CT. Futures: 5:00pm–4:00pm CT. Crypto: 24/7.

## Price & price-effect

`price-effect` is the direction of cash relative to your account: **Debit = you pay**,
**Credit = you receive**. It must agree with the action — you can't buy for a credit or sell
for a debit; mismatches are rejected (`cant_buy_for_credit`).

For **multi-leg** orders, `price` is the **net** of all legs (e.g. a credit spread taken in for
$0.85 → `price: 0.85`, `price-effect: Credit`). For **options**, `price` is **per share**; total
cost = `price × quantity × multiplier` (multiplier usually 100). `value`/`value-effect` apply
only to Notional Market and are mutually exclusive with `price`.

## Legs

Each leg is one instrument + one action. Max legs: Equity/Future/Crypto = 1; Equity Option /
Future Option = 4. Each leg's `symbol` must be distinct.

**action** (side + open/close effect):

| Action | Meaning |
|--------|---------|
| `Buy to Open` | Open/increase a long (must not hold a short in that symbol) |
| `Sell to Open` | Open/increase a short (must not hold a long) |
| `Buy to Close` | Reduce a short |
| `Sell to Close` | Reduce a long |
| `Buy` / `Sell` | **Single-leg outright futures only** — system infers open/close from your position |

By default tastytrade auto-corrects a Close action to Open when you have no matching position
(and vice versa). To forbid that, set `advanced-instructions.strict-position-effect-validation:
true` — the order is then rejected if there's no position to close.

**instrument-type** (leg): `Equity`, `Equity Option`, `Future`, `Future Option`,
`Cryptocurrency`. (The Orders spec also lists `Event Contract`, `Fixed Income Security`,
`Liquidity Pool` for completeness.)

**quantity:** positive; whole number except crypto (decimals allowed). **Omit for Notional
Market.** Fractional equity orders need `is-fractional-quantity-eligible: true` on the equity and
a minimum $5 notional. Check `GET /instruments/quantity-decimal-precisions` for increments.

**symbol:** ticker for equities; OCC for equity options; `/`-prefixed for futures; `./`-prefixed
for future options; pair for crypto. See [instruments.md](instruments.md) for how to look each up.

## Advanced & conditional instructions

**advanced-instructions:**
```json
"advanced-instructions": { "strict-position-effect-validation": true }
```

**rules** (conditional/delayed routing — from the Orders spec):

| Field | Meaning |
|-------|---------|
| `route-after` | Earliest datetime to route (delayed submission) |
| `cancel-at` | Auto-cancel datetime |
| `conditions[]` | Price-based triggers |

Each condition: `action` (`route`/`cancel`), `symbol`, `instrument-type`, `indicator`
(`last`/`nat`), `comparator` (`gte`/`lte`), `threshold`, optional `price-components[]` for
synthetic prices.

## Endpoints

| Method & path | Purpose |
|---------------|---------|
| `POST /accounts/{acct}/orders/dry-run` | Validate; returns buying-power effect + fees + `warnings`/`errors`. No routing. |
| `POST /accounts/{acct}/orders` | Submit. Same body as dry-run. Returns order with `id` + status. |
| `GET /accounts/{acct}/orders` | Search (filters below). |
| `GET /accounts/{acct}/orders/live` | All orders created/updated today (any status). **Do not poll** — use the account streamer. |
| `GET /accounts/{acct}/orders/{id}` | Fetch one order. |
| `DELETE /accounts/{acct}/orders/{id}` | Request cancel → status `Cancel Requested`; terminal orders return 422 `cannot_update_order`. |
| `PUT /accounts/{acct}/orders/{id}` | **Replace** (full body; original legs retained). Cancel-replace, atomic. |
| `PATCH /accounts/{acct}/orders/{id}` | **Edit** (partial — only `price`/`order-type`/`time-in-force`). Cancel-replace. |
| `POST /accounts/{acct}/orders/{id}/dry-run` | Dry-run a replace/edit. |
| `GET /customers/{cust}/orders` · `/orders/live` | Orders across accounts; `account-numbers[]` filter. |

**Cancel-replace note:** the replacement sits `Contingent` until the original reaches
`Cancelled`, then routes. If the original fills in between, the replacement is aborted. Only
`price`, `order-type`, and `time-in-force` can change; the rest of the body must match the
original.

**Search filters** (`GET /accounts/{acct}/orders`): `start-date`, `end-date`, `start-at`,
`end-at` (datetime, more precise), `status[]` (e.g. `status[]=Live&status[]=Filled`),
`underlying-symbol`, `underlying-instrument-type` (`Equity` covers equity+option, `Future`
covers future+future-option), `futures-symbol`, `sort` (`Asc`/`Desc`, default `Desc`),
`page-offset`, `per-page`.

## Complex orders

Endpoints under `/accounts/{acct}/complex-orders`. Supported types: **OTOCO, OCO, OTO, PAIRS**.
**BLAST is deprecated / unsupported.** Cancel via the **complex-order id**, not the trigger or
nested order ids.

| Type | Structure | Behavior |
|------|-----------|----------|
| `OTO` | `trigger-order` + `orders[]` (up to 3) | Trigger fills → child orders route. Children don't cancel each other. |
| `OCO` | `orders[]` (2+) | All live at once; one fills → others cancel. All closing orders (need existing position). No trigger. |
| `OTOCO` | `trigger-order` + `orders[]` (a take-profit + a stop) | Trigger fills → the OCO pair activates. Classic bracket. |
| `PAIRS` | `orders[]` + ratio fields | Ratio-threshold pairs trade. |

Submit: `POST /accounts/{acct}/complex-orders`. Dry-run: `.../complex-orders/dry-run`.
Get: `.../complex-orders`, `.../complex-orders/live`, `.../complex-orders/{id}`.
Cancel: `DELETE .../complex-orders/{id}`.
Edit PAIRS threshold: `PATCH .../complex-orders/{id}` with `ratio-price-comparator` (`gte`/`lte`)
+ `ratio-price-threshold`.

The trigger order goes live immediately; the `orders[]` sit in **Contingent** status
(`contingent-status: "Pending Order"`) until the trigger fills. The response returns an `id` for
the whole complex order plus individual `id`s per nested order.

OTOCO body skeleton (full example at bottom):
```json
{ "type": "OTOCO", "trigger-order": { ...opening order... }, "orders": [ {take-profit}, {stop} ] }
```

## Order status lifecycle

Three phases:

- **Submission (non-terminal):** `Received` (accepted, awaiting open market), `Routed` (leaving
  tastytrade), `In Flight` (en route to exchange), `Contingent` (waiting on a related
  order/trigger — replacements and OTO/OTOCO children).
- **Working (non-terminal):** `Live` (at the exchange), `Cancel Requested`, `Replace Requested`.
- **Terminal:** `Filled`, `Cancelled`, `Expired`, `Rejected`, `Removed`, `Partially Removed`.
  (`Dead` and `Remove Pending` also appear in the Orders spec.)

Typical transitions: immediate fill `Received → Routed → In Flight → Live → Filled`; customer
cancel `... → Live → Cancel Requested → Cancelled`; unfilled day order `... → Live → Expired`;
bad symbol `Received → Rejected`.

On fill, tastytrade creates/updates a **position**, updates the **balance**, and writes a **trade
transaction**. An order can be marked `Filled` before every leg fill is published — multi-leg
fills stream sequentially; re-fetch shortly to get all `fills`.

## Response objects

`POST /orders` (and dry-run) return a **PlacedOrderResponse**:

- `order` (or `complex-order`) — full order incl. `id`, `status`, `legs[].fills[]`,
  `cancellable`, `editable`, lifecycle timestamps (`received-at`, `live-at`, `terminal-at`, …).
- `buying-power-effect` — `change-in-buying-power`(+`-effect`), `new-buying-power`,
  `change-in-margin-requirement`, `isolated-order-margin-requirement`, `is-spread`, `impact`.
- `fee-calculation` — `regulatory-fees`, `clearing-fees`, `commission`,
  `proprietary-index-option-fees`, `total-fees` (each with an `-effect`).
- `closing-fee-calculation` — estimated fees to later close the position (informational).
- `warnings` (non-blocking) / `errors` (blocking — order would be rejected).

Each **leg** in a response carries `remaining-quantity` and a `fills[]` array
(`fill-id`, `fill-price`, `quantity`, `filled-at`, `destination-venue`, `ext-exec-id`).

A **rejection** is HTTP 422:
```json
{"error": {"code": "preflight_check_failure", "message": "One or more preflight checks failed",
  "errors": [ {"code": "cant_buy_for_credit", "message": "You cannot buy for a credit."} ] }}
```

## Worked examples

**Equity market (single leg):**
```json
{ "time-in-force": "Day", "order-type": "Market",
  "legs": [ {"instrument-type":"Equity","symbol":"AAPL","quantity":1,"action":"Buy to Open"} ] }
```

**Equity limit close (GTC):**
```json
{ "time-in-force":"GTC","order-type":"Limit","price":150.25,"price-effect":"Credit",
  "legs":[ {"instrument-type":"Equity","symbol":"AAPL","quantity":1,"action":"Sell to Close"} ] }
```

**Bear call (credit) spread — net price, 2 legs:**
```json
{ "time-in-force":"Day","order-type":"Limit","price":0.85,"price-effect":"Credit",
  "legs":[
    {"instrument-type":"Equity Option","symbol":"AAPL  221118C00155000","quantity":1,"action":"Sell to Open"},
    {"instrument-type":"Equity Option","symbol":"AAPL  221118C00157500","quantity":1,"action":"Buy to Open"} ] }
```

**Iron condor (4 legs, net credit):**
```json
{ "order-type":"Limit","time-in-force":"Day","price":"1.51","price-effect":"Credit",
  "legs":[
    {"instrument-type":"Equity Option","symbol":"TSLA  230714P00210000","action":"Buy to Open","quantity":1},
    {"instrument-type":"Equity Option","symbol":"TSLA  230714P00215000","action":"Sell to Open","quantity":1},
    {"instrument-type":"Equity Option","symbol":"TSLA  230714C00282500","action":"Sell to Open","quantity":1},
    {"instrument-type":"Equity Option","symbol":"TSLA  230714C00290000","action":"Buy to Open","quantity":1} ] }
```

**Short futures limit:**
```json
{ "time-in-force":"Day","order-type":"Limit","price":90.03,"price-effect":"Credit",
  "legs":[ {"instrument-type":"Future","symbol":"/CLZ2","quantity":1,"action":"Sell to Open"} ] }
```

**Notional crypto ($10 of BTC):**
```json
{ "time-in-force":"GTC","order-type":"Notional Market","value":10.0,"value-effect":"Debit",
  "legs":[ {"instrument-type":"Cryptocurrency","symbol":"BTC/USD","action":"Buy to Open"} ] }
```

**OTOCO bracket (buy 100 AAPL, then take-profit + stop-loss OCO):**
```json
{
  "type": "OTOCO",
  "trigger-order": {
    "time-in-force":"Day","order-type":"Limit","underlying-symbol":"AAPL","price":157.97,"price-effect":"Debit",
    "legs":[ {"instrument-type":"Equity","symbol":"AAPL","quantity":100,"action":"Buy to Open"} ]
  },
  "orders": [
    { "time-in-force":"GTC","order-type":"Limit","underlying-symbol":"AAPL","price":198.68,"price-effect":"Credit",
      "legs":[ {"instrument-type":"Equity","symbol":"AAPL","quantity":100,"action":"Sell to Close"} ] },
    { "time-in-force":"GTC","order-type":"Stop","underlying-symbol":"AAPL","stop-trigger":143.06,
      "legs":[ {"instrument-type":"Equity","symbol":"AAPL","quantity":100,"action":"Sell to Close"} ] }
  ]
}
```
