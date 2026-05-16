# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

Daily scraper that searches Bilibili for anime merchandise (谷子) videos across 9 product categories, classifies them by IP (franchise) and character, and produces an HTML trend report sent to the owner's parents for stocking decisions.

## Report format

Output is `reports/report_YYYY-MM-DD.html` — open in browser. It shows a card grid (thumbnail + IP + character + product type + velocity + views + Bilibili link) grouped into Breakout / Accelerating / Proven sections. Thumbnails are the video cover images set by creators, which for product review videos almost always show the specific badge/card/plush being reviewed.

The report has **two planned sections** (only section 2 is fully built):
1. **Sustained trends** (not yet built — needs 7+ days of data): at IP×character×product level, how many days in the past week had ≥1 breakout video for this combo? Combos with breakout on multiple days are stronger buy signals than one-day spikes. Ranked by breakout_count, not total views.
2. **Top velocity videos this week** (built): the breakout/accelerating/proven individual videos from this run, sorted by velocity.

## Sorting logic

Videos are fetched sorted by **view count** (`order=click`) within a 7-day publish window — NOT by publish date. This gets the top 50 most-viewed videos per category from the past 7 days. Velocity is then calculated on those to identify which proven-popular videos are still gaining vs. peaked. Sorting by pubdate was rejected because brand-new videos get artificial velocity spikes from subscriber notifications.

## Running the scraper

```bash
python3 main.py
```

Takes ~5 minutes. Writes `reports/report_YYYY-MM-DD.html` and logs to `logs/run_YYYY-MM-DD.log`.

**If the run gets blocked**, it aborts immediately. Check log for "IP is risk-controlled". Wait ~1 hour and retry. Do not retry in a loop — it extends the block.

## Cookies

Bilibili requires a logged-in session. Cookies live in `cookies.json` (not committed). Required fields: `SESSDATA`, `bili_jct`, `buvid3`, `buvid4`, `buvid_fp`, `DedeUserID`. Refresh from Chrome DevTools → Application → Cookies → bilibili.com when requests return HTML error pages.

## Architecture

```
main.py          orchestrates: fetch → classify → save → HTML report
scraper.py       Bilibili API calls with WBI signing, cookie auth, rate-limit handling
wbi.py           WBI request signing (required by Bilibili since 2023)
extractor.py     IP + character classification from video titles (two-pass)
ip_seeds.py      seed dictionary: IP name → [character names]
aggregator.py    classifies individual videos as Breakout/Accelerating/Proven
storage.py       SQLite persistence (anime_goods_intel.db)
report.py        builds the HTML report — no aggregation, video-first
config.py        all tuneable constants
```

**Note**: product-level aggregation was deliberately removed. The report surfaces individual videos, not rolled-up IP×product stats, because different videos about "鬼灭×义勇×吧唧" cover different specific badges — aggregating views across them was misleading. The right signal is breakout_count (how many videos independently hit breakout), not total views.

### IP/character classification (`extractor.py`)

Two-pass approach:
1. **First pass** (case-insensitive): IP name in title → return IP + first matching character
2. **Second pass**: any character name (len ≥ 2) in title even without IP name — catches "五条悟开箱" where the show name isn't mentioned

`ip_seeds.py` is the main thing to maintain. After each run, pull unrecognized titles, identify new IPs, add them, then reclassify in-place without re-scraping (see Workflow below). Some IPs have abbreviation aliases (`"星穹铁道"` → same chars as `"崩坏星穹铁道"`, `"鬼灭"` → `"鬼灭之刃"`, `"夏目"` → `"夏目友人帐"`, `"深空"` → `"恋与深空"`) because creators use short forms.

### Search keywords (`config.py`)

Ambiguous keywords are prefixed with `谷子` to cut noise (`"谷子吧唧"` not `"吧唧"`, which also means a lip-smacking eating sound). `CATEGORY_LABELS` strips the prefix for display. Specific keywords (一番赏, 流沙麻将, 镭射票) don't need the prefix.

### Video classification (`aggregator.py`)

Within each keyword batch, compute 67th percentile thresholds for view_count and view_velocity:
- **Breakout**: above threshold on both — high views AND gaining fast → stock now
- **Accelerating**: high velocity only — early mover not yet huge → watch
- **Proven**: high views only — established but not trending hard → safe bet

### Rate limiting

Randomised delays (12–22s between categories, 6–12s between pages) + full browser headers (`sec-fetch-*`, `sec-ch-ua`). On 412 or HTML risk-control page, `_risk_blocked` is set and the run stops immediately. Do NOT add retry logic for these responses.

## Workflow for each session

**Every session — run in this order:**

```bash
# 1. Scrape
python3 main.py

# 2. See unrecognized titles (auto-shows top 60 by views)
python3 post_run.py --show

# 3. Edit ip_seeds.py — add any new IPs you can identify from the titles
#    Focus on high-view titles. Ignore obvious noise (sports, food, lifestyle).

# 4. Reclassify DB + regenerate report
python3 post_run.py --reclassify

# 5. Open the HTML report
open reports/report_$(date +%Y-%m-%d).html
```

For a past date: `python3 post_run.py --show --date 2026-05-16`

## What to build next (sustained trends section)

Once 7+ days of data exist, build section 1 of the report: for each IP×character×product combo, query the DB for how many distinct run_dates had ≥1 breakout video. Rank by that count. A combo with breakout on 5 of the last 7 days is a strong buy signal. A combo with 1 breakout day is noise. This is the compound value — it cannot be replicated by a manual Bilibili search.

## Known noise floor

- **~20% genuinely off-topic**: food/life videos that happen to use the same keywords. The 谷子 prefix helps but doesn't eliminate this.
- **~25% creator OC content**: original characters by bilibili creators (财财, AI豪猫传, etc.) — not commercial IPs, accepted as baseline noise.
- **~46% of recognized videos lack character detection**: title names the IP but not the character. Shows as "—" in report. Improves as ip_seeds.py grows.
