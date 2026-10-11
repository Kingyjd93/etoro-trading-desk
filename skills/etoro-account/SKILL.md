---
name: etoro-account
description: "How the desk reads the user's eToro account - portfolio totals and currency, balances across account types, open positions and pending orders, stop-loss coverage, closed-trade history, order lookups - and how to read the numbers without mistakes (unavailable is not zero, netProfit excludes fees, 0.0001 means no stop). Read-only. Use for \"how's the book\", sizing inputs, reconciliation and reviews."
---
# eToro account reads

All read-only. Report the account (real/demo), tool or route, and UTC time with every figure. Never repeat or store personal profile fields.

## Which tool for which question

| Question | Tool / route |
| --- | --- |
| Equity, available cash, unrealised P&L, holdings, copied traders, pending orders, **account currency** | `get-my-portfolio-summary` (`account: real`; `includePositions: true` for per-position rows) |
| Money in other account types (Cash, Crypto, Options, ...) | `get-my-balances` (read `fieldGuide` first; spendable field differs by type; amounts in each account's own currency) |
| Positions (with SL/TP) and pending orders, direct vs copied | `get-my-positions-and-orders` (`includeOrderDetail: true` for fills, max 10 orders) |
| **Stop-loss coverage** per position | `GET /api/v1/trading/info/real/pnl` -> `isNoStopLoss`, `stopLossRate`, `isTslEnabled` |
| One order's status and resulting positions | `GET /api/v2/trading/info/orders:lookup?orderId=` or `?referenceId=` |
| Closed trades | `get-my-trading-history` (`minDate` YYYY-MM-DD; default 90 days; `pageSize` <= 200) |
| Which account / scopes | `get-my-profile-and-scopes` (`realCid`, `demoCid`, `scopes`) |
| Portfolio breakdowns | `GET /api/v1/trading/info/portfolio`, `GET /api/v2/trading/info/instrument-breakdown`, `GET /api/v1/trading/info/aggregate-portfolio` |

## Account facts

- Account currency is normally **USD**; confirm it from `get-my-portfolio-summary` `accountCurrency`. The account type (real/demo) and its id are in `desk.md`.
- If the user thinks in another currency, convert only for display, with a live FX quote for that pair and its time, labelled as a conversion.
- Live account state (positions, stop-loss coverage, cash) belongs in the latest book check and `desk.md`, never in this skill. Re-read it.

## Reading the numbers correctly

- **Unavailable is not zero.** A 502 or statusCode 0 on a balance or portfolio read means it could not be read. Say "unavailable"; never report $0.
- `get-my-portfolio-summary` totals cover the **Trading account only**. Do not add Crypto or Cash account money to trading capacity.
- Portfolio rows are one consistent snapshot; do not re-price them against live quotes in the same table.
- Position `pnl` is unrealised; trading-history `netProfit` is realised and **excludes fees** - subtract `fees` for the loaded result.
- Stop-loss: in positions-and-orders, `stopLossRate` null means none set; in trading history and some rows a near-zero placeholder (`0.0001`) means none. The pnl route's `isNoStopLoss` is authoritative.
- Pending-order `status` strings (positions-and-orders) and `orders:lookup` `status.id` are different enums for the same state: 11 WaitingForMarket, 12 PendingTriggeredRate, 6 PendingCancel, 3 Filled, 7 Canceled, 8 Expired, 4 Rejected.
- Copied-trader (mirror) positions are separate from direct positions and can only be closed in full.
- Trading history on wide ranges (much more than 90 days) can time out: narrow the range.

## Rate limits (verified when this manual was written; re-check with `get-route-spec`)

- pnl / portfolio / instrument-breakdown: 60 / 60 s shared.
- orders:lookup / real orders / close-orders: 60 / 60 s shared.
- Everything else without a dedicated limit: default pool 60 / 60 s.

## Never

- Call any `/api/v1/money/*` route (transfers, deposits, withdrawals) - reading balances uses `get-my-balances` only.
- Write the user's name, date of birth or other personal fields into desk files or chat.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

