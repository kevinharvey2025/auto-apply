# Ingestion Plan: SimplifyJobs → Jobs Database

This is the detailed build plan for **Stage 1 (Ingest)** in `docs/PLAN.md`. An hourly job pulls the SimplifyJobs listings, opens every relevant posting on the company's own site, extracts the facts that matter (location, position, pay, work model, dates, requirements and more), and keeps the local jobs database current: new jobs added, changed jobs updated, closed jobs marked closed.

_Written 2026-10-09. Numbers are from the repo snapshot of that day._

---

## 1. What we're ingesting

| | |
|---|---|
| Source | `SimplifyJobs/Summer2027-Internships`, file `.github/scripts/listings.json` |
| Total listings | 17,211 (all years, open and closed) |
| Active + visible | 4,626 |
| **Active in our 4 categories** (Software, AI/ML/Data, Quant, Product) | **3,655**: this is the initial backfill |
| Change rate | ~110 listings new/updated per day (~850/week), **about 5 per hour** |

After the one-time backfill, a typical hourly run only has to fetch a handful of new or changed postings, plus a slice of the daily re-check (§5.4). That's light enough to run every hour on the Mac without anyone noticing.

### Where the 3,655 relevant postings live (what we have to parse)

| Application system | Postings | How we read it | Pay usually available? |
|---|---|---|---|
| Workday | 1,337 | Public JSON endpoint (`/wday/cxs/...`) | In description text (pay-transparency sentence) |
| Custom career sites | 570 | HTML → schema.org JSON-LD → rendered page | Sometimes (JSON-LD or text) |
| Greenhouse (boards + `gh_jid` on company domains) | 410 + 196 | Public Job Board API | **Structured** (`pay_input_ranges`) |
| iCIMS | 298 | HTML + JSON-LD | Text |
| Oracle Cloud HCM | 251 | Public REST endpoint | Text |
| Ashby | 248 | Public Posting API | **Structured** (`compensation`) when the company shows it |
| Lever | 88 | Public Postings API | **Structured** (`salaryRange`) when set |
| SuccessFactors, SmartRecruiters, Workable, Rippling, Eightfold, Taleo, Jobvite, Avature | ~260 total | SmartRecruiters has a public API; the rest use HTML + JSON-LD, with a rendered-page fallback | Text |

**Verified on 2026-10-09** against real postings from your tracker:
- **Greenhouse** (`boards-api.greenhouse.io/v1/boards/figma/jobs/6143238004?pay_transparency=true`): returns `title`, `location`, `offices`, `departments`, `first_published`, `updated_at`, `requisition_id`, `application_deadline`, full HTML `content`, and **`pay_input_ranges`** (Figma: $55/hr, plus a housing stipend mentioned in the pay blurb).
- **Ashby** (`api.ashbyhq.com/posting-api/job-board/notion?includeCompensation=true`): returns `location`, `secondaryLocations`, `workplaceType` (Hybrid/Remote/OnSite), `employmentType`, `isRemote`, `publishedAt`, `descriptionPlain`, `jobUrl`/`applyUrl`, and a `compensation` block. The block is empty when the company hides pay.
- **Workday** (`mastercard.wd1.myworkdayjobs.com/wday/cxs/mastercard/Campus/job/...`): JSON with `title`, `location`, `additionalLocations`, `timeType`, `startDate`, `jobReqId`, `country`, and `jobDescription` HTML. The description contained *"National Salary Range … $35–$46 hourly"*, so pay has to be parsed from text.

---

## 2. Architecture

```
 launchd (every hour, on the Mac)
        │
        ▼
 ingest/run.py ──► 1. SYNC      fetch listings.json (ETag / commit SHA; skip if unchanged)
                   2. DIFF      upsert by Simplify id → new / changed / closed / unchanged
                   3. FILTER    category ∈ {Software, AI/ML/Data, Quant, Product}, term filter, active
                   4. ENRICH    queue: new + changed + due-for-recheck
                                 └─ per-ATS adapter fetches the company page/API
                                 └─ extractors → structured fields (pay, location, …)
                   5. STORE     jobs + job_details + raw snapshot (hash) + history
                   6. REPORT    ingest_runs row; Telegram digest of new relevant jobs (later)
```

- **Python 3.12**, `httpx` (async, HTTP/2), `selectolax` or BeautifulSoup for HTML, `extruct` for JSON-LD, Playwright only for the fallback, SQLAlchemy + SQLite, `tenacity` for retries.
- One entry point: `uv run python -m app.ingest.run [--backfill] [--limit N] [--only-ats greenhouse] [--dry-run]`.

