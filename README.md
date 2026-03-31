# UCI Dashboard — data pipeline

Scrapes [procyclingstats.com](https://www.procyclingstats.com) to build a local
DuckDB database of UCI road race results, rider points, and team standings for
the current season.

---

## Project layout

```
uci-dashboard/
├── scraper/
│   ├── __init__.py
│   ├── session.py          # requests.Session with browser headers
│   ├── cache.py            # disk-based HTML cache (TTL-aware)
│   ├── race_scraper.py     # catalogue + result page parsing
│   └── rider_scraper.py    # rider profile parsing
├── db/
│   ├── __init__.py
│   ├── schema.py           # DuckDB DDL + analytical views
│   └── upsert.py           # all write operations
├── pipeline.py             # main orchestrator (incremental + backfill)
├── scheduler.py            # long-running refresh daemon
├── requirements.txt
└── README.md
```

---

## 1. Prerequisites

- Python **3.11+**
- pip

---

## 2. Installation

```bash
# Clone / unzip the project, then:
cd uci-dashboard

# Create and activate a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

---

## 3. Populate the database for the first time

### Option A — backfill all races up to today (recommended first run)

This fetches every race from the current season that has already taken place,
including all stage results, GC, points, KoM, and youth classifications.

```bash
python pipeline.py --backfill
```

What it does:
- Fetches the WorldTour and ProSeries race catalogues
- For each race whose `date_start` is on or before today, scrapes all result pages
- Inserts riders, teams, and result rows into `uci.db`
- Caches all raw HTML in `.html_cache/` so re-runs are fast

**Expected duration:** 3–8 hours for a full season to date (polite 2–4 s delay
between requests). The process is safe to interrupt and resume — it picks up
where it left off via the `scrape_log` table and the on-disk HTML cache.

### Option B — backfill a specific previous year

```bash
python pipeline.py --backfill --year 2025
```

### Option C — backfill just one race (useful for testing)

```bash
python pipeline.py --backfill --race strade-bianche
python pipeline.py --backfill --race tour-de-france
```

The `--race` flag matches on a substring of the slug, so
`--race paris` would match `paris-roubaix`, `paris-nice`, etc.

---

## 4. Keep the database up to date

After the initial backfill, run the pipeline on a schedule to pick up new results.

### Manual incremental run

```bash
python pipeline.py
```

Only races currently in their "live window" (start − 1 day to end + 3 days)
are scraped. Completed races marked `is_final=true` in `scrape_log` are skipped.

### Automatic scheduled refresh

```bash
# Foreground (useful for testing):
python scheduler.py

# Background with logging:
nohup python scheduler.py > scheduler.log 2>&1 &
```

The scheduler automatically adjusts its interval:
- **March–October** (race season): every 2 hours
- **November–February** (off-season): every 6 hours

---

## 5. Querying the database

Open an interactive DuckDB shell:

```bash
python -c "import duckdb; con = duckdb.connect('uci.db'); con.sql('SELECT * FROM rider_standings LIMIT 20').show()"
```

Or from Python:

```python
import duckdb
con = duckdb.connect("uci.db")

# Top 20 riders by UCI points
con.sql("SELECT rank, full_name, team_name, nationality, total_points FROM rider_standings LIMIT 20").show()

# Team standings
con.sql("SELECT rank, name, uci_tier, total_points, riders_scoring FROM team_standings LIMIT 20").show()

# Points accumulated over the season for a specific rider
con.sql("""
    SELECT date_start, race_name, classification, uci_points, cumulative_points
    FROM points_timeline
    WHERE full_name ILIKE '%pogacar%'
    ORDER BY date_start
""").show()

# Races in the database with result counts
con.sql("SELECT name, date_start, race_class, finishers, total_points_awarded FROM race_summary ORDER BY date_start").show()
```

---

## 6. Useful maintenance queries

```sql
-- Check scrape progress
SELECT
    COUNT(DISTINCT race_id)  AS races_with_results,
    COUNT(*)                 AS total_result_rows,
    SUM(uci_points)          AS total_points_in_db
FROM results;

-- See which races are still pending
SELECT slug, year, date_start, date_end, is_stage_race
FROM races
WHERE race_id NOT IN (SELECT DISTINCT race_id FROM results)
ORDER BY date_start;

-- Force re-scrape of a race that was marked final too early
UPDATE scrape_log
SET is_final = false
WHERE url LIKE '%tour-de-france/2026%';

-- Check scrape_log for errors
SELECT url, http_status, error, scraped_at
FROM scrape_log
WHERE error IS NOT NULL OR http_status != 200
ORDER BY scraped_at DESC
LIMIT 20;

-- Fix stage count if auto-detection was wrong
UPDATE races SET stage_count = 21 WHERE slug = 'tour-de-france' AND year = 2026;
```

---

## 7. HTML cache management

All raw HTML is cached in `.html_cache/` with the following TTLs:

| Content type     | TTL      |
|------------------|----------|
| Live race result | 6 hours  |
| Race catalogue   | 24 hours |
| Rider profile    | 7 days   |

To clear the cache entirely (forces full re-fetch on next run):

```python
from scraper.cache import clear_all
print(clear_all(), "files deleted")
```

---

## 8. Notes on PCS data

- **Points source:** PCS reports UCI points directly in the result tables.
  Seed the `uci_scale` table from the official UCI regulations if you want
  to cross-validate or compute points for unscraped races.

- **Stage races vs one-day races:** Stage races produce results for each
  individual stage plus GC, points, KoM, and youth classifications — all
  carry separate UCI points. The pipeline scrapes all of them.

- **Mid-season team transfers:** `riders.team_id` reflects the rider's team
  at the time of their most recent profile scrape. The `results.team_id`
  column stores the team at the time of the race, which is the historically
  correct value — do not back-fill it when a rider changes team.

- **Respectful scraping:** The 2–4 second random delay between requests is
  genuine courtesy — PCS is a volunteer-run site. Do not reduce these delays.

## To Do
* check GC points - jonathon milan for example seems wrong 

