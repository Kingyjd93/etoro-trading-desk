---
name: etoro-connection
description: "How the desk reaches eToro - the shared user-Etoro-trading MCP connector, its broad OAuth scopes, which tools are read and which write, the forbidden money and account-admin routes, real versus demo, the native approval gate, and what to do when authentication fails (the user reconnects; never take keys in chat). Use before any eToro call and whenever access, scopes or auth come up."
---
# eToro connection

## What it is

One Grok Bot MCP connector, namespace `user-Etoro-trading` (the name when this manual was written; if the user's eToro connector shows a different namespace, use that and record it in `desk.md`), authenticated by OAuth bearer (`authChannel: bearer`). Every Bot on the desk uses the same connection. Credentials live on the connector; no tool takes a key or token as an argument. Discover tools with `GetDynamicTools` on the namespace; call them with `CallDynamicTool` and a plain `mcpDetails.description` of what the call does.

## Scopes (as typically granted; check yours)

`get-my-profile-and-scopes` returns the granted scopes. When this manual was written they were broad: real and demo account read/write umbrellas (`etoro-public:real:write`, `etoro-public:trade.real:write`, demo equivalents), **money** (`money:transfer`, `money:withdraw`, `money:deposit`, `money:write`, `money.transfer:write`), sub-accounts write, SSO applications write, agent portfolios write, KYC write, feed (social posting) write, watchlists, price and P&L alerts write, crypto write, rebalancer write, plus reads. Consequence: **nothing at the eToro level limits which Bot can trade or move money.** The desk's protections are procedural and the native gate:

1. **Native gate**: the user's Grok Bot Auto-review rule, "Ask First", covering any `place-trade`, `place-close` or `execute-write` on this connector and any non-read request to `public-api.etoro.com`. Recorded in `desk.md`.
2. **One writer**: only the Execution Trader calls `prepare-trade`, `place-trade`, `prepare-close`, `place-close`, `execute-write`.
3. **Forbidden routes**: below, for every Bot.

## Tools: read vs write

| Tool | Kind | Who |
| --- | --- | --- |
| `get-instruments-overview` | read (quotes, performance, eligibility summary; 1-100 instruments per call) | all |
| `get-my-portfolio-summary` | read (Trading account totals, holdings, copied traders, pending orders; currency) | all |
| `get-my-positions-and-orders` | read (positions with SL/TP, pending orders and status) | all |
| `get-my-balances` | read (every account type; needs `money.balance:read`) | Risk Manager, Desk Lead |
| `get-my-trading-history` | read (closed trades; `netProfit` excludes fees) | all |
| `get-my-watchlists` | read | all |
| `get-my-profile-and-scopes` | read (cids, scopes; also personal fields - never repeat or store them) | Desk Lead, Execution Trader |
| `get-trader-profile-summary` | read (public investor stats) | Research Analyst |
| `get-tags`, `get-all-routes`, `get-route-spec` | read (catalog) | all |
| `execute-read` | read: any GET, plus read-semantics POSTs (`getCost`, `getInstrumentEligibility`, `postAggregatedPortfolio`) | all |
| `prepare-trade`, `prepare-close` | preview that mints a confirmation token; not a send, but **Execution Trader only** | Execution Trader |
| `place-trade`, `place-close` | **write**: sends the prepared order | Execution Trader, behind native card |
| `execute-write` | **write**: any POST/PUT/PATCH/DELETE | Execution Trader, behind native card, allowed routes only |

`get-all-routes` returns `mutatingRouteIds` (state-changing) and `readOnlyPostRouteIds`; a route in the first list is a write no matter how it is called.

## Allowed write routes (Execution Trader, approved ticket only)

- Open: `prepare-trade` -> `place-trade` (never a raw order route).
- Close: `prepare-close` -> `place-close`.
- Edit stop-loss/take-profit: `PATCH /api/v2/trading/positions/{positionId}` (`updateRealPosition`).
- Cancel pending order: `DELETE /api/v3/trading/execution/orders/{orderId}` (`deleteTradingExecutionOrdersByOrderIdV3`).
- Price alerts (`POST/PATCH/DELETE /api/v1/price-alerts...`): only when the user explicitly asks for that alert.

## Forbidden routes (every Bot, always)

- `/api/v1/money/*` - transfers, transfer eligibility probing for a transfer, deposits, withdrawals (`postMoneyTransfer` and friends). Read-only views of these are unnecessary for the desk; leave them alone.
- Sub-accounts (`/api/v1/sub-accounts/*`), SSO applications and scopes (`/api/v1/sso/*`), agent portfolios and their tokens, copy trading start/stop/adjust, social feed posts/comments/likes, KYC, watchlist writes, notification writes, rebalancer writes, demo-account writes outside an approved rehearsal.
- Raw order routes (`POST /api/v1/trading/execution/...`, `POST /api/v2|v3/trading/execution/orders`): use prepare/place.

A Bot asked to call any of these refuses, says why in one line, and tells the user to do it in the eToro app if they want it.

## Real vs demo

- Real: `realCid` from `get-my-profile-and-scopes`, recorded in `desk.md`. Account currency **USD** (from `get-my-portfolio-summary` `accountCurrency`).
- Demo: `demoCid`; `account: demo` on prepare-trade/prepare-close and the read tools. Rehearsal only, still with a ticket and approval.
- `get-instruments-overview` eligibility reflects the connection's account, not a chosen one.
- Every message about an action names the account.

## Authentication failures

- Connector missing, or `GetDynamicTools` shows `needsAuth`: the user connects eToro in Grok Bot's connector settings (OAuth sign-in). The desk cannot do it for them.
- `error` mentioning missing credentials, a 401, or "token failed verification": the connection needs reconnecting. Tell the user: open Grok Bot's connector settings, reconnect eToro (OAuth), then re-run `get-my-profile-and-scopes` and confirm `realCid` matches `desk.md`.
- Never ask for, accept, paste or store passwords, API keys (`x-api-key`, `x-user-key`) or tokens in chat or files. If the user pastes one, tell them to rotate it in eToro and do not use it.
- A changed `realCid` after reconnecting is an incident: stop, tell the user, do not trade.

## Errors and limits

- Upstream 4xx/5xx come back in `statusCode`/`body`, not as a tool error; `statusCode 0` = timeout or unreachable (safe to retry reads).
- A 502 or statusCode 0 on a balance or portfolio read is **unavailable**, never zero.
- On 429 wait `retryAfterSeconds`. All Bots share one connection, so they share every rate budget.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

