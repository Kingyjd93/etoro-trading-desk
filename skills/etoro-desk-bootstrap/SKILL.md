---
name: etoro-desk-bootstrap
description: "Read-only runbook for setting up and verifying the eToro trading desk from the installed desk manual skills - check the skills, take an opening snapshot, confirm the connector and account, prepare /workspace/trading-desk, create or verify the six specialist Bots, their skills and the Trading Floor, record desk.md, and return a receipt. Never trades. Use for \"set up the desk\", \"check the desk\" or after anything is missing."
---
# eToro desk bootstrap

## Safety boundary

Bootstrap is research-only.

- eToro: read tools only (`get-*`, `execute-read`). Never `prepare-trade`, `place-trade`, `prepare-close`, `place-close` or `execute-write`; never a money, transfer, deposit or withdrawal route.
- No keys, passwords or tokens in chat or files. The connector holds credentials.
- Local writes only under `/workspace/trading-desk`.
- Creating the six specialist Bots, enabling their skills and creating the private Trading Floor group is in scope when the user asks for it (or agrees during `getting-started-etoro-desk`). Sharing anything publicly is not.
- If a capability is unavailable, give the user the exact manual step and continue independent work. Never mark an unverified step done.

## 1. Check the manual

The desk manual is a set of skills installed on the Desk Lead with the template; there is no manual folder. Confirm each is present with its full text (not a pointer):

