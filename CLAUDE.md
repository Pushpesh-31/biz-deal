# CLAUDE.md — BizBuySell Deal Scanner

## Project Overview

Automated daily deal scanner that scrapes business-for-sale marketplaces, scores listings against buyer criteria, and delivers a ranked HTML digest via email every morning. Runs as a **Claude Code Routine** on a scheduled trigger — no laptop required.

**Owner:** Pushpesh Sharma  
**Cadence:** Daily at 6:30 AM CT  
**Trigger type:** Scheduled routine  
**Connectors required:** Gmail  
**Repo:** This repo (stores seen-listings cache and config)

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Claude Code Routine (Anthropic Cloud)          │
│                                                 │
│  6:30 AM CT ──► Scrape Sources                  │
│                  ├── BizBuySell                  │
│                  ├── BizQuest                    │
│                  └── BusinessBroker.net           │
│                                                 │
│  ──► Parse & Extract Structured Data            │
│  ──► Deduplicate (title + location + price)     │
│  ──► Score (composite algorithm, 0-100)         │
│  ──► Diff vs seen-listings.json (NEW / DROP)    │
│  ──► Format HTML email digest                   │
│  ──► Send via Gmail connector                   │
│  ──► Update seen-listings.json in repo          │
└─────────────────────────────────────────────────┘
```

---

## Buyer Criteria (Default Filters)

These are the baseline filters. If a listing does not meet ALL of these, exclude it from the digest entirely.

| Parameter          | Value                    |
|--------------------|--------------------------|
| Asking Price       | $500,000 – $2,000,000    |
| Cash Flow (SDE)    | Minimum $150,000         |
| Owner Involvement  | Any (scored, not filtered)|
| Location           | USA (any state)          |
| Business Type      | All (no category filter)  |
| Listing Age        | Prefer ≤ 30 days         |

### Location Bonus (Optional)

If the listing is in Texas or within ~200 miles of Houston, apply a +5 scoring bonus. This is a soft preference, not a hard filter.

---

## Data Sources

### Primary: BizBuySell
- Search URL pattern: `bizbuysell.com/businesses-for-sale/` with price filters
- Also check: `bizbuysell.com/absentee-run-businesses-for-sale/` for semi-absentee/absentee listings
- This is the largest marketplace and should yield the most results

### Secondary: BizQuest
- Search URL pattern: `bizquest.com/businesses-for-sale/`
- Smaller inventory but sometimes has exclusive broker listings

### Tertiary: BusinessBroker.net
- Search URL pattern: `businessbroker.net/businesses-for-sale/`
- Good for regional broker listings not cross-posted elsewhere

### Future Sources (Add When Ready)
- **Acquire.com** — SaaS and digital businesses only
- **DealStream** — mid-market, often higher price points
- **LoopNet** — asset-heavy businesses with real estate
- **SearchFunder** — community-sourced off-market deals

To add a new source: add a new search query block in the scraping step and map its fields to the standard schema below.

---

## Listing Data Schema

Every listing, regardless of source, must be normalized to this schema before scoring:

```json
{
  "id": "bizbuysell-2419080",
  "source": "BizBuySell",
  "title": "Cabinetry Business – SE Florida",
  "url": "https://www.bizbuysell.com/business-opportunity/...",
  "location": {
    "city": "Boca Raton",
    "state": "FL",
    "region": "Southeast Florida"
  },
  "category": "Manufacturing / Construction",
  "asking_price": 1100000,
  "gross_revenue": 921000,
  "cash_flow_sde": 412000,
  "ebitda": null,
  "year_established": 2018,
  "owner_involvement": "Full-time",
  "owner_hours_per_week": 45,
  "employees": 8,
  "real_estate_included": false,
  "sba_prequalified": false,
  "franchise": false,
  "days_on_market": 12,
  "highlights": [
    "Premium cabinetry & custom furniture",
    "$412K SDE on $921K revenue"
  ],
  "first_seen_date": "2026-04-17",
  "previous_price": null
}
```

**Field extraction rules:**
- If a field is not available on the listing page, set it to `null` — do NOT guess or infer
- `owner_involvement` must be one of: `"Absentee"`, `"Semi-absentee"`, `"Full-time"`, or `null`
- `owner_hours_per_week` — extract if stated, otherwise leave `null`
- `cash_flow_sde` — BizBuySell calls this "Cash Flow"; map it here
- `ebitda` — only populate if explicitly stated separately from SDE
- `highlights` — extract 2-4 key selling points from the listing description

---

## Composite Scoring Algorithm (0–100)

Each listing receives a composite score. Higher = better fit for the buyer profile.

### Dimension 1: Price / SDE Multiple (max 30 pts)

```
multiple = asking_price / cash_flow_sde