---

## 3. Data model

```
companies          id, name, normalized_name, aliases[], ats, ats_board_token, careers_domain
jobs               id (ours), simplify_id (unique), company_id, title, category, terms[],
                   simplify_url, apply_url (company site, cleaned), ats, ats_job_id,
                   status (open|closed|unknown), simplify_active, first_seen_at, last_seen_at,
                   simplify_date_posted, simplify_date_updated, closed_at
job_details        job_id (1:1, latest), title_on_site, department, team, req_id,
                   locations[] (city, state, country, raw), work_model (onsite|hybrid|remote),
                   employment_type, time_type,
                   pay_min, pay_max, pay_currency, pay_interval (hour|year|month|week),
                   pay_hourly_min/max (normalized), pay_source (structured|text|none), pay_text (raw),
                   perks[] (housing, relocation, travel stipend, …),
                   start_date, end_date, duration_weeks, season/term (from page),
                   class_standing[] (freshman…senior), grad_date_window, degree_levels[], majors[],
                   gpa_min, citizenship_required, clearance_required, sponsorship (yes|no|unknown),
                   application_deadline, posted_at (site), updated_at (site),
                   description_text, requirements_text, responsibilities_text, skills[] (later, model B),
                   parse_confidence, extracted_at, extractor_version
job_snapshots      job_id, fetched_at, http_status, content_hash, raw_payload (gz), adapter
job_changes        job_id, at, field, old, new          (title/pay/location/status diffs)
ingest_runs        id, started_at, finished_at, source_sha, listings_total, new, changed,
                   closed, enriched_ok, enriched_failed, duration_s, errors_json
fetch_errors       job_id, at, adapter, http_status, error, attempt
```

Rules:
- **The Simplify `id` is the identity.** Upsert on it. Our own `jobs.id` is what the rest of the app references.
- **Two kinds of status:** `simplify_active` (what the repo says) and `status` (what the company site says). A job is `closed` if the repo marks it inactive **or** the company page returns 404/410 or "no longer accepting applications".
- **Raw payloads are kept** (gzipped, deduplicated by hash), so extractors can be improved later and re-run without refetching.
- **Changes are logged.** Pay or location changes and closures go into `job_changes` for the dashboard.

---

## 4. Field extraction

Every adapter returns the same `RawPosting` (title, locations, description HTML/text, dates, any structured pay, req id, department, raw JSON). One shared extraction pipeline then fills `job_details`:

| Field | Structured source (preferred) | Text fallback |
|---|---|---|
| **Position/title** | API title | `<h1>` / JSON-LD `title` |
| **Location(s)** | Greenhouse `location`/`offices`; Ashby `location` + `secondaryLocations`; Workday `location` + `additionalLocations`; JSON-LD `jobLocation` | Simplify `locations` |
| **Work model** | Ashby `workplaceType`; Workday `remoteType`; JSON-LD `jobLocationType` | Keywords: "hybrid", "remote", "in-office N days" |
| **Pay** | Greenhouse `pay_input_ranges`; Ashby `compensation.compensationTiers`; Lever `salaryRange`; JSON-LD `baseSalary` | Regex parser (below) |
| **Perks** | — | Keywords: housing stipend, relocation, travel reimbursement, sign-on |
| **Start/end, duration** | Workday `startDate` is the *posting* start, not the internship start, so it's ignored | Patterns: "May–August 2027", "12-week", "10 to 12 weeks" |
| **Class standing / grad window** | — | "rising junior", "sophomore", "graduating between Dec 2027 and Jun 2028" |
| **Degree / major / GPA** | JSON-LD `educationRequirements` | "pursuing a BS in…", "GPA of 3.0 or higher" |
| **Citizenship / clearance / sponsorship** | — | "U.S. citizenship required", "Secret clearance", "will not sponsor" (important for defense and gov-adjacent postings) |
| **Deadline** | Greenhouse `application_deadline`; JSON-LD `validThrough` | "apply by…" |
| **Posted / updated** | Greenhouse `first_published`/`updated_at`; Ashby `publishedAt`; JSON-LD `datePosted` | Simplify dates |
| **Responsibilities / requirements** | — | Section splitter on headings ("What you'll do", "Qualifications", "Preferred") |

