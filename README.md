# BizBuySell Deal Scanner

Daily scanner that scrapes business-for-sale marketplaces, scores listings against buyer criteria, and emails a ranked HTML digest.

See **CLAUDE.md** for the full spec: architecture, buyer filters, scoring algorithm, listing schema, delta detection, and the routine prompt.

## Layout

- `CLAUDE.md` — source of truth (auto-loaded by Claude Code / routines)
- `config/filters.json` — buyer criteria (price range, SDE min, location bonuses)
- `config/sources.json` — enabled marketplaces and search queries
- `config/scoring-weights.json` — composite scoring weights
- `data/seen-listings.json` — persisted listing cache, updated and committed each run
