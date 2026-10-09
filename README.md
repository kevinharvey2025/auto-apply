# auto-apply

Local app that finds internships from the SimplifyJobs list, picks or builds the best resume for each one, applies on the company's own site, asks over Telegram for anything it doesn't know, and tracks every application in a custom dashboard.

## Docs
- [docs/PLAN.md](docs/PLAN.md): overall plan, stages, open questions
- [docs/INGESTION_PLAN.md](docs/INGESTION_PLAN.md): hourly job ingestion from SimplifyJobs

## Layout
```
auto-apply/
  docs/                 plans and design notes (tracked)
  backend/              FastAPI app, ingestion, ML, apply engine (coming in Stage 0)
  frontend/             Next.js dashboard (coming in Stage 0)
  data/                 local only, never committed (see .gitignore)
    seed/               one-time imports (e.g. tracker_job_links.csv)
    db/                 SQLite database (app.db)
    snapshots/          raw job-page payloads from ingestion
    resumes/generated/  tailored resumes (.tex/.pdf)
    screenshots/        application confirmation screenshots
    models/             trained model artifacts and eval reports
    logs/               ingest and worker logs
```
