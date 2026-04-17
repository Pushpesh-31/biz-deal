# Project Status — Paused 2026-04-17

The daily BizBuySell digest routine is **paused** pending a decision on data source. Scaffolding is done; the trigger exists; the end-to-end run does not yet produce a useful digest.

## What's built and working

- Repo scaffolded per CLAUDE.md: `config/filters.json`, `config/sources.json`, `config/scoring-weights.json`, `data/seen-listings.json`
- Repo pushed to https://github.com/Pushpesh-31/biz-deal (public)
- Claude Code Routine / remote trigger created: **`bizbuysell-digest`** — `trig_01NN5MXtYnBzxuR13NUKJvHC` — https://claude.ai/code/scheduled/trig_01NN5MXtYnBzxuR13NUKJvHC
  - Cron `30 11 * * *` (6:30 AM CDT), currently **disabled** — does not auto-fire
  - Runs `claude-sonnet-4-6`, tools: Bash, Read, Write, Edit, Glob, Grep, WebSearch, WebFetch
  - Source: this repo
- Cloud run executes cleanly through workflow steps 1–3

## What's blocked — three independent issues

1. **WebFetch is 403-blocked on BizBuySell, BizQuest, BusinessBroker.net.** Local and cloud both fail.
2. **WebSearch returns category/index pages, not listing snippets.** Today's run produced 1 usable listing from 6 searches.
3. **Routine-side plumbing gaps** (solvable, but lower priority than #1–2):
   - Gmail MCP connector not attached to the trigger — email send fails
   - Git push from the cloud sandbox returns 403 — needs Claude GitHub App installed on `biz-deal` with **write** scope (separate from the OAuth "Authorized Apps" entry on GitHub)

## Decision needed before continuing

Which direction for data sources?

- **A. Pivot to API/scrape-friendly marketplaces** — Acquire.com (has a real API), DealStream, BizNexus, Flippa. Keep the scoring / digest / email framework. ~70% of existing work carries over. Realistic output: 5–15 listings/day. *Recommended.*
- **B. Keep BizBuySell, accept thin data.** 0–2 listings/day most days. Low value.
- **C. Abandon scraping. Subscribe to BizBuySell's saved-search emails.** Move the scoring work into an inbox-filter that scores *incoming* alerts. Simpler, more reliable.
- **D. Add a Playwright-based local MCP** to bypass the 403. Requires laptop running — defeats "no laptop always on" goal.

## Pick-up checklist (once decision is made)

- [ ] Decide A / B / C / D
- [ ] If A: edit `config/sources.json`, update routine prompt to drop BizBuySell
- [ ] Install Claude GitHub App (https://github.com/apps/claude) on `biz-deal` with write access, so commits can push from the cloud run
- [ ] Attach Gmail MCP connector to the trigger (see trigger URL above; MCP connections field is currently empty)
- [ ] Re-run trigger manually; verify email arrives and `data/seen-listings.json` gets a new commit
- [ ] If clean, flip trigger `enabled: true`