≤ 2.0x  →  30 pts
≤ 2.5x  →  25 pts
≤ 3.0x  →  20 pts
≤ 3.5x  →  15 pts
> 3.5x  →   5 pts
null    →   0 pts (no SDE data)
```

### Dimension 2: Owner Involvement (max 25 pts)

```
Absentee         →  25 pts
Semi-absentee    →  18 pts
Full-time ≤ 40h  →  10 pts
Full-time > 40h  →   5 pts
null             →   8 pts (unknown, assume moderate)
```

### Dimension 3: SDE Margin (max 20 pts)

```
margin = cash_flow_sde / gross_revenue × 100

≥ 40%  →  20 pts
≥ 30%  →  15 pts
≥ 20%  →  10 pts
< 20%  →   5 pts
null   →   0 pts
```

### Dimension 4: Business Maturity (max 10 pts)

```
age = current_year - year_established

≥ 10 years  →  10 pts
≥  5 years  →   7 pts
<  5 years  →   3 pts
null        →   4 pts
```

### Dimension 5: Bonuses (max 15 pts)

```
SBA pre-qualified      →  +8 pts
Real estate included   →  +5 pts
Listed ≤ 7 days ago    →  +2 pts
Texas / Houston area   →  +5 pts (location bonus, can exceed 15)
```

### Score Interpretation

```
75–100  →  "Strong"    — worth immediate attention
55–74   →  "Moderate"  — review if time permits
0–54    →  "Review"    — included for completeness
```

---

## Deduplication Logic

Brokers frequently cross-list the same business on multiple platforms.

**Matching criteria (ANY of these = probable duplicate):**
1. Exact title match (case-insensitive, after stripping broker names)
2. Same state + same asking price + category overlap
3. Same city + asking price within 5% + SDE within 10%

**Resolution:** Keep the listing with the most complete data (most non-null fields). Note all source URLs in the record.

---

## Delta Detection (New / Price Drop / Removed)

The routine maintains `data/seen-listings.json` in this repo.

### On each run:

1. **Load** `seen-listings.json` — contains all previously seen listing IDs with their last-known price and first-seen date
2. **Compare** today's scraped listings against the seen set
3. **Flag**:
   - `NEW` — listing ID not in seen set → first appearance
   - `PRICE_DROP` — listing ID exists but `asking_price` decreased → flag with old price and % drop
   - `PRICE_INCREASE` — listing ID exists but price increased → note but don't highlight
   - `REMOVED` — listing ID was in seen set but not found today → may be sold or delisted (track for 7 days, then archive)
4. **Update** `seen-listings.json` with today's data
5. **Commit** the updated file to the repo

### seen-listings.json format:

```json
{
  "bizbuysell-2419080": {
    "title": "Cabinetry Business – SE Florida",
    "asking_price": 1100000,
    "first_seen": "2026-04-15",
    "last_seen": "2026-04-17",
    "price_history": [
      { "date": "2026-04-15", "price": 1200000 },
      { "date": "2026-04-17", "price": 1100000 }
    ]
  }
}
```

---

## Email Digest Format

Send via Gmail connector. The email should be well-formatted HTML that renders cleanly in Gmail, Apple Mail, and Outlook.

### Subject Line

```
BizBuySell Digest — {month} {day} · {new_count} new · Top: {best_listing_title_truncated}
```

Example: `BizBuySell Digest — Apr 17 · 3 new · Top: Cabinetry Business – SE Florida`

### Email Body Structure

```
HEADER
  "BizBuySell · Morning Digest"
  Date, listing count, filter summary

QUICK STATS BAR
  Total matched | Avg multiple | Top score | New today

SECTION: 🔥 NEW TODAY (if any)
  Listings with NEW flag, ranked by score
  Each listing card: title, location, price, SDE, multiple, 
  owner involvement, score badge, highlights, direct URL

SECTION: 📉 PRICE DROPS (if any)
  Listings with PRICE_DROP flag
  Show old price → new price, % drop

