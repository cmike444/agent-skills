# Market Data (REST), Metrics, Search, Sessions, Alerts

Non-streaming market data and related lookups. For real-time streaming, see
[streaming.md](streaming.md).

## Table of contents
- [REST market data (by-type)](#rest-market-data-by-type)
- [Market metrics (IV, liquidity)](#market-metrics-iv-liquidity)
- [Historical dividends & earnings](#historical-dividends--earnings)
- [Symbol search](#symbol-search)
- [Market sessions & holidays](#market-sessions--holidays)
- [Quote alerts](#quote-alerts)

## REST market data (by-type)

`GET /market-data/by-type` — point-in-time snapshot for many symbols in one call. The
snapshot alternative to DXLink. **Not available (real-time) in sandbox.**

**Two big gotchas:** parameter names are **singular hyphenated**, and the response is
**camelCase** (not the usual kebab-case).

Params (each an array, repeat to add symbols) — combined **max 100 symbols**:
`equity`, `equity-option`, `future`, `future-option`, `cryptocurrency`, `index`.
Example: `?equity[]=AAPL&equity[]=SPY&equity-option[]=AAPL  260619C00200000`.

`MarketData` fields (camelCase): `symbol`, `instrumentType`, `bid`/`bidSize`, `ask`/`askSize`,
`mid`, `mark`, `last`, `lastExt` (extended hours), `lastTradeTime` (epoch ms), `volume`, `open`,
`dayHighPrice`, `dayLowPrice`, `close`, `closePriceType`, `prevClose`, `yearHighPrice`,
`yearLowPrice`, `beta`, `dividendAmount`, `dividendFrequency`, `tradingHalted`(+`Reason`,
`haltStartTime`/`haltEndTime` epoch ms), `lowLimitPrice`/`highLimitPrice` (futures circuit
breakers), `updatedAt`, nested `instrument`. Deprecated: `dayOpen`→`open`, `dayHigh`→
`dayHighPrice`, `dayLow`→`dayLowPrice`, `dayClose`→`close`, `prevDayClose`→`prevClose`.

Use `mark` for valuation, `bid`/`ask`/`mid` for limit pricing, `tradingHalted` to skip halted
names.

## Market metrics (IV, liquidity)

`GET /market-metrics?symbols=AAPL,SPY,TSLA` — volatility + liquidity for underlyings. Core inputs
for premium-selling decisions. **Not available in sandbox.** URL-encode special chars
(`BRK/B` → `BRK%2FB`).

`MarketMetricInfo` fields (decimals, so `0.2845` = 28.45%):
- `implied-volatility-index`, `implied-volatility-index-5-day-change`
- `implied-volatility-rank` (IVR — where current IV sits in the 52-wk high/low range; higher =
  options relatively expensive)
- `implied-volatility-percentile` (% of past-year days IV was below current)
- `liquidity`, `liquidity-rank`, `liquidity-rating` (integer, e.g. 1–5)
- `option-expiration-implied-volatilities[]` — per expiration: `expiration-date`,
  `settlement-type` (`AM`/`PM`), `option-chain-type` (`Standard`/`Non-standard`),
  `implied-volatility`

IVR vs IV-percentile differ: IVR uses the range; percentile uses day-count. They can diverge.

## Historical dividends & earnings

- `GET /market-metrics/historic-corporate-events/dividends/{symbol}` → `DividendInfo[]`:
  `occurred-date` (ex-div), `amount`. (Watch ex-div dates for early-assignment risk on short
  ITM calls.)
- `GET /market-metrics/historic-corporate-events/earnings-reports/{symbol}` → `EarningsInfo[]`:
  `occurred-date`, `eps`. Requires `start-date`; optional `end-date`.

## Symbol search

`GET /symbols/search/{symbol}` — prefix/typeahead match (single endpoint, no pagination).
`SymbolData` fields: `symbol`, `description`, `listed-market`, `instrument-type`, `options`
(bool — options available), `price-increments`, `trading-hours`. `AAP` returns `AAP`, `AAPL`, …

## Market sessions & holidays

Determine whether markets are open, when they open next, and holidays. All timestamps **UTC**.

Collections: `Equity`, `CME`, `CFE`, `Zero Hash CLOB` (crypto — range endpoint only).

- `GET /market-time/sessions?to-date=&from-date=&instrument-collection=` — sessions over a range
  (**≤ 9 months**), default `Equity`.
- `GET /market-time/sessions/current?instrument-collections[]=Equity&instrument-collections[]=CME`
  — current session(s) for one/more collections, with next/previous and state.
- Equities: `GET /market-time/equities/sessions/current` (has `state`),
  `.../sessions/next?date=`, `.../sessions/previous?date=`, `.../holidays`.
- Futures: `GET /market-time/futures/sessions/current` (all exchanges),
  `.../sessions/current/{collection}` (`CME`/`CFE`), `.../sessions/next/{collection}`,
  `.../sessions/previous/{collection}`, `.../holidays/{collection}`.

Session fields: `start-at` (pre-market/overnight open), `open-at` (regular open), `close-at`
(regular close), `close-at-ext` (extended close), `instrument-collection`. `CurrentSession` adds
`state` (`Open`/`Closed`/`Pre-Market`/`After-Hours`), `next-session`, `previous-session`.
`MarketCalendar`: `market-holidays`, `market-half-days`. US equity regular hours = 9:30am–4:00pm
ET.

## Quote alerts

Price alerts, **user-scoped (no account number)**, one-shot triggers.

- `GET /quote-alerts` → `QuoteAlert[]`.
- `POST /quote-alerts` → 201. Body: `symbol` (req), `field` (req: `Last`/`Bid`/`Ask`/`IV`),
  `operator` (req: `>`/`<`), `threshold` (req, **string** e.g. `"200.00"`), optional
  `instrument-type`, `dx-symbol`, `expires-at`.
- `DELETE /quote-alerts/{alert_external_id}` → 204.

`QuoteAlert` lifecycle timestamps: `created-at` → `triggered-at` → `completed-at`, or
`dismissed-at` / `expired-at`. For `field: IV`, threshold is a decimal (`"0.35"`). Alerts fire
once; recreate to keep monitoring. Delivered over the account streamer via
`quote-alerts-subscribe` (see [streaming.md](streaming.md)).
