---
name: etoro-orders
description: "The eToro write paths, for the Execution Trader only - open with prepare-trade/place-trade, close with prepare-close/place-close, edit a position's stop-loss/take-profit and cancel a pending order with execute-write - with exact arguments, eToro's order semantics (ask/bid, units vs amount, settlement, leverage, SL/TP bounds, pending orders, fees), idempotent retries and confirmation. Other roles read it to understand tickets; they never call these tools."
---
# eToro orders (Execution Trader only)

Every call here needs: a Risk PASS, the user's approval of that exact ticket (YES as a reply to its approval post, or `approve ET-...`) (or the written standing protective-stop approval), and the Grok Bot native approval card. One call per approval.

## eToro order semantics

- **Price**: buys fill around the **ask**, sells/shorts around the **bid**. One quote, no order book.
- **Size**: exactly one of `units`, `amount` (USD) or `contracts` (futures). The desk sizes in **units** from the stop; `amount` is shown for context. `orderFillBehaviorType: bestEffort` may fill fewer units if the rate moves against an amount order.
- **Settlement**: `real` (the asset, full value), `cfd` (contract for difference: overnight and weekend fees, may allow leverage and shorting), `realFutures`, `marginTrade`. Allowed combinations come from eligibility `leverageConfigs`. Many accounts get stocks and crypto at 1x only, and some index/commodity/FX CFDs are not openable via the API (empty leverage lists): check eligibility, never assume.
- **Leverage**: omit or 1 for none. Above 1 requires `stopLossRate`. `sellShort` requires `stopLossRate`.
- **Stop-loss / take-profit**: attributes of the position, set at open or edited later. No separate stop order exists. Bounds are % **of margin** (`min/maxStopLossPercentage`). `stopLossType: trailing` needs `allowTrailingStopLoss` and a `stopLossRate`.
- **Order types**: `mkt`; `mit` (market-if-touched at `triggerRate`; can sit pending); `limitIOC` (`limitRate` within 10% of market; fills now or cancels).
- **Closed market**: if `allowEntryOrders` is true, an open order may be accepted and wait as `pending` (WaitingForMarket). Closes outside hours are rejected at prepare unless exits are allowed.
- **Fees**: transaction fee per order (USD 2 seen on stock positions), spread, markup, overnight/weekend for CFDs, SDRT on eligible UK buys. Preview with `POST /api/v2/trading/info/costs` and in the prepare-trade confirmation.

## Open

```
prepare-trade
  account: real
  direction: buy | sellShort
  instrumentId: 1001                # never a bare symbol when ids can collide
  orderType: mkt | mit | limitIOC
  triggerRate / limitRate           # only for mit / limitIOC
  units: 4.2201                     # or amount / contracts, exactly one
  leverage: 1
  stopLossRate: 330.00
  takeProfitRate: (optional)
  stopLossType: fixed | trailing
```

`verdict: ready` -> compare with the ticket (see `desk-execution-protocol`) -> `place-trade token: <confirmationToken>` once, before `expiresAt`. Outcomes: executed, partiallyFilled, rejected (errorCode/errorMessage; 623 = account trading block), cancelled, expired, pending (never re-place), unknown (same-token retry only).

## Close

```
prepare-close account: real  positionId: <id>  [unitsToDeduct: <n>]   # omit for full close
place-close  token: <confirmationToken>
```

Verdict rejected when the position is not on this account, the instrument does not allow closing now (e.g. US stock outside hours), or a partial on a mirror position. Outcomes as above; `pending` = accepted, not yet final: do not close again.

## Edit stop-loss / take-profit

```
execute-write
  method: PATCH
  path: /api/v2/trading/positions/<positionId>
  body: {"stopLossRate": 330.00, "stopLossType": "fixed"}
```

- 202 returns `operationId`, `positionId`, `referenceId`: accepted, not confirmed. Confirm in `GET /api/v1/trading/info/real/pnl` (`isNoStopLoss` false, `stopLossRate` as set).
- 404 not found, 409 position closed.
- Never `clearStopLoss: true`. Never move a stop further from price than the ticket says. Deepening beyond the allowed maximum moves cash from the balance into margin: forbidden.
- Under the standing protective approval: only on a position with no stop, at a level from its ticket or the written policy in `risk-limits.md`, within `min/maxStopLossPercentage`.

## Cancel a pending order

```
execute-write  method: DELETE  path: /api/v3/trading/execution/orders/<orderId>
```

202 = cancellation requested; the order may already have executed. Confirm with `orders:lookup`: 7 Canceled or 9 CanceledPartiallyFilled = done; 6 PendingCancel = still processing. Idempotent if already closed or cancelled.

## Retries

- `place-trade`/`place-close`: on `unknown`, the same token may be re-sent while unexpired; it carries the idempotency key. Never mint a new token for the same ticket while the first result is unknown.
- `execute-write`: on statusCode 0 or 5xx, retry once with the **same** `xRequestId` from the failed call. Never reuse an xRequestId for a different operation.

## Rate limits (verified when this manual was written; re-check with `get-route-spec`)

Order create and cancel routes share 20 / 60 s. Position edits use the default pool 60 / 60 s. Order lookups 60 / 60 s shared.

## Never

- Raw order routes (`POST /api/v1/trading/execution/*`, `POST /api/v2|v3/trading/execution/orders`) - use prepare/place.
- Copy-trading, money, transfer, deposit, withdrawal, sub-account or agent-portfolio routes.
- A write from a routine or a loop; a write without the native card.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