**Pay parser** (no LLM, unit-tested):
- Recognizes `$35–$46 hourly`, `$35 - $46/hr`, `$42.50 per hour`, `$120,000 – $150,000 annually`, `$8,000/month`, `USD 45-55/hour`, and single values.
- Normalizes to `pay_interval` and also to **hourly** (yearly ÷ 2,080; monthly ÷ 173.3; weekly ÷ 40), so jobs are comparable in the dashboard.
- Keeps the raw sentence in `pay_text` and records `pay_source`. When several ranges are listed (e.g. by location), it stores the full span and keeps all ranges in `pay_text`.
- Sanity bounds: an hourly value outside $10–$200 is flagged for review, not stored as-is.

**Skills** are filled later by the trained skill extractor (model B in `docs/PLAN.md`) from `description_text`. Ingestion just stores clean text.

---

## 5. Hourly run in detail

### 5.1 Sync
1. Fetch `listings.json` (raw GitHub URL, with `If-None-Match` ETag) **or** do a shallow `git fetch` and compare the commit SHA with the last run's `source_sha`.
2. If unchanged: skip to the re-check queue (5.4). Otherwise parse the JSON.

### 5.2 Diff
- **New** = id not in DB → insert.
- **Changed** = `date_updated` moved, or url/title/locations/active differ → update + log in `job_changes`.
- **Closed** = `active` false → set `status=closed`, `closed_at`.
- Normalize as we go: category spellings ("Software Engineering" → Software), company aliases, strip `utm_*`/`ref` params, and turn Ashby `/application?embed=true` URLs into clean job URLs.

### 5.3 Filter
Only jobs that pass the filter get enriched: active, in the 4 categories, and in the configured terms (open question from `docs/PLAN.md`; default **Summer 2027** + co-op terms you approve). Everything else is still stored from the Simplify data, just not fetched. That keeps the repo history complete for later analysis.

### 5.4 Enrichment queue (what gets fetched this hour)
1. New relevant jobs.
2. Changed relevant jobs.
3. **Re-check slice:** each open job is re-fetched about once every 24h, spread across the hours (≈ 3,655 ÷ 24 ≈ 150/hour after backfill). This catches closures and pay edits that the repo doesn't reflect.
4. Retries of failed fetches, with backoff (1h, 3h, 12h, then give up and mark `unknown`).

### 5.5 Politeness and reliability
- **Per-host concurrency of 2**, global concurrency of 8, and a jittered delay of 0.5–1.5s between requests to the same host. A real browser-like User-Agent with contact info.
- Respect `429`/`Retry-After`. Use a circuit breaker per host: after 5 consecutive failures, pause that host for the rest of the run.
- Cache Ashby boards per org per run, since one board call returns every job at that company.
- **No CAPTCHA solving, no logins.** Pages behind a wall are marked `needs_manual_fetch`.
- **Lock file** so a slow run never overlaps the next hour's run.
- **Backfill mode** (`--backfill`) runs the 3,655 jobs once, in batches of ~500, ordered by posting date (newest first). At these rates that's about 1–2 hours total.

### 5.6 Scheduling (macOS)
- A **launchd agent** (`~/Library/LaunchAgents/com.kevin.autoapply.ingest.plist`) with `StartInterval = 3600` runs `uv run python -m app.ingest.run`. It runs whether or not the dashboard is open. If the Mac was asleep, launchd runs it once on wake rather than piling up missed runs.
- Logs go to `data/logs/ingest-YYYY-MM-DD.log`. Each run writes an `ingest_runs` row, so the dashboard can show "last run 12 min ago · 6 new · 1 closed · 0 errors".
- When the project moves to the cloud later, the same command runs from cron or a scheduled worker.

---

## 6. Adapters (build order)

Each adapter is a small class: `matches(url) → bool`, `fetch(job) → RawPosting`. The list is ordered by **coverage × ease**:

