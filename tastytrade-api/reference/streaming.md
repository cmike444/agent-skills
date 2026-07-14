# Streaming (WebSockets)

Two independent WebSocket streamers:
1. **DXLink market data** — real-time quotes/greeks/candles from dxFeed.
2. **Account streamer** — real-time order/balance/position updates from tastytrade.

## Table of contents
- [DXLink market data](#dxlink-market-data)
- [DXLink protocol sequence](#dxlink-protocol-sequence)
- [DXLink event types](#dxlink-event-types)
- [Candle events (historical OHLC)](#candle-events-historical-ohlc)
- [Account streamer](#account-streamer)
- [Account streamer actions](#account-streamer-actions)
- [Notifications & fill nuances](#notifications--fill-nuances)

## DXLink market data

Two steps: get a token, then stream from dxFeed.

1. `GET /api-quote-tokens` → `{ token, dxlink-url (wss://…dxfeed.com/realtime), level, ... }`.
   Token is unique to the customer and **expires after 24 hours**. Requires an actual brokerage
   customer — a username-only account gets `quote_streamer.customer_not_found_error`.
2. Open a WebSocket to `dxlink-url` and run the protocol below.

Prefer the official **JavaScript/TypeScript SDK** (wraps the dxFeed JS SDK) or dxFeed's SDKs
(.NET, Swift, C/C++, Java). Use the **`COMPACT` data format** — `FULL` is heavy and being retired.

**Streamer symbols, not OCC.** Subscribe with the `streamer-symbol` from the instrument /
`/nested` chain responses (e.g. `.AAPL250117C230`, `/6AM23:XCME`, `BTC/USD:CXTALP`), not the
OCC/tastytrade symbol. See [instruments.md](instruments.md).

Low-liquidity tickers may emit no events for long stretches even on a healthy connection — that's
normal, not a bug.

## DXLink protocol sequence

Send messages in this order (channel numbers are yours to choose; channel 0 is the connection):

1. `SETUP` — `{"type":"SETUP","channel":0,"version":"0.1-DXF-JS/0.3.0","keepaliveTimeout":60,"acceptKeepaliveTimeout":60}`
2. On receiving `AUTH_STATE: UNAUTHORIZED`, send `AUTH` — `{"type":"AUTH","channel":0,"token":"<api quote token>"}` → expect `AUTH_STATE: AUTHORIZED`.
3. `CHANNEL_REQUEST` — open a feed channel: `{"type":"CHANNEL_REQUEST","channel":3,"service":"FEED","parameters":{"contract":"AUTO"}}`.
4. `FEED_SETUP` — declare fields per event type with `"acceptDataFormat":"COMPACT"`, e.g.
   `{"type":"FEED_SETUP","channel":3,"acceptAggregationPeriod":0.1,"acceptDataFormat":"COMPACT","acceptEventFields":{"Quote":["eventType","eventSymbol","bidPrice","askPrice","bidSize","askSize"]}}`.
5. `FEED_SUBSCRIPTION` — add symbols/events:
   `{"type":"FEED_SUBSCRIPTION","channel":3,"reset":true,"add":[{"type":"Quote","symbol":"SPY"},{"type":"Trade","symbol":"SPY"}]}`.
   Unsubscribe with `"remove":[...]`.
6. `KEEPALIVE` — `{"type":"KEEPALIVE","channel":0}` **every ~30s** (timeout is 60s or the
   connection closes).

Data arrives as `FEED_DATA` messages (COMPACT: arrays of values matching your declared fields).

## DXLink event types

Available via the tastytrade quote token: **Profile, Quote, Summary, Trade, Greeks,
TimeAndSale** (plus `Candle`, below). Highlights:

- `Quote` — `bidPrice`, `askPrice`, `bidSize`, `askSize`.
- `Trade` / `TradeETH` — `price`, `dayVolume`, `size` (ETH = extended-hours trade).
- `Summary` — `openInterest`, `dayOpenPrice`, `dayHighPrice`, `dayLowPrice`, `prevDayClosePrice`.
- `Profile` — `description`, `tradingStatus`, `haltStartTime`/`haltEndTime`, `highLimitPrice`/
  `lowLimitPrice`, `high52WeekPrice`/`low52WeekPrice`, `shortSaleRestriction`.
- `Greeks` — `volatility`, `delta`, `gamma`, `theta`, `rho`, `vega`.
- `TimeAndSale` — tick-by-tick prints.

Full per-event schemas are in dxFeed's protocol docs (search e.g. `QuoteEvent`, `GreeksEvent`).

## Candle events (historical OHLC)

Subscribe to `Candle` events for historical bars. The symbol embeds period+type as `SYMBOL{=Nx}`
where `N` is the period and `x` the unit (`m` min, `h` hour, `d` day). Provide a `fromTime`
(Unix epoch **ms**); dxFeed streams every candle from then to now (no end-time). The final candle
is the "live" one and updates continuously as the quote moves.

Recommended granularity by lookback (avoid millions of events):

| Lookback | Type | Symbol | ~Count |
|----------|------|--------|--------|
| 1 day | 1 min | `AAPL{=1m}` | ~1,440 |
| 1 week | 5 min | `AAPL{=5m}` | ~2,016 |
| 1 month | 30 min | `AAPL{=30m}` | ~1,440 |
| 3 months | 1 hour | `AAPL{=1h}` | ~2,160 |
| 6 months | 2 hours | `AAPL{=2h}` | ~2,160 |
| 1 year+ | 1 day | `AAPL{=1d}` | ~365 |

TS SDK example:
```js
const date = new Date(); date.setDate(date.getDate() - 1)
dxLinkFeed.addSubscriptions({ type: 'Candle', symbol: 'AAPL{=5m}', fromTime: date.getTime() })
```

## Account streamer

One-directional push of account state changes (orders, balances, positions) plus public
watchlists and quote-alert triggers. Streamer messages always carry the **full object**, never a
diff.

Hosts: Sandbox `wss://streamer.cert.tastyworks.com` · Production `wss://streamer.tastyworks.com`.

Setup order (do it in this sequence or subscribe may fail with "not implemented"):
1. Open the WebSocket.
2. Send a `connect` action to subscribe to account updates.
3. Start sending `heartbeat`s (every **2s–1m**).

Each message includes `auth-token` = your access token. **Prefix it with `Bearer `**
(`"auth-token": "Bearer <access token>"`).

Subscribe message schema: `{ "action": "<action>", "value": <string|array>, "auth-token":
"Bearer …", "request-id": <n> }`. The server echoes `request-id` and returns
`web-socket-session-id`.

Connect example:
```json
{ "action": "connect", "value": ["5WT00000","5WT00001"], "auth-token": "Bearer <token>", "request-id": 2 }
```
Heartbeat:
```json
{ "action": "heartbeat", "auth-token": "Bearer <token>", "request-id": 1 }
```

## Account streamer actions

| Action | Purpose | `value` |
|--------|---------|---------|
| `connect` | Subscribe to account updates (orders, balances, positions) | array of account numbers |
| `heartbeat` | Keep-alive (send every 2s–1m) | blank |
| `public-watchlists-subscribe` | Public watchlist updates (still needs auth token) | blank |
| `quote-alerts-subscribe` | Quote-alert triggers (**user-level**, not per account) | blank |

## Notifications & fill nuances

Each notification has `type` (e.g. `Order`), a full `data` object, and a `timestamp`:
```json
{ "type": "Order", "data": { "id": 1, "status": "Live", "legs": [ ... ] }, "timestamp": 1688595114405 }
```

Use the streamer instead of polling `GET /orders/live` — it's lower-latency and won't get you
throttled.

**Order-fill nuance:** tastytrade marks an order `Filled` as soon as there's no remaining
quantity. For multi-leg option orders (which execute simultaneously), the **first** streamer
message can show `Filled` with only one leg's fills populated; the remaining legs' fills publish
immediately after in follow-up messages. A single leg can also fill in many pieces (100 shares →
up to 100 fills). Accumulate fills across messages rather than trusting the first `Filled`.
