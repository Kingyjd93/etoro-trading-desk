---
name: desk-monitoring
description: "How the desk watches markets and the eToro account between trades using Grok Bot routines and connector polling (there is no streaming feed) - the desk brief, book and stop-loss checks, price and pending-order watches, alert conditions, rate-limit budgets, the opening snapshot, and what a watch may and may not do. Use for briefings, alerts, \"watch X\" or scheduled checks."
---
# Monitoring, briefs and watches

Monitoring reads and reports. It never sends. A routine that finds something actionable drafts a proposal or posts an alert; a human-approved ticket does the rest.

## Grok Bot routines

Each Bot can own routines (schedules or event triggers). Ask the owning Bot to create one with: schedule and time zone, inputs, expected output, approval boundary ("read and alert only"). Keep routines few and specific. The app keeps only recent run records, so a routine whose output matters also writes to `/workspace/trading-desk`.

| Routine | Owner | Schedule (suggested) | Output |
| --- | --- | --- | --- |
| Opening snapshot | Market Analyst | weekdays 09:15 New York time (15 minutes before the US open), set in the owner's time zone; adjust when daylight saving changes on either side | the opening snapshot below -> `briefs/YYYY-MM-DD-open.md` |
| Book and stop-loss check | Risk Manager | every 2-4 h while positions exist | book summary; unprotected positions as incidents |
| Pending-order check | Execution Trader | hourly while any order is pending | status per orderId via `orders:lookup` |
| Catalyst scan | Research Analyst | daily | changes to `research/calendar.md` |
| Daily journal close | Trade Reviewer | daily, after the US close (owner's time zone) | `journal/YYYY-MM-DD.md` summary |

The user picks which exist. Routines inherit the owning Bot's tools; their instructions say "read and alert only; never call prepare-trade, place-trade, prepare-close, place-close or execute-write".

## The desk brief

Assembled by the Desk Lead; facts first, every figure sourced and timestamped (UTC, with the owner's local time in brackets).

```
DESK BRIEF | 2026-01-15 21:00 UTC (<owner local time>) | eToro real <realCid>
account: equity $X (get-my-portfolio-summary 20:58 UTC), available cash $Y, unrealised P&L $Z
book: N positions; unprotected (isNoStopLoss true): <list> ; pending orders: <list with status>
markets (get-instruments-overview 20:59 UTC): BTC ask/bid, 1d %; ETH ...; AAPL (asOf = last session close); ...
calendar (Research Analyst): <dated items>
open items: <proposal ids and stage>
```

## When to post

- Requested updates: always answer, even "no change", with the evidence.
- Routine runs: post only on a change, a breach, a failure or stale data. Successful quiet runs append to the file and stay silent.
- An unprotected position, a read failure on account state, an unexpected position or order, or a pending order nobody expected: post immediately with the next owner.

## Watches (polling)

The connector has no WebSocket or push feed; watches poll.

- **Price watch**: `get-instruments-overview` with every watched instrument in one call; compare live bid (for longs' stops) or ask against the level. Alert when crossed, quoting rate, `asOf` and call time. A watch never "triggers" anything else.
- **Stop-loss coverage**: `GET /api/v1/trading/info/real/pnl`; any position with `isNoStopLoss: true` or `stopLossRate` at the `0.0001` placeholder is unprotected.
- **Pending orders**: `get-my-positions-and-orders` (status per order) and `GET /api/v2/trading/info/orders:lookup?orderId=` for one.
- **Account drawdown**: `get-my-portfolio-summary` totals versus start-of-day equity recorded in the journal.
- Price alerts on eToro itself (`/api/v1/price-alerts`) are writes to the user's account: only on the user's explicit request, via the Execution Trader, and they notify the user's eToro app, not the desk.

### Rate-limit budgets (verified when this manual was written; re-check with `get-route-spec`)

- Market-data group (rates, search, instruments, closing prices, market-data candles): 120 requests / 60 s shared.
- Data candles + coverage: 120 / 60 s shared.
- Eligibility: 20 / 60 s; costs: 20 / 60 s (each dedicated).
- Order lookups (`orders:lookup`, `real/orders/{id}`, `real/close-orders/{id}`): 60 / 60 s shared.
- PnL/portfolio group (`real/pnl`, `info/portfolio`, `instrument-breakdown`): 60 / 60 s shared.
- Order create/cancel group: 20 / 60 s shared.
- Default pool for everything else (including position SL/TP edits): 60 / 60 s.

All Bots share one connection, so these budgets are shared across the whole desk. Poll no faster than once a minute per watch; on 429 wait `retryAfterSeconds`.

### Market hours

- Crypto: trades every day; quotes should be seconds old.
- US stocks: regular session 09:30-16:00 New York time (convert to the owner's time zone and check daylight saving on both sides). Some have `.RTH` or extended-hours instruments with different ids. Outside hours the quote `asOf` is the last tick; `allowEntryOrders` says whether an order may be queued (it would sit `pending`).
- Index, commodity and FX CFDs: near 24/5 with daily breaks; weekends closed.
- A quote hours old during supposed trading hours is a data problem: say so.

## Opening snapshot (procedure)

A read-only market snapshot for the desk's default watchlist. Not a signal. Owner: Market Analyst (or any Bot during bootstrap). It uses one read tool.

| Symbol | eToro id | What it is |
| --- | --- | --- |
| BTC | 100000 | Bitcoin (crypto, trades daily) |
| ETH | 100001 | Ethereum |
| SPX500 | 27 | S&P 500 index CFD (non-expiry) |
| NSDQ100 | 28 | Nasdaq-100 index CFD (non-expiry) |
| AAPL | 1001 | Apple (US stock) |
| NVDA | 1137 | NVIDIA (US stock; NVDA.RTH 8760 is a different instrument) |
| GOLD | 18 | Gold CFD (non-expiry) |

The user may add their own holdings' instrument ids, and a home-currency FX pair found with instrument search, to the same call.

1. Note the UTC time.
2. Call `get-instruments-overview` once with all ids: `{"instrumentIds":[100000,100001,27,28,1001,1137,18]}` (plus any the user added). Never one call per instrument.
3. For each instrument record: ask, bid, spread and spread in bps of mid (`(ask-bid)/((ask+bid)/2) x 10,000`), `asOf` and its age at call time, previous close, eToro's 1d/1w/1m change %, allowed leverage long/short, min exposure, entry orders allowed, W-8BEN.
4. Flag: quotes older than 2 minutes (market closed, in a break, or stale feed - say which and why); empty leverage lists (not openable through the API on this account); `warnings` and `notFound*` entries; change % that does not match live versus previous close (eToro computes it from official closes; label it as eToro's).
5. Write the table to `/workspace/trading-desk/briefs/YYYY-MM-DD-open.md` with the tool name, xRequestId and UTC time, and post the summary.

```
OPENING SNAPSHOT | eToro via get-instruments-overview | called <UTC> (<owner local time>) | xRequestId <id>
| symbol | id | ask | bid | spread (bps) | asOf UTC | age | prev close | 1d % | 1w % | 1m % | lev long/short | min exp $ | entry orders | W-8BEN |
flags: ...
```

Never forecast, call anything a signal, reuse an old snapshot as current, or call any write tool.

## Hygiene

- Log watch results with UTC times under `/workspace/trading-desk/watch/`.
- A failed read is reported as failed, never as "all clear".
- A routine that owns a watch writes a heartbeat line each run; the Desk Lead checks heartbeats in the brief.

## Never

- A watch or routine that calls a write tool, or that "auto-protects" without the Execution Trader and the native card.
- Polling loops that exhaust a shared rate budget.
- Treating eToro social feeds as market data.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

