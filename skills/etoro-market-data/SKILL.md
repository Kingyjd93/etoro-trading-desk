---
name: etoro-market-data
description: "How the desk reads eToro market data through the connector - batch quotes and eligibility with get-instruments-overview, instrument search and ids, rates, candle history (two routes, and which one lags), closing prices, full eligibility and what-if costs - with freshness checks, market-hours reading and rate limits. Use for any price, spread, history, eligibility or cost question."
---
# eToro market data

All reads; no approval needed. Every figure you report carries the tool or route, the UTC time of the call and the quote `asOf`.

## 1. First call: get-instruments-overview

`get-instruments-overview` with every instrument of interest in **one** call (1-100; symbols and/or instrumentIds, or a free-text `query`). Returns per instrument:

- `market`: instrumentId, symbol, name
- `quote`: `ask` (you buy here), `bid` (you sell here), `spread`, `asOf`
- `performance`: `previousClose`, daily/weekly/monthly change %
- `eligibility` (summary for the connection's account): `allowOpenPosition`, `minPositionExposure` (USD), `maxUnitsPerOrder`, `allowedLeveragesLong`/`Short`, `allowMitOrders`, `allowEntryOrders`, `allowTrailingStopLoss`, `requiresW8Ben`, `unitsQuantityType`
- `warnings`: sections that degraded (their fields are null); `notFoundSymbols`/`notFoundInstrumentIds`

Read it carefully:
- **Empty `allowedLeveragesLong`/`Short` means the account cannot open it** through the API right now, even if `allowOpenPosition` is true. This is common for index, commodity and FX CFDs on some accounts.
- **Quote age**: `asOf` minutes or hours old means the market is closed or in a break (index/commodity CFDs showed `asOf` 20:59:59 UTC during the 21:00 UTC daily break), or the feed is stale. Say which.
- **Change %**: computed by eToro from official closes. During breaks it may not equal live/previousClose - 1 (observed for index and commodity CFDs). Quote eToro's figure as eToro's, or compute your own and label it.
- Spread in bps of mid: `(ask - bid) / ((ask + bid) / 2) x 10,000`.

## 2. Instrument ids

Symbols collide and similar instruments have different ids (NVDA 1137 vs NVDA.RTH 8760). Always carry the id.
- Search: `get-instruments-overview` `query`, or `GET /api/v2/market-data/instruments/search` (free text), `GET /api/v2/market-data/instruments` (by id or symbol), `GET /api/v1/market-data/search`.
- Known ids (verified when this manual was written; re-check): BTC 100000, ETH 100001, SPX500 27, NSDQ100 28, GOLD 18, AAPL 1001, NVDA 1137.

## 3. Rates

`GET /api/v2/market-data/rates` (bid/ask for one or more instruments) or `GET /api/v1/market-data/instruments/rates`. Prefer `get-instruments-overview` unless you need only the rate.

## 4. Candles: two routes

| Route | Use | Notes |
| --- | --- | --- |
| `GET /api/v1/market-data/instruments/{id}/history/candles/{asc\|desc}/{interval}/{count}` | recent bars, live edge | intervals OneMinute, FiveMinutes, TenMinutes, FifteenMinutes, ThirtyMinutes, OneHour, FourHours, OneDay, OneWeek; count <= 1000; one side (no bid/ask choice) |
| `GET /api/v1/data/instruments/{id}/candles?interval=1d&side=bid\|ask\|both&from=&to=&limit=&cursor=` | deep history | intervals 1m..1w; up to 2000 per page, cursor paging; values are decimal strings; **can lag by days** - call `.../candles/coverage` first and check the last `time` |

For backtests, take deep history from the data route and stitch recent bars from the market-data route; check the overlap.

## 5. Closing prices

`GET /api/v1/market-data/instruments/history/closing-price` - official closes used for performance figures.

## 6. Full eligibility (before any ticket)

`execute-read` POST `/api/v2/trading/info/eligibility` body `{"instrumentIds":[1001],"currency":"USD"}` (read-semantics POST). Per instrument: `allowOpenPosition`, `allowClosePosition`, `allowPartialClosePosition`, `allowMitOrders`, `allowEntryOrders`, `allowExitOrders`, `allowTrailingStopLoss`, `unitsQuantityType`, `orderFillBehaviorType`, `allowedOrderQuantityType`, `tradeUnitType`, `additionalBufferPercent`, and `leverageConfigs[]` each with `settlementType` (real, cfd, realFutures, marginTrade), `direction`, `leverageValues`, `isPotential`, `minPositionAmount`, `allowEditStopLoss`, `min/max/defaultStopLossPercentage`, take-profit equivalents. Stop-loss and take-profit percentages are **of margin**. Changes rarely: once per session per instrument is enough. 20 requests / 60 s.

## 7. What-if costs

`execute-read` POST `/api/v2/trading/info/costs` body e.g. `{"action":"open","transaction":"buy","instrumentId":1001,"settlementType":"real","orderType":"mkt","leverage":1,"units":4.2201}`. Returns `markup`, `marketSpread`, `transactionFee`, `overnightFee`, `overWeekendFee`, `sdrt` with currency and `lastUpdated`. Close costs are not supported by this route yet. 20 requests / 60 s.

## 8. Market hours (practical)

- Crypto: every day. Fresh `asOf`.
- US stocks: 09:30-16:00 New York regular session; eToro may quote extended hours for some stocks (AAPL and NVDA quotes were seconds old at 21:22 UTC, after the regular close). `allowEntryOrders` true means an order may be accepted while closed and sit `pending` (WaitingForMarket).
- Index/commodity/FX CFDs: about 24/5 with a daily break; closed weekends.

## Rate limits (verified when this manual was written; re-check with `get-route-spec`)

Market-data group (all `/api/v1/market-data/*` and `/api/v2/market-data/*`): 120 / 60 s shared. Data candles + coverage: 120 / 60 s shared. Eligibility 20 / 60 s; costs 20 / 60 s. Shared by every Bot on the connection.

## Never

- Answer from memory, or from a quote you did not fetch this turn.
- Loop `get-instruments-overview` per instrument.
- Call indicators signals; forecast.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