- 14 desk and eToro skills: `desk-operating-model`, `desk-trade-lifecycle`, `desk-risk-limits`, `desk-execution-protocol`, `desk-monitoring`, `desk-post-trade-review`, `desk-incident-response`, `desk-strategy-lab`, `etoro-connection`, `etoro-api-reference`, `etoro-market-data`, `etoro-account`, `etoro-orders`, `etoro-desk-bootstrap`.
- 6 role skills: `desk-role-market-analyst`, `desk-role-research-analyst`, `desk-role-strategist`, `desk-role-risk-manager`, `desk-role-execution-trader`, `desk-role-trade-reviewer`.
- `desk-role-desk-lead` (this Bot's own role prompt) and `getting-started-etoro-desk` (the first conversation with a new owner).

A missing skill means the import is incomplete: tell the user; do not improvise a replacement. The manual derives from HyperGrok Trading Desk v1.5.0, commit `5eb1d9a82157440d64f25a72cd06d9f3977dc472`, used under the MIT license.

## 2. Opening snapshot

Follow the opening snapshot procedure in `desk-monitoring`: one `get-instruments-overview` call for the default watchlist, a table with UTC time. It is a snapshot, not a signal. If unavailable, say so; never substitute stale figures.

## 3. Confirm the connection and account

- `GetDynamicTools`: find the eToro connector namespace (`user-Etoro-trading` when this manual was written; use whatever the user's eToro connector is called and record it in `desk.md`). Status must be `ready`; if missing or `needsAuth`, the user connects eToro in Grok Bot's connector settings.
- `get-my-profile-and-scopes`: `authChannel` bearer; record `realCid` and `demoCid` as appropriate for the account type the user chose; list scope families; do **not** repeat name, date of birth or other personal fields.
- `get-my-portfolio-summary` (the chosen account): `accountCurrency` (normally USD), totals, pending orders.
- `GET /api/v1/trading/info/real/pnl` (real accounts): positions and `isNoStopLoss`. Report unprotected positions plainly.
- `get-all-routes` (tags `Trading - Real`, `Market Data`, `Data`): note `mutatingRouteIds` and `readOnlyPostRouteIds`; compare with `etoro-api-reference` and report differences.

## 4. Prepare the desk folder

```bash
mkdir -p /workspace/trading-desk/{proposals,briefs,research,strategies,data,journal/incidents,watch}
```

## 5. Build the team

The Desk Lead is this Bot (imported with the template). Create the other six with `CreateAgent`, one per role skill: Name, Job and Description exactly as in the skill's **Bot profile**, and the skill's full **System prompt** section as standing instructions. Read back what was stored; a matching name alone is not evidence. Record each native Bot id. Reuse an existing Bot only if the user says so and its profile matches.

| Role | Create from | Skills to enable |
| --- | --- | --- |
| Desk Lead (this Bot) | imported; role prompt in desk-role-desk-lead | desk-operating-model, desk-trade-lifecycle, desk-monitoring, etoro-connection, etoro-api-reference, etoro-desk-bootstrap, getting-started-etoro-desk, desk-role-* |
| Market Analyst | desk-role-market-analyst | etoro-market-data, etoro-api-reference, etoro-connection, desk-operating-model, desk-monitoring |
| Research Analyst | desk-role-research-analyst | desk-operating-model, desk-trade-lifecycle, etoro-market-data, etoro-api-reference |
| Strategist | desk-role-strategist | desk-strategy-lab, desk-operating-model, etoro-market-data, etoro-api-reference |
| Risk Manager | desk-role-risk-manager | desk-risk-limits, desk-trade-lifecycle, desk-monitoring, etoro-account, etoro-market-data, etoro-api-reference |
| Execution Trader | desk-role-execution-trader | desk-execution-protocol, desk-trade-lifecycle, desk-incident-response, etoro-orders, etoro-account, etoro-market-data, etoro-connection, etoro-api-reference |
| Trade Reviewer | desk-role-trade-reviewer | desk-post-trade-review, desk-incident-response, desk-operating-model, etoro-account, etoro-market-data, etoro-api-reference |

If skills cannot be enabled per Bot, tell each Bot in its instructions to load its skills by name, and record the skill status as `instructions-only`. Status per skill per Bot: `enabled`, `instructions-only` or `missing`.

Create a private group channel **Trading Floor** with the six floor Bots: Desk Lead, Market Analyst, Research Analyst, Strategist, Risk Manager, Execution Trader. The Trade Reviewer stays off the floor and works by DM. Record the group id and member ids from readback.

## 6. Native approval gate

The user, not a Bot, adds the Grok Bot Auto-review rule in settings, set to **Ask First**. Recommended wording:

> place, close, or edit any trade or order on my eToro account, or move, deposit, withdraw or transfer any money on eToro (any eToro place-trade, place-close or execute-write action, or any non-read eToro API request)

Optional: a second, stricter rule for money only if Auto-review offers one ("move, deposit, withdraw or transfer any money, or create sub-accounts, OAuth applications or agent-portfolio tokens on eToro"); include `prepare-trade` and `prepare-close` only if the user wants a card for previews too (they mint tokens but send nothing).

Record in `desk.md` exactly what was done and verified, for example: `Auto-review Ask First rule added by user <UTC>; read path confirmed unaffected; write coverage not live-tested (by design, no test order)`. Do not create a test order to prove it. If the user wants evidence of write coverage, the lowest-risk check is their call: the Execution Trader calls `place-trade` with a deliberately invalid token string. The tool rejects invalid tokens in-band and sends nothing; what matters is whether the Ask First card appears **before** the call. The user can deny the card to stop it. Record the result.

## 7. Record desk.md

```markdown
# Desk record
- created: <UTC>
- manual: desk manual skills (template; adapted from HyperGrok Trading Desk v1.5.0, 5eb1d9a)
- engagement level: research
- platform: eToro via Grok Bot connector <namespace>
- account: <real|demo>, <realCid or demoCid>, currency <accountCurrency>
- markets in scope: <e.g. US stocks and ETFs, 1x long>
- approval replies: <e.g. YES / NO / HOLD as a direct reply to the ticket's approval post; "approve ET-..." also works>
- bot ids: <role -> verified id or pending>
- trading floor id and member ids: <verified or pending>
- risk limits: not yet written | risk-limits.md vN
- standing approvals: <none | scope, granted UTC>
- exchange approval gate: <exact wording, or "not added">
- unprotected position deadline: <value>
- status: research-only
```

Record only what was observed. Pending stays pending. Never write the user's name or personal profile fields.

## 8. Receipt

Return: manual check result; snapshot with UTC time; connection status, account type and id, currency, scope families; positions without stop-loss; each Bot (id, created/reused/manual, profile match, instruction proof); each skill and status; Trading Floor id and members; desk.md path; anything unverified and the manual step for it.

The desk leaves research-only only when the user says so in writing, `risk-limits.md` exists, and the gate line in `desk.md` is accepted by the user.

---

Adapted for eToro from HyperGrok Trading Desk v1.5.0 (MIT, Galleon Labs)

