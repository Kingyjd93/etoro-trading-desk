# eToro Trading Desk for Grok Bot

A seven-Bot trading desk team for an eToro account, run inside Grok Bot:
Desk Lead, Market Analyst, Research Analyst, Strategist, Risk Manager, Execution Trader (the only writer) and Trade Reviewer.
Every trade goes idea -> evidence -> risk sign-off -> **your approval of that exact ticket** -> one send -> reconciliation -> review.

- `skills/` - 22 skills: 14 desk and eToro procedures, 7 `desk-role-*` role prompts, and `getting-started-etoro-desk` (first-run setup).
- `DESK-LEAD-MEMORY.md` - the Desk Lead's standing conventions.

## Use
Import the **eToro Desk Lead** Grok Bot template. On first run it installs these skills from this repo, asks a few setup questions, creates the other six Bots and the Trading Floor group, and stays read-only until you set risk limits and say go live.

Install manually: copy each `skills/<name>/SKILL.md` into your Grok Bot skills.

## Safety
The desk ships no strategies and makes no return claims. Nothing here is financial advice. The eToro connector's OAuth scopes are broad (including money movement); the desk's protection is procedural plus a Grok Bot Auto-review "Ask First" rule you add yourself.

## Credit
Adapted for eToro from [HyperGrok Trading Desk](https://github.com/galleonlabs/hypergrok-trading-desk) v1.5.0 (commit 5eb1d9a) by Galleon Labs, under the MIT License (see `LICENSE`). Not endorsed by Galleon Labs or eToro.
