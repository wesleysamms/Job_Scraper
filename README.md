# 🧬 Bay Area MLE / DS Job Scraper

Two GitHub Actions workflows that scrape **Machine Learning Engineer, Data Science, and related AI/ML roles** in the SF Bay Area and email the results as an HTML digest.

## What It Does

### 1. Biotech LinkedIn digest — daily at 8pm PT, last 24h
Hits LinkedIn's public guest endpoint for SF Bay Area MLE/DS roles posted in the last 24 hours, then post-filters results to a **biotech company allowlist** derived from `CURATED_BIOTECHS` in `scrape_jobs.py` (10x Genomics, Twist, Maze, Freenome, Cytokinetics, Natera, Inceptive, Atomwise, Profluent, Eikon, Altos Labs, Arc Institute, Caribou, Octant, Genentech, Gilead). Add to that list to expand coverage.

Output goes to `jobs.json`, `jobs.md`, and `jobs.html`. Each run dedupes against the previously-committed `jobs.json` so the email surfaces only postings new since the last run.

> Why allowlist instead of LinkedIn's industry filter? The `f_I` industry parameter is silently ignored on the public guest endpoint (verified by probing IDs 12, 14, 16, 1763, 1862 — all returned identical non-biotech results).

### 2. LinkedIn MLE/DS watcher — every 2 hours, last 2h
Hits LinkedIn's public guest endpoint for SF Bay Area roles posted in **the last 2 hours** across multiple search terms, dedupes by job ID, and sorts by recency. Output goes to `linkedin_jobs.json`, `linkedin_jobs.md`, and `linkedin_jobs.html`.

Runs every 2 hours from **8am to 10pm Pacific time**. Cron is fixed in UTC; in PST (UTC-8) the schedule shifts to 7am–9pm PT — acceptable seasonal drift. Each run dedupes against the previous run so empty windows don't trigger an email.

> ⚠️ Uses the unauthenticated public guest endpoint only — **never** signs in with a user account and does not use LinkedIn cookies, tokens, or credentials.

## Keywords Matched

A title is included if it contains any of:

`machine learning engineer`, `ml engineer`, `mle`, `machine learning infra`, `ai engineer`, `mlops`, `research engineer`, `applied scientist`, `ai scientist`, `ml scientist`, `data scientist`, `data science`, `computational scientist`, `computational biologist`, `bioinformatics scientist`, `bioinformatics engineer`, `cheminformatics`

## Output Files

| File | Source | Description |
|---|---|---|
| `jobs.json` / `.md` / `.html` | Biotech LinkedIn digest | Allowlisted biotech-company roles in the last 24h, deduped against the previous run |
| `linkedin_jobs.json` / `.md` / `.html` | LinkedIn watcher | Roles posted in the last 2h, deduped against the previous run |
| `checked_companies.json` | (legacy) | Tracking file from earlier Wikipedia-based discovery |

The `.html` files are styled email-ready digests; the `.md` files render nicely on GitHub.

Both workflows keep a GitHub history of generated digests: result files are committed when changed, and each scheduled workflow still runs `git push`.

## Setup

### Gmail secrets (for email delivery)

In **Settings → Secrets and variables → Actions**:

| Secret | Value |
|---|---|
| `GMAIL_USER` | Gmail address |
| `GMAIL_APP_PASSWORD` | [Gmail App Password](https://myaccount.google.com/apppasswords) |

Both workflows email `GMAIL_USER` from `GMAIL_USER` via `smtp.gmail.com:465`.

### Run manually

From the **Actions** tab:
- *Biotech MLE Job Scraper* → Run workflow (biotech LinkedIn, last 24h)
- *LinkedIn MLE/DS Watcher* → Run workflow (general LinkedIn, last 2h)

Or locally:
```bash
python scrape_jobs.py --biotech-only   # biotech LinkedIn, last 24h, allowlist-filtered
python scrape_jobs.py --linkedin-only  # general MLE/DS LinkedIn, last 2h
python scrape_jobs.py                  # legacy curated Greenhouse/Workday/Phenom sweep
```

No third-party Python deps — uses only the standard library.

## Repo Structure

```
├── scrape_jobs.py                  # All scraping logic
├── jobs.{json,md,html}             # Curated biotech sweep output
├── linkedin_jobs.{json,md,html}    # LinkedIn last-hour output
├── checked_companies.json          # Legacy tracking file
├── deep-dive/                      # Notes / analysis
└── .github/workflows/
    ├── scrape_jobs.yml             # Daily 8pm PT — biotech LinkedIn (last 24h, allowlist)
    └── linkedin_watch.yml          # Every 2h, 8am–10pm PT — general LinkedIn (last 2h)
```

## ATS Endpoints Used

| ATS | Endpoint |
|---|---|
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true` |
| Workday | `https://{tenant}.wd1.myworkdayjobs.com/wday/cxs/{tenant}/{site}/jobs` (POST) |
| Phenom (Genentech) | `https://careers.gene.com/us/en/search-results` (HTML + JSON-LD) |
| LinkedIn | `https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search` (public guest) |
