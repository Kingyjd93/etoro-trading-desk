---
name: desk-post-trade-review
description: "How the Trade Reviewer keeps the desk journal and reviews eToro trades and incidents - reconstructing from eToro's own record (orders lookup, positions, closed-trade history, stop-loss state), measuring execution cost, and grading process separately from outcome. Use after every send, when a trade closes, for weekly reviews and for incident reviews."
---
# Journal and post-trade review

## 1. Journal

`/workspace/trading-desk/journal/YYYY-MM-DD.md` (UTC date), one file per day with activity. Append-only; corrections are new timestamped lines.

```
## 2026-01-15
- 14:02 ET-20260115-01 opened (user idea: buy AAPL, invalidation 330)
- 14:12 ET-20260115-01 Risk PASS (units 4.2201, risk $50.00 = 0.50%)
- 14:20 ET-20260115-01 approved by user (YES on the approval post)
- 14:30 ET-20260115-01 executed, orderId ..., positionId ..., 4.2201 @ 340.57; SL 330.00 confirmed
- start-of-day equity: $X (get-my-portfolio-summary 00:00 UTC)
- learned: <one line>
```

## 2. Trade review

Run when a position closes (or a ticket voids after partial activity).

Inputs, each with UTC time:
- proposal file `proposals/<id>.md`
- `GET /api/v2/trading/info/orders:lookup?orderId=<open order>`: `positionExecutions[].openingData` (`avgPrice`, `units`, `marketSpread`, `markup`, `fees`, `taxes`), `totalCosts`
- close order: `GET /api/v1/trading/info/real/close-orders/{orderId}` when the close was a desk order
- `get-my-trading-history` with `minDate` at or before the open: open/close rate and time, units, `fees`, `netProfit` (**excludes fees**; subtract them yourself), `stopLossRate`/`takeProfitRate` (0.0001 means none)
- stop-loss coverage history from the desk's book checks (`/real/pnl` `isNoStopLoss`)

Measure:
- entry slippage: (fill - ticket ask) / ticket ask in bps for buys (bid for shorts); same for exit
- spread paid (from `marketSpread` x units, both legs), fees, overnight/weekend fees, taxes
- result net of all costs, in USD and in R (R = the ticket's risk budget)
- protection: was a stop-loss on the position the whole time?
- lifecycle: every stage present, in order, with the right owner; approval by exact id; one send

Grade **process** (clean / defects listed) and **outcome** (net R) separately. One finding per review.

## 3. Weekly desk review

Trades opened/closed, hit rate, average win and loss in R, costs as % of gross P&L, max drawdown of equity from journal snapshots, incidents, process breaks, unprotected-position minutes. Facts only; strategy questions go to the Strategist.

## 4. Incident review

Timeline (UTC) from journal and eToro records; what the desk did; what the controls did (did the native card appear? did the one-writer rule hold?); impact in USD; one corrective action with an owner and date. Save to `journal/incidents/YYYY-MM-DD-<slug>.md`.

## Pitfalls

- Reviewing from chat instead of eToro's record.
- Using `netProfit` as the net result (fees are separate).
- Reading `stopLossRate: 0.0001` as a stop.
- Grading a profitable trade as good process.
- Comparing fills to mid rather than the ask/bid in the ticket.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

