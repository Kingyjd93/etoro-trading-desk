---
name: desk-execution-protocol
description: "The Execution Trader's protocol for sending one approved eToro ticket - pre-send checklist, fresh prepare-trade or prepare-close, field-by-field match against the ticket, one place call, the meaning of each outcome, same-token retry only on unknown, and reconciliation from eToro's record. Use before and after every eToro write call."
---
# Execution protocol

For the Execution Trader only. Other roles read this to know what "sent" means; they never perform it.

## 1. Pre-send checklist

All must be true. Any failure: do not send, name the failed item, hand back to the Desk Lead.

1. **Ticket**: has an `ET-` id, a Risk PASS with exact fields, and the user's approval of that exact ticket (their recorded reply word, default YES, as a direct reply to that ticket's approval post, or `approve <id>`) typed after the ticket was shown, within its expiry (default 30 minutes).
2. **Native gate**: `desk.md` records the Grok Bot Auto-review rule covering `place-trade`, `place-close` and `execute-write` on `user-Etoro-trading`, and `status` is not research-only. The rule's approval card must appear for this call. A write tool that runs with no card is an incident: stop.
3. **Account**: ticket and call both name the account recorded in `desk.md` (`real`, or `demo` on a demo desk); `get-my-profile-and-scopes` `realCid` (or `demoCid`) equals the account id in `desk.md`.
4. **Eligibility live**: `allowOpenPosition` (or `allowClosePosition`; `allowPartialClosePosition` for partials) true; settlement type, direction and leverage present in `leverageConfigs`; stop-loss % of margin within bounds; units within `maxUnitsPerOrder` and `unitsQuantityType`.
5. **Quote fresh**: `asOf` within the ticket's age bound; live ask (buy) or bid (sellShort) within the ticket's slippage bound of the ticket price. Market closed: send only if the ticket says pending is allowed.
6. **Capacity rechecked** by the Risk Manager immediately before new risk; no earlier send on this account still `unknown` or unexpectedly `pending`.

## 2. Prepare, compare, place

```
prepare-trade  account=real direction=<buy|sellShort> instrumentId=<id> orderType=<mkt|mit|limitIOC>
               [triggerRate | limitRate] units=<n> (or amount=<usd>) leverage=<n>
               stopLossRate=<r> [takeProfitRate=<r>] stopLossType=<fixed|trailing>
```

- `verdict: rejected` -> stop; quote `reasons[]` to the Desk Lead.
- `verdict: ready` -> compare the confirmation with the ticket field by field: instrument id and symbol, direction, order type, units or amount, leverage, stop-loss, take-profit, stop type, settlement; estimated units and costs within tolerance (default 1%); balance impact; warnings. A new warning that changes the economics (balance unreadable, costs higher, SL out of bounds) is a mismatch.
- Mismatch -> abort, return the ticket. Never "fix" a field yourself.
- Match -> write the preview and `expiresAt` into the proposal file, then call `place-trade token: <confirmationToken>` **once**.

Closes are the same with `prepare-close account=real positionId=<id> [unitsToDeduct=<n>]` and `place-close`. Copied (mirror) positions close only in full. A US stock outside session hours is rejected at prepare: report it; do not improvise.

## 3. Outcomes

| outcome | meaning | do |
| --- | --- | --- |
| `executed` | filled | reconcile, report |
| `partiallyFilled` | some units filled | reconcile; remainder is an incident, not a re-send |
| `rejected` | refused | quote `errorCode`/`errorMessage`; 623 = user-level trading block, tell the user to contact eToro support; no retry without a fresh ticket |
| `cancelled` / `expired` | did not execute | reconcile; journal; a new attempt needs a new ticket |
| `pending` | **accepted and waiting** (market closed, MIT trigger) | never place again; record orderId; reconcile later; cancelling is its own ticket |
| `unknown` | timeout; state not known | call `place-*` again with the **same token** while unexpired (idempotency key sealed in it). Never prepare a new token for the same ticket. Expired: reconcile only |

## 4. Writes through execute-write

Used for stop-loss/take-profit edits and pending-order cancels only.

- `PATCH /api/v2/trading/positions/{positionId}` body `{"stopLossRate": r, "stopLossType": "fixed"}` (and/or `takeProfitRate`). 202 = accepted, not done. Never `clearStopLoss: true`; never widen past the allowed maximum (moves cash into margin).
- `DELETE /api/v3/trading/execution/orders/{orderId}`. 202 = request accepted; it may already have executed.
- One call per approval. Timeout (statusCode 0) or 5xx: retry only with the **same** `xRequestId` returned by the failed call. Never reuse an xRequestId for a different operation.
- Before a write, show the exact method, path and body in the floor message; the native card shows the same.

## 5. Reconcile from eToro's record

- `GET /api/v2/trading/info/orders:lookup?orderId=<id>` (or `referenceId=`): `status.id` 1 Received, 2 Placed, 3 Filled, 4 Rejected, 5 PartiallyFilled, 6 PendingCancel, 7 Canceled, 8 Expired, 9 CanceledPartiallyFilled, 10 RejectedPartiallyFilled, 11 WaitingForMarket, 12 PendingTriggeredRate; `positionExecutions[]` with `positionId`, `openingData.avgPrice`, `units`, `marketSpread`, `fees`, `stopLossRate`.
- `get-my-positions-and-orders` (positions and pending orders), `get-my-portfolio-summary` (cash).
- Stop-loss state: `GET /api/v1/trading/info/real/pnl` -> `isNoStopLoss`, `stopLossRate`, `isTslEnabled` per position.
- Report eToro's numbers: fill rate vs ticket price in bps, units, fees, positionIds, SL confirmed, UTC times.

## 6. Report

```
EXECUTION | ET-... | eToro real <realCid> | <UTC>
prepared: <time> verdict ready; <fields>; expiresAt <time>
placed: <time> -> outcome <x>, orderId <n>, referenceId <uuid>
reconciled <time>: status <id name>; positionId <n>, <units> @ <rate> (<bps> vs ticket); SL <rate> confirmed (isNoStopLoss false)
costs: <fees, spread, overnight>
next: <owner>
```

Post on the floor, update the proposal file, DM the Trade Reviewer.

## Never

- More than one placement per approval; batching; "while I'm here".
- A send from a routine or loop.
- Money, transfer, deposit, withdrawal, sub-account, OAuth-app, agent-portfolio or copy-trading routes.
- A raw order route (`POST /api/v*/trading/execution/...`) instead of prepare/place.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

