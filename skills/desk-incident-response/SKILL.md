---
name: desk-incident-response
description: "How the desk responds when an eToro send goes wrong or the account state does not match expectations - declare, freeze new sends, reconcile from eToro, playbooks for unknown/rejected/pending/unprotected/orphaned/unreachable cases, containment rules and how to close an incident. Use whenever a write result is not a clean match to the ticket."
---
# Incident response

An incident is anything that makes the desk unsure what the account holds or what just happened. Declare early; freeze new risk; reconcile from eToro; act only through approved tickets.

## Declare

The Execution Trader or Desk Lead posts on the Trading Floor:

```
INCIDENT | ET-... or none | 2026-01-15 14:35 UTC
what: place-trade returned unknown for order ...
impact: unknown whether AAPL 1001 is open; reservation $1,437 still held
freeze: no new risk until reconciled
owner: @Execution Trader
```

Set `status: incident` in `desk.md`. New risk tickets wait; exits and protective stop-losses continue under their own tickets (or the standing protective approval).

## Playbooks

### A. Unknown send result (timeout, statusCode 0, exception after place)

1. Do **not** prepare a new token.
2. If the confirmation token is unexpired: call `place-trade` / `place-close` again with the **same** token (idempotent).
3. Reconcile: `orders:lookup` by orderId or referenceId; `get-my-positions-and-orders`; `get-my-trading-history` for recent closes.
4. If found filled: treat as executed, update the proposal, lift the freeze for that ticket.
5. If nothing found and the token expired: the order did not land (or landed under an id you cannot see). Journal "unknown, no eToro record"; any replacement needs a fresh ticket, PASS and approval.

### B. Rejected or partially filled

- Rejected: quote `errorCode`/`errorMessage`. 623 = the account is blocked from trading; tell the user to contact eToro support; the desk cannot clear it. Other codes: Risk re-sizes or the idea is voided.
- Partial: record the filled units and positionId; the unfilled remainder is not automatically re-sent. A top-up is a new ticket.

### C. Position does not match expectation

Unexpected size, side, settlement, leverage or instrument. Freeze. Reconcile from `/real/pnl`, positions-and-orders and portfolio-summary. Do not "fix" by sending another order. The user decides whether to close or keep; closing is a ticket.

### D. Unprotected position (no stop-loss)

`isNoStopLoss: true` or `stopLossRate` at the 0.0001 placeholder. Alert Desk Lead and Risk Manager with exposure. If the position has an approved ticket stop, or `risk-limits.md` holds a protective-stop policy covering it, the Execution Trader may set that stop under the standing approval (reduce-only, if the user granted one in `desk.md`) after the native card. Without a written level: do not invent one; after the deadline in `desk.md`, tell the user the fastest fix is theirs in the eToro app.

### E. Orphaned or stuck pending order

An order nobody expected, or one still `pending`/`WaitingForMarket` long after the market opened. Reconcile via `orders:lookup`. Cancel only with an approved ticket (`DELETE /api/v3/trading/execution/orders/{orderId}`); confirm status 7 or 9. Status 6 PendingCancel is still in flight.

### F. Connector unreachable, 429, or degraded

Backoff. On 429 wait `retryAfterSeconds`. Do not hammer shared budgets. If the connector reports missing credentials or 401: the user reconnects it in Grok Bot's connector settings; never ask for keys in chat. While account state is unreadable, treat capacity as unavailable and reject new risk.

### G. Suspected misuse of the shared connector

A write that no desk Bot authorised, a money-transfer route call, a write that ran without a native approval card. Freeze. Tell the user immediately. Suggested containment (the user does these; the desk does not): revoke the connector in Grok Bot settings; change the eToro password; review recent activity in the eToro app; re-add the connector with the minimum scopes needed. Record what happened in `journal/incidents/`.

### H. Daily loss stop hit

Per `desk-risk-limits`: post once, set `status: no-new-risk`, reject new risk until the user resets in writing. Exits and protection continue.

## Containment rules

- Freeze new risk; do not freeze exits and protection.
- One writer still: only the Execution Trader calls write tools, and only behind the native card.
- Never "fix" by sending a compensating order without a ticket and approval.
- Never use money, transfer, deposit or withdrawal routes as incident response.
- Never clear a stop-loss.

## Close the incident

When the account state is known and matches the journal: post `INCIDENT CLOSED | <id> | <UTC> | cause: ... | action: ...`, restore `status` in `desk.md`, and hand the Trade Reviewer a one-page timeline for `journal/incidents/`.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

