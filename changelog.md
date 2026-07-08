# Changelog

## Session 1 — 2026-07-08

### State on arrival
- Empty repo: only SPEC.md and .claude/CLAUDE.md existed, nothing committed, no branch, no code

### What was done
- Reviewed SPEC.md with the user and confirmed scope (feed, filters, detail modal with link to original, 3 active + 9 dormant sources)
- Confirmed DB models (Source, NewsItem) are good as specced; flagged one open question on nullable `raw_content` (see below)
- Discussed Phase 2 scraping risks (truncated Substack feeds, paywalls, PDFs from Papers with Code, rate limiting) — no action needed for Phase 1
- Created conda env `ai-research-tracker` (Python 3.12.13)
- Scaffolded frontend with `npm create vite@latest frontend -- --template react` (React, Vite; deps installed)
- Created backend skeleton: `backend/app/{api,models,schemas,scrapers,services}` with empty `__init__.py` files, empty `app/main.py`, empty `backend/.env.example`
- Added root CLAUDE.md, README.md, .gitignore, empty changelog.md
- Created branch `feature-sheen`, made initial commit (ff7e7b6), pushed to GitHub with upstream tracking
- Wrote `IMPLEMENTATION_PLAN.md` (repo root): full backend implementation plan — 14-step build order, module breakdown, Alembic + seed commands, data-flow diagram, function signatures, pinned requirements.txt, testing checkpoints, 23 gotchas

### What changed
- No dependencies in requirements.txt yet — the pinned list lives in IMPLEMENTATION_PLAN.md §6, to be created next session
- Frontend: package.json/package-lock.json created by Vite scaffold (React 18 template defaults, nothing added manually)
- No model fields, endpoints, or env vars implemented yet (all planned only)
- New files: IMPLEMENTATION_PLAN.md (uncommitted), this changelog entry (uncommitted)

### Watch out for
- `IMPLEMENTATION_PLAN.md` is NOT yet committed to `feature-sheen` — commit it before or at the start of next session
- `backend/app/main.py` and `backend/.env.example` are committed but EMPTY (1 blank line each) — plan §6 has the .env.example contents
- Root `CLAUDE.md` adds one rule beyond `.claude/CLAUDE.md`: all requirements.txt versions must be exactly pinned
- Supabase pooled port (6543/pgbouncer) breaks asyncpg prepared statements — use direct port 5432 or `statement_cache_size=0` (plan gotcha #6)
- Open design decision: items with empty `raw_content` will be logged and skipped, not summarised from title alone (plan gotcha #16) — revisit if too many items get dropped
- First-ever ArXiv scrape can return hundreds of entries → plan caps ~50 new items per source per cycle to protect the Claude bill (gotcha #18)
- User's earlier manual `git push` failed because the branch had no commits — resolved; branch now tracks origin

### Next session
1. Read IMPLEMENTATION_PLAN.md and follow it top to bottom
2. Commit IMPLEMENTATION_PLAN.md + changelog.md to `feature-sheen`
3. Create `backend/requirements.txt` (plan §6) and `pip install -r requirements.txt` in the `ai-research-tracker` conda env
4. Fill `backend/.env.example`; create real `.env` with Supabase DATABASE_URL + ANTHROPIC_API_KEY
5. Build steps 1–5 of the plan: config.py → database.py → models → Alembic init/migrate → seed_sources.py (12 sources)
6. Verify each step with its checkpoint from plan §7 before moving on
7. Continue with schemas → tagger → summariser → scrapers → scrape_runner → API routes → scheduler → main.py