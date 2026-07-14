# Accounts, Balances, Positions, Transactions, Margin

Account-scoped read endpoints. Discover accounts first, then everything hangs off
`/accounts/{account_number}/`.

## Table of contents
- [Customers & accounts](#customers--accounts)
- [Trading status](#trading-status)
- [Balances](#balances)
- [Positions](#positions)
- [Transactions](#transactions)
- [Net-liq history](#net-liq-history)
- [Margin requirements & dry-run](#margin-requirements--dry-run)
- [Risk parameters & position limits](#risk-parameters--position-limits)

## Customers & accounts

Use the literal `me` for the customer id — no need to know the internal id.

- `GET /customers/me` — full customer profile (identity, contact, tax/citizenship, affiliations,
  suitability, agreements). `allow-missing=true` returns partials. `is-professional` and
  `has-delayed-quotes` drive market-data tier.
- `GET /customers/me/accounts` — **session bootstrap.** Returns `AccountAuthorityDecorator`
  items: an `account` object + `authority-level` (`owner`/`power-of-attorney`/`custodian`).
- `GET /customers/me/accounts/{acct}` — one account.

`Account` fields: `account-number`, `account-type-name` (`Individual`/`Joint`/`IRA`/`Entity`),
`nickname`, `margin-or-cash`, `is-closed`, `opened-at`, `funding-date`, `day-trader-status`,
`is-futures-approved`, `suitable-options-level`, `investment-objective`, `risk-tolerance`.

## Trading status

`GET /accounts/{acct}/trading-status` → `TradingStatus`. Use it to gate trades before submitting.

Key fields:
- **State:** `is-closed`, `is-frozen`, `is-closing-only`, `is-risk-reducing-only`.
- **Options:** `options-level`, `short-calls-enabled`, plus restriction flags
  (`are-far-otm-net-options-restricted`, `are-options-values-restricted-to-nlv`, …).
- **Margin/equities:** `equities-margin-calculation-type` (e.g. `Reg-T`),
  `is-portfolio-margin-enabled`, `is-in-margin-call`, `has-intraday-equities-margin`.
- **Day trading:** `is-pattern-day-trader`, `day-trade-count` (rolling 5-day),
  `is-in-day-trade-equity-maintenance-call`, `pdt-reset-on`.
- **Futures:** `is-futures-enabled`, `is-futures-closing-only`, `is-futures-intra-day-enabled`,
  `futures-margin-rate-multiplier`, `is-small-notional-futures-intra-day-enabled`.
- **Crypto:** `is-cryptocurrency-enabled`, `is-cryptocurrency-closing-only`.
- **Fees:** `fee-schedule-name`.

Only fields relevant to the account's config are returned.

## Balances

- `GET /accounts/{acct}/balances` — array of `AccountBalance` (one per currency, usually USD).
- `GET /accounts/{acct}/balances/{currency}` — one currency.
- `GET /accounts/{acct}/balance-snapshots` — historical snapshots. Filters: `snapshot-date`,
  `start-date`, `end-date`, `time-of-day` (`BOD`/`EOD`), `currency`, pagination.

`AccountBalance` is a 71-field object. Headline / most-used fields:

| Field | Meaning |
|-------|---------|
| `net-liquidating-value` | Primary account value = cash + longs − shorts |
| `cash-balance` | Total cash |
| `cash-available-to-withdraw` | Withdrawable without liquidating |
| `equity-buying-power` | Buying power for stock |
| `derivative-buying-power` | Buying power for options |
| `day-trading-buying-power` | Intraday BP (≈4× margin equity for PDT) |
| `maintenance-requirement` | Total maintenance margin |
| `maintenance-excess` | Cushion above maintenance — **negative = margin call** |
| `margin-equity` | NLV minus non-margineable assets |
| `maintenance-call-value` / `reg-t-call-value` / `day-equity-call-value` | Specific call amounts |
| `long-equity-value`, `short-equity-value`, `long-derivative-value`, … | Per-asset-class market values |
| `pending-cash` (+ `-effect`) | Unsettled/pending cash |

Many fields have a paired `*-effect` (`Debit`/`Credit`/`None`): Debit reduces, Credit increases.

## Positions

`GET /accounts/{acct}/positions` → `CurrentPosition[]`. Filters: `symbol`, `underlying-symbol[]`,
`instrument-type`, `underlying-product-code`, `include-closed-positions`, `include-marks`,
`net-positions`.

Fields: `symbol` (OCC for options, `/…` for futures), `underlying-symbol`, `instrument-type`,
`streamer-symbol`, `quantity` (**string-encoded decimal**), `quantity-direction`
(`Long`/`Short`), `multiplier` (1 equity, 100 standard option), `average-open-price` (cost basis
per unit), `close-price`, `mark-price` (per unit), `mark` (total = mark-price × qty × multiplier),
`cost-effect`, `realized-day-gain` (+`-effect`), `realized-today`, `expires-at`, `is-frozen`,
`created-at`, `updated-at`, `order-id`.

Unrealized P&L per position ≈ (`mark-price` − `average-open-price`) × qty × multiplier (sign by
direction). Use the returned `multiplier`, don't assume 100.

## Transactions

- `GET /accounts/{acct}/transactions` → `Transaction[]`. The definitive ledger — trades,
  dividends, interest, fees, transfers, assignments, expirations.
- `GET /accounts/{acct}/transactions/{id}` — one transaction.
- `GET /accounts/{acct}/transactions/total-fees?date=YYYY-MM-DD` — day's total fees (defaults
  today).

Filters: `start-date`/`end-date`, `start-at`/`end-at`, `symbol`, `underlying-symbol`,
`futures-symbol`, `instrument-type`, `type`, `types[]`, `sub-type[]`, `action`, `sort`,
`currency`, `page-offset`, `per-page` (default 250, **max 2000**).

Key fields: `transaction-type` (`Trade`/`Receive Deliver`/`Dividend`/`Money Movement`/…),
`transaction-sub-type` (`Sell to Open`/`Assignment`/`Expiration`/`Dividend`/…), `action`,
`quantity`, `price`, `value` (+`-effect`), `net-value` (+`-effect`, **value after fees — use for
P&L**), `commission`, `clearing-fees`, `regulatory-fees`, `proprietary-index-option-fees` (each
+`-effect`), `is-estimated-fee`, `order-id`, `executed-at`, `lots`,
`cost-basis-reconciliation-date`. Cost basis can lag a day due to nightly reconciliation.

## Net-liq history

`GET /accounts/{acct}/net-liq/history` → OHLC candles of net-liq. **camelCase fields** and
**not available in sandbox.**

Params: `time-back` (`1d`/`1w`/`1m`/`3m`/`6m`/`1y`/`all`) **or** `start-time`/`end-time`
(ISO-8601 zoned, e.g. `2026-01-01T00:00:00+00:00[UTC]`), optional `interval`.

`NetLiqOhlc`: `open`/`high`/`low`/`close` (net-liq), `totalOpen`…`totalClose` (net-liq + pending
cash), `pendingCashOpen`…`pendingCashClose`, `time`. For a value line chart use `close`.

## Margin requirements & dry-run

- `GET /margin/accounts/{acct}/requirements` — current margin/capital report across positions.
- `POST /margin/accounts/{acct}/dry-run` — margin impact of a prospective order **without placing
  it**. Body is an order object (numeric values as **strings**), requires `account-number` +
  `underlying-symbol` + `order-type` + `time-in-force` + `legs`. Focuses on capital impact
  (buying power, initial/maintenance) rather than order validation. Use it for position sizing and
  strategy capital-efficiency comparisons.

## Risk parameters & position limits

- `GET /accounts/{acct}/position-limit` → `PositionLimit`: per-order and per-position size caps by
  instrument type (`equity-order-size`, `equity-option-position-size`, `future-order-size`, …,
  `underlying-opening-order-limit`).
- `GET /accounts/{acct}/margin-requirements/{underlying}/effective` → `MarginRequirement`:
  per-symbol rates as decimals (`long-equity-initial`, `short-equity-maintenance`,
  `naked-option-standard`/`-minimum`/`-floor`, …). `0.50` = 50%.
- `GET /margin-requirements-public-configuration` — **no auth**; returns `risk-free-rate`
  (useful for options pricing models too).
- `GET /span/rows?date=YYYY-MM-DD&exchange=CME|CFE` — raw SPAN rows for futures margin
  (`page-offset`, `per-page` up to 50000).