SECTION: 📋 TOP 10 OVERALL
  Top 10 by composite score (excluding duplicates from above sections)
  Same card format

FOOTER
  "Filters: $500K–$2M | Sources: BizBuySell, BizQuest, BusinessBroker.net"
  "Scoring: composite (multiple 30, owner 25, margin 20, maturity 10, bonus 15)"
  "Reply to this email to adjust filters or add sources"
```

### Email Design Rules

- Max width: 640px (email-safe)
- Fonts: system fonts only (Arial, Helvetica, Georgia) — no Google Fonts in email
- Colors: dark text on white, green for "Strong" scores, amber for "Moderate", gray for "Review"
- Each listing card: light border, clear metric grid, prominent score badge
- All links open in new tab (`target="_blank"`)
- Keep total email under 100KB to avoid Gmail clipping

---

## Routine Prompt Template

This is the prompt that drives the Claude Code Routine. It should be set as the routine's prompt configuration.

```
You are a business acquisition research assistant. Every morning, you scan 
business-for-sale marketplaces and deliver a scored, ranked digest via email.

## Your workflow:

1. SEARCH for business listings on these platforms:
   - BizBuySell (primary): search for listings $500K–$2M asking price
   - BizQuest (secondary): same price range
   - BusinessBroker.net (tertiary): same price range
   
   Use web_search to find listing pages. Fetch detail pages for 
   promising results to extract full data.

2. EXTRACT structured data for each listing matching the buyer criteria:
   - Asking price: $500K–$2M
   - Cash flow (SDE): minimum $150K
   - Any business type, any US location
   
   For each listing, extract: title, URL, location (city/state), category,
   asking price, gross revenue, cash flow/SDE, EBITDA (if stated), year
   established, owner involvement level, owner hours/week (if stated),
   employee count, SBA status, real estate inclusion, franchise flag,
   and 2-4 highlight bullets.

3. DEDUPLICATE across sources using title + location + price matching.

4. LOAD seen-listings.json from the repo. Compare today's listings.
   Flag: NEW (not previously seen), PRICE_DROP (price decreased),
   REMOVED (was seen before, not found today).

5. SCORE each listing using the composite algorithm (see CLAUDE.md 
   for full scoring breakdown — multiple, owner involvement, margin,
   maturity, bonuses). Total score 0–100.

6. FORMAT an HTML email digest with sections for New Today, Price Drops,
   and Top 10 Overall. Include quick stats, metric cards for each listing,
   score badges, and direct links.

7. SEND the email via Gmail to s.pushpesh@gmail.com with subject:
   "BizBuySell Digest — {date} · {new_count} new · Top: {best_title}"

8. UPDATE seen-listings.json with today's results and commit to repo.

## Important:
- Never fabricate listing data. If you can't extract a field, set it to null.
- Prefer detail pages over search result snippets for data accuracy.
- If a source is down or returns no results, note it in the email footer 
  and continue with other sources.
