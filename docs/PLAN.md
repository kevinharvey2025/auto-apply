# auto-apply — Project Plan

A local web app that finds internships from the SimplifyJobs list, picks or builds the best resume for each one, fills out and submits the application on the company's own site, texts (Telegram, for now) for anything it doesn't know, and tracks every application with metrics.

_Last updated: 2026-10-08 (v4: Google Sheet tracker with 81 applications as the history source; custom dashboard replaces the spreadsheet)_

---

## 1. Decisions so far

| Topic | Decision |
|---|---|
| Submission mode | **Fully automatic** for now. Build the submit step behind a `mode` switch so `review-queue` and `fill-only` can be added later without rewrites. |
| Asking for unknown info | **Telegram bot first**, behind a `Notifier` interface so SMS (Twilio) or iMessage can be plugged in later. |
| Job categories | **Software, AI/ML/Data, Quant, Product** (all four). |
| Where to apply | **Directly on the company's careers site / ATS**, never through Simplify. |
| Hosting | **Local on the Mac first**, cloud later. |
| Jobs vs internships | Treated as the same thing for now. |
| Resume template | **Jake's Resume** (already what the master resume and all tailored resumes use). |
| Tracker | **A custom dashboard replaces the spreadsheet.** The Google Sheet "Internship Application Tracker" (81 applications) is imported once. After that the app's database is the source of truth (§2.11). |
| Classification / extraction | **Locally trained ML models, not Claude** (see §2.9). Claude is used only for generating text: rewriting bullets and drafting essays. |

---

## 2. Research findings

### 2.1 The SimplifyJobs data source
- The README table is generated from a machine-readable file: **`.github/scripts/listings.json`**. We read that JSON, never scrape the README.
- Snapshot on 2026-10-08: **17,200 listings total, 4,622 active & visible**.
- Fields per listing: `id`, `company_name`, `title`, `category`, `url`, `locations`, `terms` (e.g. "Summer 2027"), `sponsorship`, `degrees`, `active`, `is_visible`, `date_posted`, `date_updated`, `company_url`, `source`.
- Active listings by category: AI/ML/Data 1,641 · Software 1,506 · Hardware 970 · Product 254 · Quant 213 (plus a few dozen with alternate category spellings like "Software Engineering" that need normalizing).
- Active listings by term: Summer 2027 2,133 · Winter 2026 626 · Summer 2026 515 · N/A 443 · Fall 2026 442 · Winter 2027 400. **We need a term filter** (see open questions).
- **Good news for "apply on the company site":** the `url` field already points at the employer's own ATS, not at Simplify (0 simplify.jobs links among active listings; a handful carry `utm_source=Simplify` params, which we strip).
- `sponsorship` is "Other" for almost everything, so it isn't a useful filter.
- Sync approach: `git pull` (or fetch the raw JSON) every few hours, upsert by `id`, mark jobs closed when `active` flips to false.

### 2.2 Which application systems we'll actually be filling out
Active listings by the ATS behind their URL:

| ATS | Active jobs | Difficulty | Notes |
|---|---|---|---|
| Workday (`myworkdayjobs.com`) | 1,696 (37%) | Hard | Needs an account **per company tenant** + email verification, multi-page wizard. |
| Custom / other career sites | 968 | Hard | No common structure; needs a generic LLM-driven form filler. Some embed Greenhouse (e.g. `?gh_jid=` on company domains) and can be redirected to the Greenhouse board. |
| Greenhouse | 517 | Easy | Single-page form; question schema is public. |
| iCIMS | 387 | Medium-hard | Often iframes + account creation. |
| Oracle Cloud (Taleo successor) | 326 | Hard | Multi-step, account/email verification. |
| Ashby | 291 | Easy | Single-page form; public posting API. |
| SuccessFactors | 164 | Hard | Account required. |
| Lever | 111 | Easy | Single-page form; public postings API. |
| SmartRecruiters | 100 | Medium | |
| Workable / Taleo / Jobvite | ~60 | Medium | |

**Implication:** Greenhouse + Ashby + Lever (~920 jobs) are the fastest path to a working end-to-end system. Workday is the biggest prize but the most work, so it comes second.

