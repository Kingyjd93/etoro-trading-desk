---
name: desk-risk-limits
description: "How the Risk Manager writes the desk's risk limits with the user, sizes every proposed eToro trade from live account state and instrument eligibility on a stressed stop (spread, slippage, fees), checks the book and stop-loss coverage, and issues a PASS or REJECT with exact ticket fields. Use for setting or changing limits, sizing any trade, and \"how's the book\"."
---
# Risk limits and sizing

The user sets the limits in writing; the Risk Manager enforces them on every ticket with live data. eToro's own constraints (allowed leverage per settlement type and direction, minimum exposure, max units per order, stop-loss and take-profit bounds, available cash) always apply on top. All money is in the account currency, **USD**.

## 0. Desk ceilings

Deliberately loose backstops so a mistyped or corrupted limits file cannot authorise a catastrophic ticket. Not advice.

| Ceiling | Value |
| --- | --- |
| max risk per trade | 2% of equity |
| max total open risk (positions with stops + reservations) | 6% of equity |
| max leverage | 5x, and never above the instrument's allowed `leverageValues` |
| daily loss stop | -10% of start-of-day equity |
| stop-loss on every desk entry | mandatory, set on the position at open |
| standing approval that opens, adds to or flips a position | never |

The one action a standing approval may cover is **reduce-only protection**: setting or tightening a stop-loss on an existing position that has none, within eToro's allowed bounds, never deepening it past the allowed maximum (which would move cash into the position's margin) and never clearing it. The user's file may only be **stricter** than these ceilings. A looser value is not applied: REJECT with `gate failed: limits file exceeds desk ceiling <name>`, keep enforcing the ceiling, ask the user to edit the file. No Bot raises a ceiling.

## 1. Write the limits file

Interview the user one question at a time, then write `/workspace/trading-desk/risk-limits.md`. Version and date every change; only the user changes it, in chat.

```markdown
# Risk limits v1 - <YYYY-MM-DD> - set by user

- account: eToro <real|demo>, <realCid or demoCid>, currency USD
- equity basis: totalValue from get-my-portfolio-summary (Trading account), read live
- max risk per trade: 2% of equity            # setup default; loss if the stop is hit, stressed
- max total open risk: 6% of equity           # setup default
- max leverage: 1x                            # setup default scope: 1x long only
- max positions opened by the desk: 5
- allowed instruments: US stocks and ETFs the account can buy at 1x real (setup default), or a list by id, e.g. 1001 AAPL, 1137 NVDA
- allowed settlement: real (long); cfd short only if listed here
- stop-loss: mandatory on every desk entry; initial stop 8% below entry unless the ticket sets a tighter one   # setup default
- daily loss stop: -5% of start-of-day equity -> no new risk until the user resets in writing   # setup default
- send bounds: max slippage 10 bps vs ticket price; quote age <= 60 s; pending orders only if the ticket says so
- correlated clusters: US tech (e.g. AAPL, NVDA, MSFT and tech-sector ETFs) = one cluster, max 2 desk positions
- protective stop policy for positions the desk did not open: none   # e.g. "stop at -25% from open rate"
- standing approvals: none | reduce-only protective stop-losses (granted <UTC>)
- approval replies: YES / NO / HOLD on the ticket's approval post, or "approve ET-..."   # as recorded in desk.md
- unprotected position deadline: 15m          # then tell the user to fix it in the eToro app
```

Do not argue the user up or down; record what they choose and enforce it within the ceilings.

## 2. Size a trade

Inputs: instrument id, direction, entry type, stop rate, the limits file, live state. Missing or stale input is a REJECT with `missing input`.

### 2.1 Read live state

- Equity and cash: `get-my-portfolio-summary` (`totals.totalValue`, `availableCash`, `accountCurrency`). Use `get-my-balances` only for non-Trading accounts; never add Crypto or Cash account money to Trading capacity.
- Positions and stop-loss state: `get-my-positions-and-orders` (units, rates, pending orders) and `GET /api/v1/trading/info/real/pnl` (`isNoStopLoss`, `stopLossRate`; `0.0001` is the "no stop" placeholder).
- Instrument: `get-instruments-overview` (ask, bid, `asOf`) and `POST /api/v2/trading/info/eligibility` (`leverageConfigs`, `minPositionExposure`, `maxUnitsPerOrder`, `unitsQuantityType`, SL bounds).
- Costs: `POST /api/v2/trading/info/costs` (read-only what-if): markup, market spread, transaction fee, overnight fee, over-weekend fee, sdrt.
- Day P&L: start-of-day equity from the journal versus equity now.

