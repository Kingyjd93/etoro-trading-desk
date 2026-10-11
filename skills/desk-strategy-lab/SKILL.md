---
name: desk-strategy-lab
description: "How the Strategist turns a user's trading idea into explicit rules, backtests it on eToro candle history with real costs (spread, fees, overnight and weekend charges), sanity-checks the result, and paper-trades it through the desk lifecycle. The desk ships no strategies. Use when the user wants to formalise, test or paper-trade an idea."
---
# Strategy lab

## Workspace

```
/workspace/trading-desk/strategies/<name>/
  RULES.md          written before any code
  backtest.py       one readable file
  results/          CSVs and summary.md
  NOTES.md          what was tried and abandoned
/workspace/trading-desk/data/   candle CSVs the Market Analyst or Strategist saved
```

## 1. Rules first

Write `RULES.md` with the user before code:

```markdown
# <name>
- universe: eToro instrument ids (e.g. 1001 AAPL, 1137 NVDA); settlement real|cfd; direction long|short
- data: interval (1d / 4h / ...), session handling (skip closed markets for stocks), timezone UTC
- entry: exact condition
- exit: exact condition; stop-loss rate or distance; take-profit if any
- sizing: reference to risk-limits.md (never "all cash")
- abandon if: <what would make the user drop the idea>
```

Outside ideas (blogs, social, books) become rules the same way: the desk tests the user's written version, not the source's claims. Cite the source as colour, never as authority.

## 2. Data

- Prefer `GET /api/v1/data/instruments/{id}/candles` (`interval`, `side=bid|ask|both`, `from`/`to`, `cursor`) for deep history; call `.../candles/coverage` first. This view can lag by days: check the last candle's `time`.
- Stitch recent bars from `GET /api/v1/market-data/instruments/{id}/history/candles/{direction}/{interval}/{candlesCount}` (up to 1000 bars; intervals OneMinute..OneWeek) and verify the overlap matches within a tick.
- Buy-side tests use ask candles; sell-side use bid; or add the measured live spread to mid. Never backtest on mid alone.
- Save the CSV under `/workspace/trading-desk/data/` with the exact request and the last candle time in the file header. Check gaps, duplicates and session breaks before use.

## 3. Backtest honestly

Include, every time:
- spread on every round trip (enter at ask, exit at bid for longs)
- transaction fees from a recent `POST /api/v2/trading/info/costs` for the instrument
- overnight and over-weekend fees for CFD holdings (usually zero for real 1x assets; still check)
- a slippage assumption on stop-loss exits and on stocks held over a session close or weekend (gaps)
- an out-of-sample period left untouched until the end

Report trade count, win rate, average win and loss in R, max drawdown, the cost share of gross P&L, and the OOS numbers separately. Do not lead with a headline return.

## 4. Sanity checks

- Future leakage (a signal that uses the close of the same bar it enters on).
- Survivorship: the current eToro instrument list is today's list.
- Parameter search: state the grid and the trial count; a result that exists only at one grid point is fragile.
- A result that flips sign when costs are added is a result: say so.
- Empty `leverageConfigs` for an instrument on this account means it is not tradeable through the API right now; do not backtest it as if it were.

## 5. Paper trade

Signals become proposals (`ET-...`) on the desk. The Risk Manager sizes; the user decides. Live rehearsal uses the eToro **demo** account only if the user asks, through the Execution Trader. The Strategist never sends on any account.

## 6. Abandon cleanly

Write one paragraph in `NOTES.md` saying why the idea was dropped. Leave the files; do not rewrite history.

## Never

- Ship a "desk strategy" or recommend one.
- Promise returns or annualise a short sample as an expectation.
- Call any write tool.
- Hide costs, trial counts or OOS failure.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