### 2.3 How submission has to work
- Greenhouse, Lever and Ashby all have **public read APIs** for job descriptions (and Greenhouse exposes each job's application questions via `?questions=true`), which is great for reading the JD and pre-planning answers.
- Their **write/apply APIs require the employer's private API key**, so we can't submit through an API. Submission has to be **browser automation (Playwright)**.
- Plan: one **adapter per ATS** (deterministic selectors for the common fields + resume upload), with an **LLM fallback** that reads the page's form fields (labels, types, options) and maps each one to a profile fact. The generic LLM filler is also how custom career sites get handled.
- **Workday / Oracle / SuccessFactors / iCIMS need accounts.** That means storing per-tenant credentials (macOS Keychain) and reading verification emails. That needs Gmail access (Gmail API or an IMAP app password) — a later stage.
- **CAPTCHAs:** we do not use CAPTCHA-solving services. When a page presents a CAPTCHA or a check we can't pass, the app pauses that application and sends you a Telegram message with the link so you can finish it by hand. The tracker records it as `needs_manual`.
- **Proof of submission:** after clicking submit, detect the confirmation page or message, save a full-page screenshot and the filled values, and only then mark the job `applied`.

### 2.4 Risks with fully automatic mode (and the guardrails we'll add)
Automatic mode is the right default for what you asked for, but a bad submission can't be undone, and some companies cap how many roles a candidate can apply to. So automatic still has guardrails:
1. **Dry-run first.** Each new adapter runs in fill-but-don't-submit mode on ~10 real jobs, saving screenshots, before it's allowed to submit.
2. **Submit only when the confidence gate passes:** every required field is answered from a known fact (or an approved generated answer), the resume score is above a threshold, and the job passes the relevance filter. If any check fails, the application waits on you instead of guessing.
3. **No fabrication.** A skill or experience is never claimed unless it's in the master resume or you confirmed it over Telegram.
4. **Dedupe** against everything already in the tracker (including your existing spreadsheet), by company + normalized title + URL.
5. **Rate limits:** a daily cap (e.g. 25/day to start) and per-company spacing.
6. **Kill switch:** `/pause` and `/resume` commands in Telegram.
7. Some sites' terms of service forbid automated submissions. Keep volumes human-like and accept that some sites may block the bot.

### 2.5 "ATS scoring" for resumes
There is no universal, official "ATS score." Most ATSs parse the resume into fields, and recruiters then search and filter by keywords. Some ATSs add AI-based ranking. Tools like Jobscan approximate this with keyword matching. We'll build our own score from four parts:

| Component | What it measures | How |
|---|---|---|
| **Parseability** (gate) | Does the PDF extract cleanly? | Compile → `pdftotext` → check that name, sections, dates and bullets come out in order. Jake's template passes this well. |
| **Keyword coverage** (~60%) | Required/preferred skills from the JD found in the resume | The trained skill extractor (§2.9, model B) pulls skills from the JD, and a requirements-sentence classifier tags them required vs. preferred. Normalize synonyms (e.g. "PyTorch" ≈ "Torch", "ML" ≈ "machine learning"). Weighted hit rate; required skills count more. |
| **Semantic fit** (~40%) | Overall relevance beyond exact keywords | Embedding cosine similarity between the JD and each resume section/bullet. |
| **Learned fit** (later) | Which resumes actually get callbacks | A learning-to-rank model trained on application outcomes, added once there's enough data (§2.9, model F). |

The result is a **0–100 score per (resume, job) pair**, stored with a breakdown and the missing keywords. Selection works like this:
- best existing resume ≥ **threshold (start at 75, tune later)** → use it
- otherwise → **generate a tailored resume** from the master resume (Stage 4) and score it too. The pipeline keeps whichever scores higher.

### 2.6 Resume generation (Jake's template)
- The **master resume** (`~/Documents/Resume/master resume/Resume.tex`) is already in Jake's template, with sections Education, Experience, Projects, Awards & Honors and Technical Skills, and 81 bullets.
- One-time step: **parse the master `.tex` into structured data** (`master.yaml`: entries → bullets, each with tags and skills). After that, the app edits the YAML, not raw LaTeX.
- Generation pipeline:
  1. From the JD keywords, rank entries and bullets by relevance.
  2. Claude picks entries and bullets, reorders them, and lightly rewrites bullets to use the JD's terminology. Rewrites **must stay factual** to the master bullet; a diff against the source is checked.
  3. **Skills section:** add JD skills that are supported by the master resume. For JD skills that aren't evidenced, send a Telegram question ("Do you have experience with Kafka? If yes, where?"). A "yes" is stored as a profile fact and can be used from then on. This is how "add skills from the listing when it makes sense" works without lying.
  4. Render with a **Jinja2 template of Jake's resume**, with LaTeX escaping (`&`, `%`, `#`, `_`).
  5. Compile with **Tectonic** (single binary, auto-fetches packages) or `latexmk`.
  6. **Enforce one page:** if the PDF is more than one page, drop the lowest-ranked bullet and recompile, in a loop.
  7. Save it to the resume library with its JD, score and source mapping.
- **Seed the library** with your ~15 existing tailored resumes (AI Engineer, SWE Intern, Quant Researcher, Product Manager, Research, etc.) so the selector has real options on day one.

### 2.7 Asking you for unknown information (Telegram now, SMS later)
- **Telegram Bot API** via `python-telegram-bot`. Long polling works on a laptop with no public URL. Setup is free and instant: create a bot with @BotFather and save the token.
- How an answer gets resolved for each form field:
  1. **Exact match** in `profile_facts` by canonical key (e.g. `work_authorization`, `graduation_date`, `gpa`, `linkedin_url`).
  2. **Semantic match** of the question text against past Q&A (embeddings). Example: "Are you legally authorized to work in the US?" ≈ a prior answer.
  3. **Open-ended / essay questions** ("Why do you want to work here?"): Claude drafts an answer from your profile and the company description. In automatic mode the draft is used and saved, and you get it in a digest. *(Configurable: require approval for essays.)*
  4. Otherwise, **ask on Telegram** with the question, the company and any options as tap-able buttons. The application moves to `waiting_on_user`. When you reply, the answer is saved to the DB (with the question text, so it matches next time) and the application resumes.
- **SMS later:** sending SMS from US long-code numbers via Twilio requires **A2P 10DLC registration** (there's a sole-proprietor brand option) or toll-free verification. That takes days and has small monthly fees, which is why Telegram goes first. An iMessage-from-the-Mac notifier is another free option.

### 2.8 Tracking and metrics
- **Source of history:** the Google Sheet **"Internship Application Tracker"** in your Drive (sheet *Applications Summer 2027*). The `.xlsx` copy on the Mac is out of date and is ignored. As of 2026-10-08 the sheet has **81 applications at 67 companies**: 44 SWE, 11 AI Engineer, 7 Data Scientist, 7 Quant, 5 PM and a handful of others. Of those, 13 are rejections (10 auto-rejections), 6 have an OA (received or completed), and 0 are offers.
- **Import (one-time, via the Google Drive connection / Sheets CSV export):** columns Position, Company, Industry, Role, Status, Date Applied, Location, Furthest Stage, Latest word, Date Posted, Cover Letter, Notes, Connections?, contact 1. Cleanup on the way in: trim whitespace, fix obvious company typos with an alias map (e.g. "Vangarud" → Vanguard, "Garmen" → Garmin), parse dates (and flag bad ones like `9/23/2926`), and map statuses: `sent→applied`, `OA sent/OA received→oa`, `OA Complete→oa_done`, `phone call→screen`, `interview→interview`, `rejection→rejected` (with `auto_rejected` when Furthest Stage says "Auto-rejection"), `offer→offer`, `draft→queued`.
- After the import, the sheet is retired. A one-click **CSV/Sheet export** stays available as a backup (see Future work).
- **Stages:** `discovered → filtered_out | queued → resume_ready → filling → waiting_on_user | needs_manual → applied → oa → interview → final_round → offer | rejected | withdrawn | ghosted`.
- **Metrics:** applications per day/week, funnel conversion by stage, response rate and time-to-first-response, and breakdowns by category, ATS, company tier, location and **resume version** (which resumes get callbacks), plus the automation success rate per ATS, questions asked vs. auto-answered, and how many jobs are open vs. applied.
- Status updates after applying (rejections, OA invites, interviews) start as manual edits in the dashboard and become **automatic from Gmail parsing** later.

---

### 2.9 Trained models instead of Claude for classification
Every classify-or-extract step uses a small model trained locally on the Mac. Claude is kept only for generating text (bullet rewrites, essay drafts). Each model outputs a **calibrated probability**. When it's below threshold, the app falls back to asking you, never to guessing.

| # | Task | Model progression (baseline → better) | Training data | When unsure |
|---|---|---|---|---|
| A | **Job relevance**: should I apply? | **Phase 1 (no labeling):** embedding similarity to jobs you already applied to (see §2.10) → **Phase 2:** positive-unlabeled classifier on embeddings + structured features (LightGBM) → **Phase 3:** SetFit once real yes/no feedback piles up | **Your 81 past applications** (Google Sheet tracker) as positives, plus your master resume. No hand-labeling up front. Later: Telegram skips/overrides and application outcomes. | Hold the job and send a one-tap Telegram question. |
| B | **Skill extraction** from JDs | Dictionary matcher (spaCy PhraseMatcher over an O*NET/ESCO + tech-alias list) → pretrained **JobBERT skill extractor** (token classification, trained on SkillSpan) → fine-tuned on a few hundred JD sentences you label | Public SkillSpan dataset, then your own corrections | Use the dictionary results only. |
| C | **Form question → profile field** (e.g. "Are you authorized to work in the US?" → `work_authorization`) | Embeddings + kNN/logistic regression over ~60 canonical keys + `unknown` → SetFit | Harvest thousands of real questions from Greenhouse's public question API, cluster them, and **label each cluster once** (one label covers many questions). New Telegram answers add examples automatically. | Mark `unknown`, then ask on Telegram. |
| D | **Role/category normalization** (Software / AI-ML / Quant / Product / other) | TF-IDF + logistic regression on the title | **Free labels:** Simplify's 17k listings already carry a category. | — |
| E | **Recruiter email → stage** (future) | Same recipe as A | Your labeled inbox | Leave as manual. |
| F | **Resume fit / callback prediction** (future) | Learning-to-rank (LightGBM LambdaRank) | Application outcomes. This needs a few hundred applications with results, realistically after one cycle. Until then, the heuristic score in §2.5 is used. | Heuristic score |

**Relevance features (model A):** title text, JD embedding, normalized category, term, location vs. your preferences, company (and whether you've applied there before), ATS, degree requirement, and skill overlap with the master resume.

**How the models are trained and checked:**
- **Cold start:** model A starts from embedding similarity to your past applications (§2.10), so **no labeling session is needed**. The labeling page still exists for corrections and for models B and C.
- **Optimize for precision, not accuracy.** A false positive is a wasted, often one-shot application at that company. Pick the auto-submit threshold for **precision ≥ 0.95** on a held-out set, and calibrate probabilities (isotonic or Platt scaling) so the confidence gate means something.
- **Retrain nightly** on new labels. Version every model, keep the held-out set fixed, and show precision/recall, label counts and drift on a "Models" page.
- Everything trains on the Mac's CPU/MPS in minutes. SetFit on MiniLM-sized models is small enough.
- **Tooling:** scikit-learn, sentence-transformers, `setfit`, LightGBM, spaCy, Hugging Face `transformers` (for JobBERT); MLflow or plain JSON run logs for tracking experiments.

### 2.10 Relevance from similarity to past applications (no labeling)
The job-relevance model starts **without any hand-labeling**. It compares each new job's embedding with the embeddings of jobs you already applied to.

**Anchors:** your **81 past applications** from the Google Sheet tracker (44 SWE, 11 AI Engineer, 7 Data Scientist, 7 Quant, 5 PM, plus a few aerospace/automation/security roles). Their titles and descriptions are recovered by matching company + title against Simplify's 17k listings, which include closed ones. **Your master resume** is added as one extra anchor.

**What gets embedded:** the job title and the *responsibilities/qualifications* part of the description. Boilerplate (EEO statements, benefits, company "about us") is stripped first, because otherwise every description looks alike and similarities bunch up around the same value. Title and body are embedded separately and combined: `0.4·sim(title) + 0.6·sim(body)`. The embedding model is local: `bge-small-en-v1.5` or `all-MiniLM-L6-v2` via sentence-transformers.

**Scoring rule: nearest neighbors, not the average.** Your interests are spread across several clusters (SWE, PM, AI/ML, quant). The average (centroid) of all 81 sits *between* those clusters and matches none of them well. Instead:
- `score(job) = mean of the top-k cosine similarities to the anchors` (k = 3). A job only has to look like *one group* of your past applications.
- Alternative to compare: cluster the anchors (k-means, ~4 clusters) and take the max similarity to any cluster centroid.
- Each score comes with an **explanation**: "closest to: Databricks PM Intern (0.82), Atlassian PM Intern (0.79)". This goes in the dashboard and Telegram messages.

**Choosing the threshold without labeled negatives:**
- **Leave-one-out recall:** score each anchor against the other 80. The threshold where ~90% of anchors would still pass estimates how many good jobs get through.
- **Precision spot-check:** sample ~30 jobs above the threshold and eyeball them. This is the only manual step, and it takes about 5 minutes.
- **Easy negatives for sanity checks:** Simplify's Hardware category and non-tech titles should score clearly lower than your anchors. If they don't, the boilerplate stripping isn't working.

**Avoid a feedback loop:** only applications **you chose** are anchors. Jobs the bot auto-applied to are **not** added as anchors. Otherwise it would drift toward ever-narrower copies of itself. A job enters the anchor set only after a positive signal: you approved it, or it reached OA/interview.

**Anchor weights:** the 6 applications that reached an OA get extra weight (×1.5) because they're the best evidence of fit. Rejected applications still count as anchors, because relevance ("would I apply?") is a different question from outcome ("did they say yes?"). Outcomes feed model F later.

**Upgrade path:** as Telegram skips ("don't apply to jobs like this") and outcomes accumulate, those become real labels. Model A then moves to a positive-unlabeled (PU) classifier (anchors = positives, everything else = unlabeled), and later to SetFit. The similarity score stays as one of its input features.

### 2.11 Custom dashboard (replaces the spreadsheet)
The web app is where you track everything. It carries over the metrics your sheet's *Dashboard* tab already computes (total applications, companies, OA recorded, auto-rejections, all rejections, offers, role breakdown with share/rejections/OAs, and status share), then goes well beyond them.

| Page | What's on it |
|---|---|
| **Overview** | KPI cards (applications, companies, OA rate, interview rate, rejections split into auto vs. after-OA, offers, applied this week vs. last, questions waiting on you). A **funnel** (applied → OA → interview → offer), applications over time, response rate and median days-to-first-response, and breakdowns by role, industry, category, ATS and location. |
| **Applications** | Table view (search, filter, sort, inline stage edit, bulk actions) **and** a Kanban board by stage with drag-and-drop. Clicking an application opens a drawer with the JD, the resume PDF that was sent, its ATS score, every answer that was submitted, the confirmation screenshot, a stage timeline, notes, contacts/connections, and links. |
| **Discover / Queue** | New Simplify jobs with their relevance score and "closest to" explanation, the resume picked for each, and status (queued, waiting on you, needs manual). Approve, skip ("don't show jobs like this") or apply now. |
| **Resumes** | The library (existing + generated), a PDF preview, a diff vs. the master resume, average ATS score, and **callback rate per resume version**. |
| **Profile & Questions** | Stored answers (editable), the questions waiting on Telegram, and a history of which questions came from where. |
| **Models** | Relevance threshold, leave-one-out recall, spot-check precision, label counts and model versions (§2.9–2.10). |
| **Bot control** | Mode (auto / paused, later review-queue / fill-only), daily cap, a live run log, failures, and per-ATS success rate. |

**Design notes:** Next.js + Tailwind + shadcn/ui components, Recharts for charts, and TanStack Table for the grid. It supports dark mode and is responsive, so it works from your phone over the local network now (and anywhere once it's deployed to the cloud). Edits in the dashboard write straight to the database, which is the single source of truth.

---

## 3. Architecture

```
                ┌────────────────────┐
  Simplify  ──► │ Ingest (cron, 3h)  │──► jobs table
  listings.json └────────────────────┘
                          │
                          ▼
                ┌────────────────────┐   JD fetch (ATS public APIs / page text)
                │ Match & Score      │──► relevance filter + resume scores
                └────────────────────┘
                          │ no resume ≥ threshold
                          ▼
                ┌────────────────────┐
                │ Resume Generator   │──► master.yaml → Jinja → Tectonic → PDF
                └────────────────────┘
                          │
                          ▼
                ┌────────────────────┐   unknown field   ┌──────────────┐
                │ Apply Worker       │ ────────────────► │ Notifier     │◄─► you (Telegram)
                │ Playwright +       │ ◄──────────────── │ (Telegram/   │
                │ ATS adapters + LLM │   answer stored   │  SMS later)  │
                └────────────────────┘                   └──────────────┘
                          │ screenshot + confirmation
                          ▼
                ┌────────────────────┐
                │ SQLite DB          │◄── Dashboard (Next.js): tracker, metrics, resumes, Q&A
                └────────────────────┘
```

**Stack (local-first)**
- **Backend:** Python 3.12, FastAPI, SQLAlchemy + Alembic, **SQLite** (one file, trivial on a Mac; swap to Postgres for cloud).
- **Jobs/scheduling:** APScheduler inside the backend (or `launchd`). There's a single apply worker so the browser isn't shared.
- **Browser automation:** Playwright (Chromium) with a persistent profile dir.
- **ML (classification/extraction):** scikit-learn, sentence-transformers, SetFit, LightGBM, spaCy, JobBERT. Training and inference are local (§2.9).
- **LLM (generation only):** Claude API for bullet rewriting and essay drafts.
- **Embeddings:** a local `sentence-transformers` model (free, fast enough), stored as numpy blobs or with `sqlite-vec`.
- **Resume build:** Jinja2 + Tectonic, plus `pdftotext` (poppler) for the parseability check.
- **Notifier:** `python-telegram-bot`.
- **Frontend:** custom dashboard (§2.11). Next.js + Tailwind + shadcn/ui, Recharts, TanStack Table and dnd-kit (Kanban), talking to the FastAPI backend.
- **Secrets:** `.env` for API keys; macOS Keychain (`keyring`) for ATS account passwords.

**Project layout** (current; `backend/` and `frontend/` arrive in Stage 0)
```
auto-apply/
  README.md
  .gitignore
  docs/
    PLAN.md
    INGESTION_PLAN.md
  backend/
    app/
      main.py            # FastAPI
      db/                # models, migrations
      ingest/            # simplify.py (+ future sources), adapters/, extract/
      matching/          # relevance.py, ats_score.py, keywords.py
      ml/                # datasets.py, train_*.py, calibrate.py, registry (versioned models), active_learning.py
      labeling/          # API for the dashboard labeling page
      resumes/           # master_parser.py, generator.py, templates/jake.tex.j2
      apply/
        engine.py        # orchestration, confidence gate, mode switch
        adapters/        # greenhouse.py, lever.py, ashby.py, workday.py, generic_llm.py
      answers/           # profile facts, resolver, essay drafting
      notify/            # base.py, telegram.py, (sms.py later)
      metrics/
  frontend/              # Next.js dashboard
  data/                  # local only, git-ignored
    seed/                # one-time imports (tracker_job_links.csv, master.yaml)
    db/                  # app.db (SQLite)
    snapshots/           # raw job-page payloads from ingestion
    resumes/generated/   # tailored .tex/.pdf
    screenshots/         # application confirmations
    models/              # trained model artifacts + eval reports
    logs/                # ingest/worker logs
```

**Core tables:** `labels` (item, task, label, source, created_at), `model_versions` (task, metrics, path), `jobs`, `job_descriptions`, `resumes`, `resume_scores`, `applications`, `application_events` (stage history), `profile_facts` (key, value, question_text, embedding, source, confirmed_at), `pending_questions`, `ats_accounts`, `artifacts` (screenshots, PDFs).

---

## 4. Stages

### Stage 0 — Scaffold (½ day)
- Repo, Python env (`uv`), Next.js app, `.env.example`, SQLite + Alembic, Playwright and Tectonic installed.
- Config file: categories, term filter, locations, daily cap, score threshold, submit mode.
- **Done when:** `make dev` starts the backend and frontend, and the DB is created.

### Stage 1 — Ingest + tracker (1–2 days)
- Simplify sync: fetch `listings.json`, normalize categories, strip UTM params, classify the ATS from the URL, upsert, close inactive jobs.
- Import the **Google Sheet tracker** (81 applications), with the cleanup and status mapping from §2.8.
- **Dashboard v1:** the Applications page (table + Kanban + detail drawer) and the Overview KPI cards matching your sheet's Dashboard tab, plus the Discover list of incoming jobs.
- **Done when:** all 81 applications appear in the dashboard with correct stages, the Overview numbers match the sheet (81 / 67 companies / 6 OA / 13 rejections), and filtered Summer 2027 jobs show in Discover.

### Stage 2 — Profile + master resume (1–2 days)
- Parse the master `.tex` → `master.yaml` (entries, bullets, skills, tags). Hand-check it once.
- Seed `profile_facts` with the common answers (contact, school, grad date, GPA, links, work authorization, relocation, start/end availability, EEO preferences, how you heard about us).
- Profile page in the dashboard to view and edit facts.
- **Done when:** every field on a typical Greenhouse form can be answered from the DB.

### Stage 3 — Relevance + resume scoring (4–6 days)
- JD fetch: Greenhouse, Lever and Ashby public APIs; page-text extraction for everything else.
- **ML foundation:** `labels` table, labeling page, train/eval/calibrate scripts, model registry.
- **Model D** (category normalization, trained on Simplify's labels). This is the first end-to-end run of the training pipeline.
- **Model A** (relevance): rules first (category, term, exclusions), then the embedding-similarity scorer from §2.10 anchored on your 81 past applications. Run the leave-one-out recall check, then spot-check precision on a random sample of above-threshold jobs.
- **Model B** (skill extraction): dictionary matcher, then JobBERT, compared on ~50 hand-labeled JDs.
- ATS score (parseability, keywords, semantic, LLM rubric) for every resume in the library × each queued job.
- **Done when:** each queued job shows a relevance score with its nearest past application as the explanation, plus its best resume, the score breakdown and missing keywords, and the spot-check shows ≥0.95 precision above the auto-submit threshold.

### Stage 4 — Resume generator (2–3 days)
- Jinja version of Jake's template, the selection and rewrite pipeline, the one-page enforcement loop, and the factuality diff check.
- The skill-confirmation questions route through the notifier (stubbed to the console until Stage 5).
- **Done when:** a job with no good existing match gets a new one-page PDF that scores higher than any library resume.

### Stage 5 — Telegram loop (1–2 days)
- Bot setup, the `Notifier` interface, the question → answer → store → resume flow, inline buttons for multiple-choice questions, and `/pause`, `/resume`, `/status` and a daily digest.
- **Done when:** an unknown question reaches your phone and your reply unblocks the application.

### Stage 6 — Auto-apply v1: Greenhouse, Lever, Ashby (4–6 days)
- Playwright adapters for the three ATSs, resume upload, the answer resolver, the confidence gate, confirmation detection and screenshots.
- **Model C** (form question → profile field): harvest Greenhouse questions, cluster and label them, train, and plug it into the answer resolver.
- Run in **dry-run on ~10 jobs per ATS**, review the screenshots, then flip to submit.
- **Done when:** ~920 jobs are eligible for fully automatic submission with evidence saved.

### Stage 7 — Auto-apply v2: Workday + the rest (1–2 weeks)
- Gmail integration for verification codes and links.
- Workday account creation per tenant (credentials in Keychain) and the multi-page wizard adapter.
- Generic LLM form-filler for custom sites, then iCIMS, Oracle, SmartRecruiters and SuccessFactors as they come up.
- **Done when:** most active jobs are reachable automatically and the rest are routed to `needs_manual`.

### Stage 8 — Full analytics dashboard (3–4 days)
- Overview funnel, time-series, response-time stats and breakdowns, plus the Resumes, Models and Bot control pages from §2.11.
- **Done when:** you can see what's working (by role, industry, ATS and resume version) and control the bot entirely from the dashboard.

---

## 5. Next steps (in order)
1. **Answer the open questions below.**
2. Create a Telegram bot with @BotFather and save the token. Get a Claude API key.
3. Install the toolchain on the Mac: `brew install uv tectonic poppler node`, then `uv run playwright install chromium`.
4. Build **Stage 0 + Stage 1** (scaffold, Simplify sync, Google Sheet import, Dashboard v1).
5. Write the master-resume parser and review `master.yaml` together.
6. Backfill job descriptions for your 81 past applications (match them by company + title against Simplify's closed listings) so they can serve as similarity anchors.

## 6. Open questions
1. **Which terms?** Only **Summer 2027**, or also Fall/Winter 2026–27 co-ops and off-season roles?
2. **What counts as "aligned with your goals"?** Any title keywords to prefer or exclude (e.g. exclude "Hardware", "Sales Engineer"), companies to prioritize or never apply to, and location preferences or remote-only?
3. **Daily cap** for automatic mode to start with (suggest 25)?
4. **Essay questions:** auto-generate and submit, or require your approval for essays even in automatic mode?
5. **EEO / demographic questions:** always "decline to answer," or should the app store your answers?
6. **Gmail access** (needed for Workday verification and automatic status updates): OK to connect your Gmail via the API? Which address do applications go from? (The resume lists a different Gmail from your account email.)
7. **Cover letters:** you have a LaTeX cover letter template. Generate cover letters when a form has an optional upload, or only when one is required?
8. **Anchor jobs:** are all 81 tracker applications ones you'd want more of? Drop any you applied to reluctantly (e.g. Texas Instruments' manufacturing role, the Collier aerospace automation role) so they don't attract similar jobs.

---

## 7. Future work
- **Submission modes:** `review-queue` (approve in Telegram or the dashboard before submit) and `fill-only` (opens a filled browser for you to submit), selectable globally or per company.
- **SMS** via Twilio after 10DLC/toll-free registration, or iMessage from the Mac.
- **More job sources:** SimplifyJobs New-Grad repo and off-season README, other GitHub lists, company career pages directly (Greenhouse/Lever/Ashby boards by company), and job-board APIs.
- **Email-driven status tracking:** classify incoming recruiter emails (rejection, OA, interview, offer) and update stages automatically.
- **Cover letter generation** from the LaTeX template.
- **Referral / connections helper:** flag jobs where you have a contact and draft outreach.
- **Interview prep:** when a job moves to `interview`, generate a prep sheet from the JD and your resume.
- **Learning loop:** model F (learning-to-rank on callbacks) replaces the hand-set weights in the resume score.
- **Fine-tune a small local generator** (e.g. a LoRA on a small open model) for bullet rewriting, to drop Claude entirely.
- **Sheet export / sync:** optional one-way export of the applications table back to Google Sheets for sharing or backup.
- **Cloud deploy:** a worker on a small VM, Postgres, and the dashboard on Vercel, so it runs with the laptop closed.
- **Multi-user** support and auth, if friends want to use it.

---

## Sources
- SimplifyJobs Summer 2027 Internships — https://github.com/SimplifyJobs/Summer2027-Internships (`.github/scripts/listings.json`, analyzed 2026-10-08)
- Twilio A2P 10DLC overview — https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Twilio: collecting business info for 10DLC (incl. sole proprietor) — https://www.twilio.com/docs/messaging/compliance/a2p-10dlc/collect-business-info
- Greenhouse API overview — https://getknit.dev/blog/greenhouse-api
- SkillSpan: Hard and Soft Skill Extraction from English Job Postings (NAACL 2022) — https://arxiv.org/abs/2204.12811
- JobBERT skill extraction model — https://huggingface.co/jjzha/jobbert_skill_extraction
- SetFit (few-shot fine-tuning of sentence transformers) — https://github.com/huggingface/setfit
- Jake's Resume template (based on sb2nov/resume) — https://github.com/sb2nov/resume