| # | Adapter | URL patterns | Fetch | Notes |
|---|---|---|---|---|
| 1 | **Greenhouse** | `boards.greenhouse.io/{board}/jobs/{id}`, `job-boards.greenhouse.io/...`, `.../embed/job_app?token={id}`, company domains with `?gh_jid={id}` | `GET boards-api.greenhouse.io/v1/boards/{board}/jobs/{id}?pay_transparency=true` | For `embed`/`gh_jid` links the board token isn't in the URL: try a slugified company name against `/v1/boards/{slug}` and cache the hit in `companies.ats_board_token`. Otherwise read the token from the page HTML. |
| 2 | **Workday** | `*.myworkdayjobs.com/...`, `*.myworkdaysite.com/recruiting/...` | `GET https://{host}/wday/cxs/{tenant}/{site}/job/{path}` (Accept: JSON) | `tenant`/`site` come from the URL. Pay comes from `jobDescription` text. |
| 3 | **Ashby** | `jobs.ashbyhq.com/{org}/{uuid}` | `GET api.ashbyhq.com/posting-api/job-board/{org}?includeCompensation=true` (cached per org) and pick the uuid | A uuid missing from the board means the job is closed. |
| 4 | **Generic JSON-LD** | anything else (custom, iCIMS, Avature, Eightfold, Rippling, Workable, SuccessFactors, Taleo, Jobvite) | `GET` the page → `extruct` → schema.org `JobPosting` | Covers many custom sites in one adapter, because career pages publish JSON-LD for Google Jobs. |
| 5 | **Oracle Cloud HCM** | `*.oraclecloud.com/hcmUI/CandidateExperience/.../job/{id}` | Public `recruitingCEJobRequisitionDetails` REST endpoint (site number from the URL) | Exact endpoint to confirm in Stage 1 testing. |
| 6 | **Lever** | `jobs.lever.co/{site}/{id}` | `GET api.lever.co/v0/postings/{site}/{id}` | `salaryRange`, `categories.location`, `workplaceType`. |
| 7 | **SmartRecruiters** | `jobs.smartrecruiters.com/{company}/{id}` | `GET api.smartrecruiters.com/v1/companies/{company}/postings/{id}` | |
| 8 | **Rendered fallback** | anything 4 couldn't parse | Playwright headless, wait for the main content, then run the text extractors | Slow, so it's capped at ~30 per run. |

**Coverage target:** adapters 1–4 cover about 75% of relevant postings, and 1–8 cover about 95%. Anything left over keeps its Simplify data and is listed on a "couldn't enrich" view in the dashboard.

---

## 7. Testing and quality

- **Fixture tests:** save 3–5 real payloads per adapter (including your own applications: Figma, Mastercard, Notion, DV Trading, Freddie Mac, …) in `tests/fixtures/` and assert the extracted fields. The tests run offline.
- **Pay-parser tests:** 40+ real pay sentences collected during backfill, with expected min/max/interval.
- **Golden set:** 50 hand-checked jobs across ATS types. Report per-field accuracy (pay, location, work model, deadline) after each extractor change.
- **Run health checks:** alert (Telegram, later) if a run fails, if errors exceed 10%, or if no new listings appear for 48h (the repo or the parser may have changed).

---

## 8. Build steps

| Step | Work | Done when |
|---|---|---|
| 1 | Scaffold: `uv` project, SQLAlchemy models + Alembic migration for §3 tables, config (`categories`, `terms`, concurrency) | `alembic upgrade head` creates the DB |
| 2 | Sync + diff + filter (no enrichment) | One command loads all 17,211 listings and reports new/changed/closed correctly on a second run |
| 3 | Shared HTTP client (rate limits, retries, circuit breaker, snapshots) + `RawPosting` + extraction pipeline + pay parser with tests | Pay parser passes its tests |
| 4 | Greenhouse + Workday + Ashby adapters with fixture tests | ~2,190 relevant jobs enrich successfully |
| 5 | Generic JSON-LD adapter + Oracle + Lever + SmartRecruiters | ≥ 90% of relevant jobs enriched |
| 6 | Re-check scheduling, closure detection, `job_changes`, `ingest_runs` | Re-running shows closures and pay edits |
| 7 | launchd agent + lock + logs; run the backfill | Hourly runs complete in < 5 min after backfill |
| 8 | Playwright fallback + "couldn't enrich" list | ≥ 95% enriched |
| 9 | Hook into dashboard v1: Discover list shows pay, location, work model, deadline and closures | Visible in the dashboard |

---

## 9. Open questions

1. **Terms:** enrich only **Summer 2027**, or also Fall 2026 / Winter–Spring 2027 co-ops? (This decides the backfill size.)
2. **Your past 81 applications:** also re-fetch their pages hourly so you see if a posting you applied to closes? (Suggest yes; it's cheap and useful for status tracking.)
3. **Pay display:** normalize everything to **hourly** in the dashboard (suggested for internships), or show the posted unit?
4. **Notifications:** should each hourly run send a Telegram digest of new high-relevance jobs once the bot exists, or only a daily summary?
