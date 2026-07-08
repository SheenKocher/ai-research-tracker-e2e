# AI Research Tracker — CLAUDE.md

## What this project is
FastAPI + React app that scrapes AI research sources on a schedule, summarises each item using Claude API (80–120 words), and displays them in a dark-mode filterable feed. Single user. No auth.

**Stack:** Python 3.12, FastAPI, SQLAlchemy 2.x async, Alembic, httpx, APScheduler, tenacity, feedparser, React 18 + Vite, shadcn/ui, Tailwind CSS, claude-sonnet-4-6
**DB:** PostgreSQL via Supabase (AsyncSession only)
**Deploy:** Railway (backend) + Vercel (frontend)

---

## Running locally
```bash
# Backend
cd backend && uvicorn app.main:app --reload --port 8000

# Frontend
cd frontend && npm run dev
```

---

## Env vars (never hardcode)
```
DATABASE_URL=postgresql+asyncpg://...
ANTHROPIC_API_KEY=sk-ant-...
CORS_ORIGINS=http://localhost:5173
```

---

## Coding conventions — follow these exactly

- All endpoints `async def` — never sync
- Always pin exact versions in requirements.txt (e.g. `fastapi==0.115.0` not `fastapi`) — never leave dependencies unpinned
- `httpx.AsyncClient` for all outbound HTTP — never `requests`
- Pydantic v2 only — use `model_dump()`, never `.dict()`
- All DB ops use `AsyncSession` via `get_db()` dependency
- New scrapers go in `/backend/app/scrapers/` extending `BaseScraper`
- All errors return `HTTPException` with correct status codes
- Never hardcode API keys, URLs, or credentials

---

## Scraper rules — critical

**Deduplication before Claude call:**
```python
url_hash = hashlib.sha256(url.strip().rstrip("/").split("?")[0].encode()).hexdigest()
# Check DB first. If hash exists → skip. Never re-summarise.
```

**Always use `return_exceptions=True`:**
```python
results = await asyncio.gather(*tasks, return_exceptions=True)
# One source failing must never abort the whole cycle
```

**Always use tenacity for retries:**
```python
@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=2, max=8))
async def fetch_with_retry(client, url): ...
# httpx timeout: 15s on every request
```

---

## Summarisation prompt
```
You are a technical summariser for an AI research feed read by engineers.
Summarise in 80–120 words across three parts:
1. What was built or discovered (name the model, method, or result specifically)
2. Why it matters for AI engineering
3. One concrete thing an engineer could do with this today
Rules: No filler phrases ("this paper explores", "researchers have found").
Start with the finding. Plain text only — no bullets, no headers, no markdown.
Content: {raw_content}
```

---

## Valid tags
```python
VALID_TAGS = [
    "research-paper",   # ArXiv, Papers with Code
    "model-releases",   # New models from any lab
    "platform-updates", # API/SDK/tooling changes
    "ai-engineering",   # How to build with AI
    "agents",           # Agent frameworks, MCP, orchestration
    "open-source",      # Open weight models, repos
]
```

---

## Active sources (Phase 1)
| Source | Type | Poll |
|---|---|---|
| HuggingFace Daily Papers | JSON API | 12h |
| ArXiv cs.AI | RSS | 6h |
| Anthropic News | RSS | 24h |

All other sources are seeded with `is_active=False`. Flip the flag to enable — no code change needed.

---

## Folder structure
```
/backend
  /app
    /api          # FastAPI routers
    /models       # SQLAlchemy ORM models
    /schemas      # Pydantic v2 schemas
    /scrapers     # one file per source, extends BaseScraper
    /services     # summariser.py, scheduler.py
  main.py
  alembic/
/frontend
  /src
    /components
    /pages
    /api          # fetch wrappers for backend calls
```

---

## What NOT to do
- Do not add user auth unless explicitly asked
- Do not use sync SQLAlchemy — always `AsyncSession`
- Do not use `requests` — always `httpx.AsyncClient`
- Do not call Claude API if `url_hash` already exists in DB
- Do not use Pydantic v1 methods (`.dict()`, `.schema()`)
- Do not modify `alembic/versions/` manually
- Do not add `console.log` to frontend
- Do not implement search, light mode, bookmarks, email digests, or mobile layouts
- Do not add `consecutive_failures` tracking to Source model

---

## API endpoints (quick ref)
| Method | Path | Description |
|---|---|---|
| GET | `/news` | Feed with cursor pagination + filters |
| GET | `/news/{id}` | Single item |
| POST | `/scrape/trigger` | Manual scrape (for testing) |
| GET | `/sources` | All sources + status |

`GET /news` query params: `source_id`, `tags`, `from_date`, `to_date`, `cursor`, `limit` (default 20, max 50)

---

## Changelog — rules for every session

A `changelog.md` file lives in the repo root. It is the single source of truth for project state across sessions.

### At the START of every session:
1. Read `changelog.md` before doing anything else
2. Check if the last entry has a "What was done" section — if it's missing, the previous session wasn't closed properly. Complete it first based on the current state of the codebase, then proceed.
3. Use it to understand exactly where the project is before touching any code

### At the END of every session (when user says "update changelog"):
Append a new entry in this exact format:

```
## Session N — YYYY-MM-DD

### State on arrival
- What was already done when this session started

### What was done
- Bullet list of everything implemented or changed this session

### What changed
- Any dependency added to requirements.txt or package.json
- Any model field added, removed, or renamed
- Any endpoint added or modified
- Any env var added

### Watch out for
- Bugs discovered but not fixed
- Gotchas or non-obvious behaviour found
- Anything that will trip up the next session

### Next session
- Exact next tasks, in order
```

### Rules:
- Never delete old entries — append only
- Be specific ("added `feedparser` to requirements.txt") not vague ("updated dependencies")
- "Watch out for" is the most important section — don't skip it
- If a session was short or unproductive, still log it honestly

- [ ] SPEC.md written
- [ ] CLAUDE.md written
- [ ] Folder structure created
- [ ] DB models defined (Source, NewsItem)
- [ ] Alembic migrations created
- [ ] Sources seeded
- [ ] BaseScraper + scrapers implemented
- [ ] APScheduler configured
- [ ] Claude summariser service implemented
- [ ] API routes implemented
- [ ] Backend tested locally (curl /news returns items)
- [ ] Frontend feed view built
- [ ] Filters wired up
- [ ] Infinite scroll working
- [ ] Detail modal working
- [ ] CORS configured
- [ ] Deployed to Railway + Vercel
- [ ] Production DB migrated and seeded
- [ ] End-to-end verified on production