# Instruments & Symbology

Instrument definitions and option chains across all asset classes, plus the full symbology
reference and how to resolve each symbol. You only need a symbol to trade, but instrument JSON
carries useful flags (`is-fractional-quantity-eligible`, `streamer-symbol`, `expiration-date`,
`lendability`, tick sizes, etc.).

## Table of contents
- [Symbology reference](#symbology-reference)
- [Streamer symbols](#streamer-symbols)
- [Equities](#equities)
- [Equity options & chains](#equity-options--chains)
- [Futures](#futures)
- [Futures options & chains](#futures-options--chains)
- [Cryptocurrency & warrants](#cryptocurrency--warrants)
- [Quantity decimal precision](#quantity-decimal-precision)
- [Key data-model fields](#key-data-model-fields)

## Symbology reference

| Type | Format | Example |
|------|--------|---------|
| Equity | Alphanumeric ticker (occasional `/`) | `AAPL`, `BRK/A` |
| Equity Option (OCC) | 6-char space-padded root + `YYMMDD` + `C`/`P` + strike×1000 as 8 zero-padded digits | `AAPL  220617P00150000` |
| Future | `/` + product code + month code + 1–2 year digits | `/ESZ2`, `/CLZ2`, `/VXX22` |
| Future Option | `./` + future contract + space + option-product code + `YYMMDD` + `C`/`P` + strike | `./CLZ2 LO1X2 221104C91` |
| Cryptocurrency | Pair with `/` | `BTC/USD`, `BCH/USD` |
| Warrant | Ticker (often ends in `W`) | `RGTIW` |

**Month codes:** F=Jan G=Feb H=Mar J=Apr K=May M=Jun N=Jul Q=Aug U=Sep V=Oct X=Nov Z=Dec.

**OCC strike:** first 5 digits = whole dollars, last 3 = decimals. `00197500` = 197.500.
`01050550` = 1050.55. `00000500` = 0.50.

**Future options** have a unique product code per expiration series. Monthly CL options = `LO`;
weekly CL = `LO1`–`LO5` (first–fifth Friday). ES weekly Monday options = `E1A`–`E5A`. A future
option symbol carries both the future contract code and the option product code:
`./ESZ2 E1AZ2 221205P3720` = a 2022-12-05 3720 put on the Dec ES future. Treat a future option
product's `root-symbol` (not `code`) as its primary identifier.

## Streamer symbols

Streaming (DXLink) and REST quote/greeks lookups use a **`streamer-symbol`**, which differs from
the OCC/tastytrade symbol (e.g. `.AAPL250117C230`, `/6AM23:XCME`). It's returned in the
`streamer-symbol` field of instrument and nested-chain responses. Endpoints that include it:
`GET /instruments/cryptocurrencies`, `GET /instruments/equities/{symbol}`,
`GET /instruments/futures`, `GET /futures-option-chains/{code}/nested`,
`GET /option-chains/{symbol}/nested`.

## Equities

- `GET /instruments/equities/{symbol}` — one equity. Fields: `symbol`, `instrument-type`,
  `instrument-sub-type` (`Common Stock`, `ADR`…), `description`, `active`, `is-etf`, `is-index`,
  `is-fractional-quantity-eligible`, `is-closing-only`, `is-options-closing-only`, `is-illiquid`,
  `listed-market`, `streamer-symbol`, `lendability` (`Easy To Borrow`/`Locate Required`/
  `Preborrow`), `borrow-rate`, `halted-at`, `stops-trading-at`, `tick-sizes`, `option-tick-sizes`.
- `GET /instruments/equities/active` — all active equities, paginated (`page-offset`, `per-page`,
  `lendability`).
- Multi-fetch by symbol: `GET /instruments/equities?symbol[]=AAPL&symbol[]=TSLA`.

## Equity options & chains

- `GET /instruments/equity-options/{symbol}` — one option (OCC symbol); `active` query filter.
- Chains for an underlying:
  - `GET /option-chains/{symbol}/nested` — **preferred**: grouped by expiration, then strike with
    `call`/`put` symbols and `streamer-symbol`. Compact and UI-friendly.
  - `GET /option-chains/{symbol}/compact` — delimited strings of all `symbols` and
    `streamer-symbols`; smallest payload.
  - `GET /option-chains/{symbol}` — full `EquityOption` objects for every contract (large).

`EquityOption` fields: `symbol`, `underlying-symbol`, `root-symbol` (differs for adjusted options
like `SPXW`), `option-type` (`C`/`P`), `strike-price`, `expiration-date`, `expiration-type`
(`Regular`/`Weekly`/`Quarterly`/`End of Month`), `expires-at`, `exercise-style`
(`American`/`European`), `settlement-type` (`Physical`/`Cash`), `shares-per-contract` (usually
100), `days-to-expiration`, `active`, `streamer-symbol`.

## Futures

- `GET /instruments/futures` — outright contracts. Filter with `product-code[]=ES&product-code[]=CL`,
  `symbol[]=/ESM6`, `exchange`, `only-active-futures`, pagination. Each item includes
  `streamer-symbol` and `streamer-exchange-code`.
- `GET /instruments/futures/{symbol}` — one contract (e.g. `/ESM6`).
- `GET /instruments/future-products` — product families (not tradeable contracts).
- `GET /instruments/future-products/{exchange}/{code}` — e.g. `CME/ES`. Exchanges:
  `CME`, `CFE`, `CBOED`, `SMALLS`.

You trade a **specific contract** (`/CLU3`), not a product (`/CL`). `Future` fields include
`product-code`, `exchange`, `expiration-date`, `expires-at`, `last-trade-date`,
`first-notice-date`, `active-month`/`next-active-month`, `contract-size`, `notional-multiplier`,
`tick-size`, `roll-target-symbol`, `streamer-symbol`. `FutureProduct` includes `listed-months`,
`active-months`, `notional-multiplier`, `market-sector`, `cash-settled`, `small-notional`.

## Futures options & chains

- `GET /instruments/future-options/{symbol}` — one future option.
- `GET /futures-option-chains/{product_code}/nested` — **preferred**: grouped by underlying future
  and expiration with `call`/`put` + `streamer-symbol`. (`{product_code}` = the future product,
  e.g. `ES`.)
- `GET /futures-option-chains/{product_code}` — full objects (large).
- Multi-fetch: `GET /instruments/future-options?symbol[]=...&symbol[]=...` or filter one series
  with `?option-root-symbol=ES`.
- Products: `GET /instruments/future-option-products`,
  `.../{root_symbol}`, `.../{exchange}/{root_symbol}`.

## Cryptocurrency & warrants

- `GET /instruments/cryptocurrencies` (optional `symbol` filter) · `.../{symbol}`. Fields:
  `symbol`, `description`, `active`, `streamer-symbol`, `tick-size`.
- `GET /instruments/warrants` (optional `symbol`) · `.../{symbol}`. Fields: `symbol`,
  `description`, `cusip`, `listed-market`, `active`.

## Quantity decimal precision

`GET /instruments/quantity-decimal-precisions` — the min order-quantity increment per instrument
type (critical for crypto and fractional equities). Each row: `instrument-type`, `symbol`
(if symbol-specific), `value` (decimal places), `minimum-increment-precision`.

## Key data-model fields

Common resolution patterns:

- **Build an option order:** verify equity is `active` and check
  `is-fractional-quantity-eligible`, then `GET /option-chains/{symbol}/nested` to pick
  expiration/strike and read the `call`/`put` symbols.
- **Resolve an OCC symbol from a position:** `GET /instruments/equity-options/{symbol}` for full
  contract details.
- **Discover tradeable futures:** `GET /instruments/future-products` → pick a product →
  `GET /instruments/futures?product-code[]=ES&only-active-futures=true`.
- **Short-sale eligibility:** check `lendability` on the equity.