Record each read's tool, UTC time and status. These are separate snapshots; if anything changed between them, refresh before a PASS.

### 2.2 Arithmetic (show every line in the PASS)

```
risk_usd       = equity x max_risk_pct
entry          = live ask (buy)  |  live bid (sellShort)
stop_distance  = |entry - stop|                                  (> 0; this is R per unit)
slip_stop      = max(2 x current spread, 10 bps of entry), rounded UP to the price tick;
                 widen for stocks held over a session close or weekend (gap risk) and say how
stop_fill      = stop - slip_stop (long)  |  stop + slip_stop (short)
per_unit       = |entry - stop_fill|
fixed_costs    = open fee + expected close fee + overnight fee x expected nights
                 + weekend fee x expected weekends (CFDs; usually 0 for real 1x assets)
units          = round_down((risk_usd - fixed_costs) / per_unit)   (to the precision eToro accepts;
                 whole units if unitsQuantityType is whole) - never risk_usd / stop_distance
amount         = units x entry
check          fixed_costs < risk_usd                   (else REJECT: costs exceed budget)
check          units <= maxUnitsPerOrder ; amount >= minPositionExposure and >= minPositionAmount
check          leverage in leverageConfigs[settlement, direction].leverageValues and <= limits and <= 5x
check          sl_pct = |entry - stop| / entry x leverage x 100 within [minStopLossPercentage, maxStopLossPercentage]
check          amount + open costs + pending reservations <= availableCash
check          open_risk_after = stressed risk of desk positions with stops + reservations + risk_usd
               <= limits.max_total_open_risk and <= 6% ceiling ; risk_usd <= 2% ceiling
check          positions and cluster counts ; instrument and settlement allowed ; stop present ; daily stop not hit
```

**Why the stress.** An eToro stop-loss closes at market when the bid (long) or ask (short) reaches it; the fill can be worse, and stocks can gap through a stop overnight or over a weekend. Sizing from the nominal distance prices a loss that cannot happen and overshoots the budget every time.

Worked (illustrative): equity $10,000, 0.5% budget = $50. AAPL buy at ask 340.55 (bid 340.45, spread 0.10), stop 330.00. slip = max(0.20, 0.34) -> 0.35; stop_fill 329.65; per_unit 10.90; fixed costs $4.00. units = 46.00 / 10.90 = 4.2201; amount $1,437.16; worst case $49.999. Sized naively (50 / 10.55 = 4.7393 units) the same stop costs $55.66, 0.557% of equity.

**Reservations.** A PASS reserves its risk and cash until the ticket is voided, expires unsent, or its result is reconciled. A `pending` or `unknown` order keeps its full reservation. Do not let two proposals spend the same cash.

**Positions without a stop.** They have no bounded risk. Count their full current value as at-risk exposure when reporting the book, and say so; they still consume cash and cluster capacity.

## 3. Book check ("how's the book")

- equity, available cash, unrealised P&L, day P&L versus the daily stop
- positions: instrument id and symbol, direction, settlement, leverage, units, open rate, current rate, P&L, value
- **stop-loss coverage per position** from `/api/v1/trading/info/real/pnl`: `isNoStopLoss`, `stopLossRate`, trailing flag. Any position with no stop is **unprotected** and reported as an incident to the Desk Lead
- pending orders and their status; copied traders (mirrors) separately
- cluster concentration; reservations by ticket

Timestamp everything. Save to `/workspace/trading-desk/briefs/YYYY-MM-DD-book.md` when asked.

## 4. When the desk hits a limit

- Daily loss stop: post once, set `status: no-new-risk` in `desk.md`, REJECT new risk until the user resets in writing. Exits and protection continue.
- Unprotected position: alert the Desk Lead and Execution Trader. If the position came from a desk ticket, or `risk-limits.md` holds a protective-stop policy covering it, the Execution Trader may set that stop under the standing approval. Otherwise alert with exposure and, after the deadline in `desk.md`, tell the user the fastest fix is theirs: set a stop or close in the eToro app.
- Limits file missing: the desk is research-only.

## Pitfalls

- Sizing from "what the cash allows" instead of from the stop.
- Using mid instead of ask/bid; forgetting the round-trip spread.
- Reading a missing `stopLossRate` field as a stop. Use the pnl route's `isNoStopLoss`.
- Treating an instrument with empty `leverageConfigs` as tradeable.
- Assuming a stop percentage is of price: eToro bounds are of margin (equal at 1x).
- Rounding units up to reach a minimum. If the minimum implies more risk than the budget, REJECT.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

