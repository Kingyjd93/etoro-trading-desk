---
name: etoro-api-reference
description: "Compact reference of the eToro Public API routes and connector tools the desk uses, as verified when the manual was written (API v1.387.0) - route ids, methods, paths, read/write class, rate-limit groups - plus the forbidden route families. Use to find the right route before execute-read/execute-write and to check whether a route is a write."
---
# eToro API reference (verified when this manual was written: API v1.387.0, connector skillVersion 1.21.1)

Routes and limits change. Treat this as a map, not the authority.

`spec` = full spec read with `get-route-spec` (scopes and rate limit confirmed). `catalog` = seen in `get-all-routes` only; read the spec before first use. Re-check with `get-all-routes` (its `mutatingRouteIds` is the authority on what is a write).

## Connector tools

Reads: `get-instruments-overview`, `get-my-portfolio-summary`, `get-my-positions-and-orders`, `get-my-balances`, `get-my-trading-history`, `get-my-watchlists`, `get-my-profile-and-scopes`, `get-trader-profile-summary`, `get-tags`, `get-all-routes`, `get-route-spec`, `execute-read`.
Writes (Execution Trader only, native card): `place-trade`, `place-close`, `execute-write`. Token minting (Execution Trader only): `prepare-trade`, `prepare-close`.

## Market data and data (read)

| Route id | Method and path | Limit | Status |
| --- | --- | --- | --- |
| getMarketDataInstrumentsByInstrumentIdHistoryCandlesByDirectionByIntervalByCandlesCount | GET /api/v1/market-data/instruments/{instrumentId}/history/candles/{direction}/{interval}/{candlesCount} | market-data group 120/60 s | spec |
| getMarketDataInstrumentsHistoryClosingPrice | GET /api/v1/market-data/instruments/history/closing-price | market-data group | catalog (group per spec above) |
| getMarketDataInstrumentsRates | GET /api/v1/market-data/instruments/rates | market-data group | catalog |
| getRates | GET /api/v2/market-data/rates | market-data group | catalog |
| getMarketDataInstruments | GET /api/v1/market-data/instruments | market-data group | catalog |
| getSecurities | GET /api/v2/market-data/instruments | market-data group | catalog |
| getInstrumentsSearch | GET /api/v2/market-data/instruments/search | market-data group | catalog |
| getMarketDataSearch | GET /api/v1/market-data/search | market-data group | catalog |
| getMarketDataInstrumentTypes, getMarketDataExchanges, getMarketDataStocksIndustries | GET /api/v1/market-data/instrument-types, /exchanges, /stocks-industries | market-data group | catalog |
| getPriceCandles | GET /api/v1/data/instruments/{instrumentId}/candles | data-candles group 120/60 s | spec |
| getPriceCandleCoverage | GET /api/v1/data/instruments/{instrumentId}/candles/coverage | data-candles group | catalog (group per spec above) |

## Trading - Real (read)

| Route id | Method and path | Limit | Status |
| --- | --- | --- | --- |
| getInstrumentEligibility | POST /api/v2/trading/info/eligibility (read-semantics; execute-read with body) | 20/60 s dedicated | spec |
| getCost | POST /api/v2/trading/info/costs (read-semantics; open only) | 20/60 s dedicated | spec |
| getTradingInfoOrdersLookup | GET /api/v2/trading/info/orders:lookup?orderId= or ?referenceId= | lookup group 60/60 s | spec |
| getTradingInfoRealOrdersByOrderId | GET /api/v1/trading/info/real/orders/{orderId} | lookup group | catalog (group per spec above) |
| getTradingInfoRealCloseOrdersByOrderId | GET /api/v1/trading/info/real/close-orders/{orderId} | lookup group | catalog (group per spec above) |
| getTradingInfoRealPnl | GET /api/v1/trading/info/real/pnl | pnl group 60/60 s | spec |
| getTradingInfoPortfolio | GET /api/v1/trading/info/portfolio | pnl group | catalog (group per spec above) |
| getTradingInfoPortfolioInstrumentBreakdown | GET /api/v2/trading/info/instrument-breakdown | pnl group | catalog (group per spec above) |
| getAggregatedPortfolio / postAggregatedPortfolio | GET or POST (read-semantics) /api/v1/trading/info/aggregate-portfolio | see spec | catalog |
| getTradingInfoTradeHistory | GET /api/v1/trading/info/trade/history | see spec | catalog |
| getMe | GET /api/v1/me | see spec | catalog |

## Trading - Real (write; Execution Trader only)

| Route id | Method and path | Limit | Status |
| --- | --- | --- | --- |
| updateRealPosition | PATCH /api/v2/trading/positions/{positionId} (SL/TP edit; 202 async) | default pool 60/60 s | spec |
| deleteTradingExecutionOrdersByOrderIdV3 | DELETE /api/v3/trading/execution/orders/{orderId} (cancel pending; 202) | order group 20/60 s | spec |
| (prefer prepare/place) createTradingExecutionOrdersV3, createTradingExecutionOrders, createTradingExecutionMarketOpenOrdersByUnits/ByAmount, createTradingExecutionLimitOrders, createTradingExecutionMarketCloseOrdersPositionsByPositionId | POST /api/v3/trading/execution/orders, /api/v2/..., /api/v1/trading/execution/... | order group 20/60 s | catalog; **not used by the desk** |
| other cancels: deleteTradingExecutionOrdersByOrderId (v2), deleteTradingExecutionLimitOrdersByOrderId, deleteTradingExecutionMarketOpenOrdersByOrderId, deleteTradingExecutionMarketCloseOrdersByOrderId | DELETE ... | order group | catalog; use v3 |

## Price alerts (write only on the user's explicit request)

| Route id | Method and path | Status |
| --- | --- | --- |
| getPriceAlerts | GET /api/v1/price-alerts | catalog |
| postPriceAlerts / patchPriceAlertsByAlertId / deletePriceAlertsByAlertId | POST /api/v1/price-alerts, PATCH and DELETE /api/v1/price-alerts/{alertId} | catalog (mutating) |

## Order status ids (orders:lookup)

1 Received, 2 Placed, 3 Filled, 4 Rejected, 5 PartiallyFilled, 6 PendingCancel, 7 Canceled, 8 Expired, 9 CanceledPartiallyFilled, 10 RejectedPartiallyFilled, 11 WaitingForMarket, 12 PendingTriggeredRate.

## Forbidden families (never call, any Bot)

- Transfer (7 routes, `/api/v1/money/*`, including `postMoneyTransfer` POST /api/v1/money/transfers), deposits and withdrawals, Cash Accounts.
- Sub-Accounts (both groups), SSO - Applications, SSO - Scopes writes, Agent Portfolios.
- Copy Trading (real and demo) writes, Social Feeds writes, Watchlists writes, Notifications writes.
- KYC and registration routes (partner-only; not normally visible on a user connection).

## Rate-limit groups

| Group | Budget |
| --- | --- |
| market-data (all /api/v1/market-data/*, /api/v2/market-data/*) | 120 / 60 s shared |
| data candles + coverage | 120 / 60 s shared |
| eligibility | 20 / 60 s dedicated |
| costs | 20 / 60 s dedicated |
| order lookups (orders:lookup, real/orders, real/close-orders) | 60 / 60 s shared |
| pnl, portfolio, instrument-breakdown | 60 / 60 s shared |
| order create + cancel (11 routes) | 20 / 60 s shared |
| default pool (everything without a dedicated limit, incl. position edits) | 60 / 60 s shared |

Headers `RateLimit-*` may be absent on success; 429 carries `Retry-After` (`retryAfterSeconds` in the tool result). One connection serves every Bot.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

