---
name: desk-trade-lifecycle
description: "The end-to-end procedure for one eToro trade on the desk - idea to reviewed, journaled result - with the ET- ticket format, who owns each stage, the prepare-trade token rule and what done looks like. Use whenever the user wants to open, add to, reduce, close or protect a position, or whenever any Bot is about to touch an eToro write tool."
---
# Trade lifecycle

Every position change goes through the same seven stages, in order. Skipping one is a defect. The Desk Lead keeps it moving; each stage's owner does the work.

```
idea -> evidence -> risk sign-off -> user approval -> execution -> reconciliation -> review
 DL       MA/RA        RM              user            ET            ET               TR
```

## 0. Open a proposal

**Owner: Desk Lead.** Assign `ET-YYYYMMDD-NN` (UTC date, NN increments per day) and create `/workspace/trading-desk/proposals/<id>.md`:

```markdown
# ET-20260115-01

- opened: 2026-01-15 14:02 UTC by user
- instrument: AAPL (id 1001, Stocks)  account: real <realCid>
- idea: buy AAPL with invalidation below 330 (user's idea)
- status: evidence

## evidence
## risk
## approval
## execution
## reconciliation
## review
```

Later stages append under their headings. The file is the record; chat is context.

## 1. Evidence

**Owner: Market Analyst; Research Analyst when the idea depends on anything beyond eToro data.** Deliver what sizing and execution need: instrument id and type, live ask/bid, spread, quote `asOf`, whether the market is open now, previous close and recent range from candles, eligibility (settlement types and leverage by direction, min exposure, max units, stop-loss/take-profit bounds, MIT and entry-order support), and any scheduled event (earnings, ex-dividend, macro release, weekend for stocks). Done when the Risk Manager has every input and nothing is older than a few minutes.

## 2. Risk sign-off

**Owner: Risk Manager.** Reads `risk-limits.md` and live account state, sizes from the stop (`desk-risk-limits`), checks every gate, writes PASS or REJECT under `## risk`. A PASS produces the **ticket**:

```
TICKET ET-20260115-01 | eToro account: real (<realCid>) | rev 1
instrument: AAPL (instrumentId 1001, Stocks)   settlement: real   direction: buy
order type: mkt                                pending allowed if market closed: no
size: units 4.2201 (amount est. $1,437.16 at ask)    leverage: 1x
stop-loss: 330.00 fixed (3.10% of margin; allowed 0-100%)   take-profit: none
live quote: ask 340.55 / bid 340.45, spread 0.10 (2.9 bps), asOf 2026-01-15 14:11:52 UTC (get-instruments-overview)
est. units at ask: 4.2201 | costs (POST /api/v2/trading/info/costs 14:12 UTC): transaction fee $2.00 open, $2.00 close est.; markup $0.00; overnight $0.00/night; weekend $0.00
risk: $50.00 = 0.50% of equity $10,000.00 (get-my-portfolio-summary totalValue 14:11 UTC); R = 10.55 USD/unit
sizing: stressed 10.90 USD/unit (stop 10.55 + slippage 0.35) + fixed costs $4.00 -> worst case $49.999
send bounds: live ask within 10 bps of 340.55; quote age <= 60 s; prepare-trade costs/units within 1% of ticket
risk sign-off: PASS 14:12 UTC, risk-limits.md v1
expires: 14:42 UTC
token rule: prepare-trade tokens expire within minutes. The Execution Trader re-runs prepare-trade after your approval, places only if the fresh preview matches this ticket within the send bounds, and aborts back to the Desk Lead if anything material changed.
approve with: YES (as a reply to this post) | NO | HOLD - or "approve ET-20260115-01"
```

All money figures are in the account currency, **USD**. If the user thinks in another currency, add a converted line using a live FX quote for that pair with its source and time, labelled as a conversion. A REJECT names one failed gate and what would have to change; the user may revise the idea and the proposal returns to stage 1 under the same id.

## 3. User approval

**Owner: the user.** The Desk Lead posts the ticket in full and asks for approval by id. Approval is the user's recorded reply word (default YES; NO rejects, HOLD pauses) as a direct reply to that ticket's approval post, or the literal `approve <id>`, typed by the user after the ticket was shown (rules in `desk-operating-model`). Record the line and its UTC time under `## approval`. No Bot writes it, quotes it forward as new, infers it, or records one it did not see. Unsure whether a line came from the user means absent. An expired ticket is void; it needs a fresh PASS.

## 4. Execution

**Owner: Execution Trader.** Confirms the native gate per `desk-operating-model`, runs the pre-send checklist in `desk-execution-protocol`, re-runs `prepare-trade` (or `prepare-close`), compares its confirmation with the ticket, calls `place-trade` (or `place-close`) **once**, and records preview, `expiresAt`, outcome, orderId, referenceId and UTC times under `## execution`. Anything other than a clean outcome goes to `desk-incident-response`.

## 5. Reconciliation

**Owner: Execution Trader.** Confirms from eToro's record what happened (`orders:lookup`, `get-my-positions-and-orders`, `/api/v1/trading/info/real/pnl` for stop-loss state) and writes it under `## reconciliation`. Posts the execution report on the floor and DMs the Trade Reviewer. A `pending` order is reconciled again when the market opens or the trigger is reached; it is never re-sent.

## 6. Review

**Owner: Trade Reviewer.** Journals the trade the day it happened; reviews when it closes, per `desk-post-trade-review`; sets `status: closed`.

## Adjustments and exits are trades too

Stop-loss or take-profit edits, partial or full closes, adds, and pending-order cancels each get a suffixed ticket (`ET-20260115-01-B`) that begins with the parent id and the original `- idea:` line, then states what changes and why. Each gets fresh evidence, a new Risk PASS, its own approval by exact suffixed id and its own expiry. Closing needs a ticket stating full or partial units read live. eToro write paths: open via prepare/place-trade; close via prepare/place-close; SL/TP edit via `execute-write PATCH /api/v2/trading/positions/{positionId}`; pending-order cancel via `execute-write DELETE /api/v3/trading/execution/orders/{orderId}`.

The only exception is a written standing approval in `desk.md`. The only kind allowed is **reduce-only protective stop-losses**, and only if the user granted it - setting or tightening the stop-loss rate on an existing position that has none, never opening or adding. The stop level must come from the position's approved ticket or a protective-stop policy the user wrote in `risk-limits.md`; without one, the desk alerts instead of inventing a level. Every such action is journaled.

## Rehearsal

The lifecycle is identical on the eToro demo account. A rehearsal on demo is optional and is the user's choice; it still needs a ticket and an approval by id.

## Definitions of done

- **live**: stage 5 shows an executed position or a pending order.
- **closed**: position gone (closed-trade history shows it), pending orders for it cancelled, stage 6 written.
- **void**: expired or rejected without a retry.

## Common failure modes

| Symptom | Fix |
| --- | --- |
| The user asks the Execution Trader directly to "just buy some" | Route to the Desk Lead; open a proposal; run the lifecycle quickly. |
| PASS computed from a stale quote | Risk re-reads; new PASS and expiry. |
| A "yes" that is not a direct reply to one ticket's approval post, or with more than one ticket open | Desk Lead asks which ticket, re-posts it, and waits for a clear reply. |
| prepare-trade preview differs from the ticket (units, costs, SL bounds) | Execution Trader aborts; Risk re-sizes; new revision and approval. |
| place-trade returns `pending` | Market closed or MIT waiting. Never place again; reconcile later; cancel only via ticket. |

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