- Keep the email under 100KB total.
- Use system fonts only in the email HTML (no external font loading).
```

---

## File Structure

```
bizbuysell-scanner/
├── CLAUDE.md                  ← this file
├── data/
│   └── seen-listings.json     ← persisted listing cache (auto-updated)
├── config/
│   └── filters.json           ← buyer criteria (price range, SDE min, etc.)
│   └── sources.json           ← list of enabled sources with search patterns
│   └── scoring-weights.json   ← scoring algorithm weights (adjustable)
└── README.md                  ← project overview
```

### config/filters.json

```json
{
  "price_min": 500000,
  "price_max": 2000000,
  "sde_min": 150000,
  "location_country": "US",
  "location_state_filter": null,
  "location_bonus_states": ["TX"],
  "location_bonus_cities": ["Houston"],
  "location_bonus_points": 5,
  "exclude_franchises": false,
  "exclude_categories": [],
  "max_days_on_market": null,
  "owner_involvement_filter": null
}
```

### config/sources.json

```json
{
  "sources": [
    {
      "name": "BizBuySell",
      "enabled": true,
      "priority": 1,
      "search_queries": [
        "site:bizbuysell.com businesses for sale $500000 $2000000",
        "site:bizbuysell.com absentee run businesses for sale"
      ]
    },
    {
      "name": "BizQuest",
      "enabled": true,
      "priority": 2,
      "search_queries": [
        "site:bizquest.com businesses for sale $500000 $2000000"
      ]
    },
    {
      "name": "BusinessBroker.net",
      "enabled": true,
      "priority": 3,
      "search_queries": [
        "site:businessbroker.net businesses for sale $500000 $2000000"
      ]
    },
    {
      "name": "Acquire.com",
      "enabled": false,
      "priority": 4,
      "search_queries": [
        "site:acquire.com businesses for sale"
      ],
      "notes": "Enable when ready to add SaaS/digital businesses"
    }
  ]
}
```

### config/scoring-weights.json

```json
{
  "multiple": {
    "max_points": 30,
    "thresholds": [
      { "max": 2.0, "points": 30 },
      { "max": 2.5, "points": 25 },
      { "max": 3.0, "points": 20 },
      { "max": 3.5, "points": 15 }
    ],
    "default_points": 5
  },
  "owner_involvement": {
    "max_points": 25,
    "values": {
      "Absentee": 25,
      "Semi-absentee": 18,
      "Full-time-light": 10,
      "Full-time-heavy": 5
    },
    "full_time_light_max_hours": 40,
    "unknown_points": 8
  },
  "sde_margin": {
    "max_points": 20,
    "thresholds": [
      { "min_pct": 40, "points": 20 },
      { "min_pct": 30, "points": 15 },
      { "min_pct": 20, "points": 10 }
    ],
    "default_points": 5
  },
  "maturity": {
    "max_points": 10,
    "thresholds": [
      { "min_years": 10, "points": 10 },
      { "min_years": 5, "points": 7 }
    ],
    "default_points": 3
  },
  "bonuses": {
    "sba_prequalified": 8,
    "real_estate_included": 5,
    "new_listing_days": 7,
    "new_listing_points": 2
  }
}
```

---

## Setup Instructions

### Step 1: Create the repo

```bash
mkdir bizbuysell-scanner && cd bizbuysell-scanner
git init
mkdir data config
echo '{}' > data/seen-listings.json
# Copy config files from this spec
# Copy this CLAUDE.md to root
git add -A && git commit -m "Initial setup"
```

### Step 2: Create the routine

```bash
claude routine create \
  --name "bizbuysell-digest" \
  --repo ./bizbuysell-scanner \
  --schedule "daily 6:30am" \
  --connector gmail
```

Or create via claude.ai/code in the Routines UI.

### Step 3: Set the routine prompt

Paste the Routine Prompt Template from this file into the routine's prompt field.

### Step 4: Test

Run the routine manually once to verify:
- Search results are being parsed correctly
- Email arrives with proper formatting
- seen-listings.json is updated and committed

### Step 5: Go live

Enable the schedule. Monitor the first 3-4 days for:
- Token usage per run (adjust source count if too high)
- Email rendering across Gmail / Apple Mail
- Score distribution (adjust weights if everything clusters)

---

## Tuning Guide

### "I'm getting too many low-quality listings"
→ Increase `sde_min` in filters.json or set a minimum composite score threshold (e.g., only email listings scoring ≥ 50)

### "I want to focus on Texas only"
→ Set `location_state_filter: ["TX"]` in filters.json

### "I want to exclude restaurants and franchises"  
→ Set `exclude_franchises: true` and add `"Restaurant"` to `exclude_categories`

### "Scoring feels off — multiples matter more to me than owner involvement"
→ Adjust weights in scoring-weights.json (e.g., multiple: 40, owner: 15)

### "I want to add a new source"
→ Add an entry to sources.json with `enabled: true` and appropriate search queries

### "Email is getting clipped in Gmail"
→ Reduce to Top 5 instead of Top 10, or split into two sections across two emails

---

## Constraints & Limitations

1. **Web search accuracy** — Search results may not capture every new listing. Some listings may be missed if they don't rank well in search results. This is a best-effort scanner, not a comprehensive API integration.

2. **Data extraction** — Listing pages vary in structure. Some fields (owner hours, EBITDA) are frequently missing. The routine should never fabricate data to fill gaps.

3. **Rate limits** — Each routine run consumes tokens from the Max plan budget. Running 3 sources with ~20 search/fetch calls per source is typical. Monitor usage in claude.ai/settings/usage.

4. **Deduplication** — Heuristic matching may produce false positives (different businesses with similar names/prices) or false negatives (same business listed with different titles). The seen-listings cache helps over time.

5. **No authentication** — The routine uses public search and public listing pages. It cannot access gated content, broker-only details, or listings behind login walls.
