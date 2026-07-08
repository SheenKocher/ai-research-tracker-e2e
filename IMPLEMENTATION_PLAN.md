# IMPLEMENTATION_PLAN.md — Backend

Technical implementation plan for the AI Research Tracker backend. Derived from `SPEC.md` and `CLAUDE.md`. Scope: **backend only** (FastAPI + SQLAlchemy async + APScheduler + Claude summariser). Frontend is a later session.

Everything below follows the CLAUDE.md conventions: all endpoints `async def`, `httpx.AsyncClient` only, Pydantic v2 only, `AsyncSession` via `get_db()`, exact pinned versions, no hardcoded secrets.

---

## 1. IMPLEMENTATION ORDER

Build in this exact sequence. Each step depends only on steps before it, so every step is verifiable in isolation before moving on.

| # | Step | Why this position |
|---|------|-------------------|
| 1 | `app/config.py` — settings from env | Everything (DB, Claude, CORS) reads settings. Zero dependencies. |
| 2 | `app/database.py` — async engine, session factory, `get_db()` | Models, seed, API, and scheduler all need sessions. Depends only on config. |
| 3 | `app/models/` — `Source`, `NewsItem` ORM models | Alembic autogenerates from these. Scrapers/API depend on them. |
| 4 | Alembic init + first migration + run it | Tables must exist before seeding or any DB test. |
| 5 | `seed_sources.py` — insert all 12 sources | Scrapers query `Source` rows; nothing to scrape without them. |
| 6 | `app/schemas/` — Pydantic v2 response/internal schemas | API and scrapers pass typed data; no DB dependency, but conceptually models-first. |
| 7 | `app/services/tagger.py` — keyword tag assignment | Pure function, no I/O. Needed by the scrape pipeline before storing items. |
| 8 | `app/services/summariser.py` — Claude API wrapper | Independent of scrapers; pipeline calls it. Test it standalone with a sample text before wiring in. |
| 9 | `app/scrapers/base.py` — `BaseScraper` + `hash_url` + `fetch_with_retry` | All concrete scrapers extend this. |
| 10 | `app/scrapers/huggingface.py`, `arxiv.py`, `anthropic_news.py` | The three Phase-1 sources. Depend on base. |
| 11 | `app/services/scrape_runner.py` — orchestrates one full cycle | Ties scrapers + dedup + summariser + tagger + DB together. Needs everything above. |
| 12 | `app/api/news.py`, `app/api/sources.py`, `app/api/scrape.py` — routers | Depend on models, schemas, `get_db()`, and scrape_runner (for the trigger endpoint). |
| 13 | `app/services/scheduler.py` — APScheduler jobs | Wraps scrape_runner on intervals. Built after the manual trigger proves the pipeline works. |
| 14 | `app/main.py` — FastAPI app, lifespan, CORS, router mounting | Last: wires all of it together. |

Key ordering rationale: **the manual `POST /scrape/trigger` endpoint (step 12) is built before the scheduler (step 13)** so the entire pipeline can be verified with one curl before adding time-based behaviour, which is the hardest thing to debug.

---

## 2. MODULE BREAKDOWN

### `backend/app/config.py`
- **Purpose:** Central settings loaded from env vars via `pydantic-settings`. Never hardcode.
- **Contains:**
  - `class Settings(BaseSettings)` — fields: `database_url: str`, `anthropic_api_key: str`, `cors_origins: str` (comma-separated), `anthropic_model: str = "claude-sonnet-4-6"`
  - `settings = Settings()` module-level singleton
- **Depends on:** nothing.

### `backend/app/database.py`
- **Purpose:** Async engine, session factory, and the `get_db()` FastAPI dependency.
- **Contains:**
  - `engine = create_async_engine(settings.database_url, ...)`
  - `async_session_factory = async_sessionmaker(engine, expire_on_commit=False)`
  - `class Base(DeclarativeBase)` — shared declarative base
  - `async def get_db() -> AsyncGenerator[AsyncSession, None]` — yields a session, used with `Depends`
- **Depends on:** `config.py`.

### `backend/app/models/source.py`
- **Purpose:** `Source` ORM model — one row per feed, exactly per SPEC §3.
- **Contains:** `class Source(Base)` with columns `id`, `name`, `url`, `rss_url`, `category`, `poll_interval_hours`, `is_active`, `last_scraped_at`, `last_error`, `last_error_at`. Unique constraint on `name` (makes seeding idempotent).
- **Depends on:** `database.py` (Base).

### `backend/app/models/news_item.py`
- **Purpose:** `NewsItem` ORM model — one row per summarised article.
- **Contains:** `class NewsItem(Base)` with columns `id`, `url`, `url_hash` (unique index), `title`, `summary`, `published_at`, `scraped_at`, `source_id` (FK), `tags` (`ARRAY(String)` with GIN index), `raw_content` (nullable). Indexes exactly per SPEC §3.
- **Depends on:** `database.py`, `source.py`.

### `backend/app/models/__init__.py` *(modify existing empty stub)*
- **Purpose:** Re-export `Source`, `NewsItem` so Alembic's `env.py` imports one module and sees all tables.

### `backend/alembic/` (generated) + `backend/alembic.ini`
- **Purpose:** Async migrations. Initialised with the **async template** (see §3).
- **Contains:** `env.py` edited to import `Base.metadata` from `app.database` and models from `app.models`, and to read `DATABASE_URL` from env (never hardcoded in `alembic.ini`).
- **Depends on:** models.

### `backend/seed_sources.py`
- **Purpose:** Idempotent script inserting all 12 sources (3 active, 9 inactive) per SPEC §2.
- **Contains:** `SOURCES: list[dict]` constant + `async def seed() -> None` using `INSERT ... ON CONFLICT (name) DO NOTHING`; `if __name__ == "__main__": asyncio.run(seed())`.
- **Depends on:** `database.py`, `models/`.

### `backend/app/schemas/news.py`
- **Purpose:** Pydantic v2 schemas for the news API.
- **Contains:** `NewsItemOut` (all NewsItem fields incl. source name), `NewsFeedResponse` (`items: list[NewsItemOut]`, `next_cursor: int | None`, `has_more: bool`). All with `model_config = ConfigDict(from_attributes=True)`.
- **Depends on:** nothing (pure Pydantic).

### `backend/app/schemas/source.py`
- **Purpose:** `SourceOut` schema matching SPEC §4 `GET /sources` response.
- **Depends on:** nothing.

### `backend/app/schemas/scrape.py`
- **Purpose:** `ScrapeResult` schema — `sources_attempted`, `sources_succeeded`, `sources_failed`, `new_items`, `skipped_duplicates`, `errors: list[str]` — matching SPEC §4 trigger response. Also `ScrapedItem` (internal DTO returned by scrapers: `url`, `title`, `raw_content`, `published_at`).
- **Depends on:** nothing.

### `backend/app/services/tagger.py`
- **Purpose:** Pure keyword-heuristic tag assignment per SPEC §5.
- **Contains:** `VALID_TAGS` constant; `def assign_tags(title: str, raw_content: str | None, source_name: str) -> list[str]` (sync — pure CPU, no reason to be async).
- **Depends on:** nothing.

### `backend/app/services/summariser.py`
- **Purpose:** Claude API wrapper producing the 80–120 word summary.
- **Contains:** `SUMMARY_PROMPT` template (verbatim from CLAUDE.md); `async def summarise(raw_content: str) -> str` using `anthropic.AsyncAnthropic`, model from `settings.anthropic_model`, wrapped in tenacity retry.
- **Depends on:** `config.py`.

### `backend/app/scrapers/base.py`
- **Purpose:** Abstract base every scraper extends; shared fetch/hash utilities.
- **Contains:**
  - `def hash_url(url: str) -> str` — SHA256 of normalised URL (strip, rstrip `/`, split `?`), exactly the CLAUDE.md snippet
  - `@retry(...) async def fetch_with_retry(client, url)` — tenacity, 3 attempts, exp backoff 2–8s, 15s timeout
  - `class BaseScraper(ABC)` — holds `source: Source`, abstract `async def fetch(self, client: httpx.AsyncClient) -> list[ScrapedItem]`
  - `def parse_rss(xml_text: str) -> list[ScrapedItem]` — shared feedparser logic for the RSS scrapers
- **Depends on:** `schemas/scrape.py`, `models/source.py`.

### `backend/app/scrapers/huggingface.py`
- **Purpose:** HuggingFace Daily Papers JSON API scraper.
- **Contains:** `class HuggingFaceScraper(BaseScraper)` — fetches `source.rss_url`, parses JSON, builds URL as `https://huggingface.co/papers/{paper.id}`, maps `paper.summary` → `raw_content`, `publishedAt` → `published_at`.
- **Depends on:** `base.py`.

### `backend/app/scrapers/arxiv.py`
- **Purpose:** ArXiv cs.AI RSS scraper.
- **Contains:** `class ArxivScraper(BaseScraper)` — fetch via httpx, parse with `parse_rss`.
- **Depends on:** `base.py`.

### `backend/app/scrapers/anthropic_news.py`
- **Purpose:** Anthropic News RSS scraper.
- **Contains:** `class AnthropicNewsScraper(BaseScraper)` — same pattern as ArXiv.
- **Depends on:** `base.py`.

### `backend/app/scrapers/__init__.py` *(modify existing empty stub)*
- **Purpose:** `SCRAPER_REGISTRY: dict[str, type[BaseScraper]]` mapping `Source.name` → scraper class, so the runner picks scrapers by DB row with no if/else chains. Sources without a registered scraper are skipped with a log line (covers the 9 dormant Phase-2 sources if someone flips `is_active` early).

### `backend/app/services/scrape_runner.py`
- **Purpose:** Orchestrates one full scrape cycle: sources → scrapers (concurrent) → dedup → summarise → tag → store → update source status.
- **Contains:** `async def run_scrape_cycle(session, interval_hours=None) -> ScrapeResult` and helper `async def _process_source(...)`.
- **Depends on:** scrapers, summariser, tagger, models, schemas.

### `backend/app/services/scheduler.py`
- **Purpose:** APScheduler `AsyncIOScheduler` with three interval jobs (6h / 12h / 24h) that call `run_scrape_cycle` filtered by interval.
- **Contains:** `scheduler = AsyncIOScheduler()`; `def start_scheduler() -> None`; `def shutdown_scheduler() -> None`; `async def _scheduled_scrape(interval_hours: int) -> None` (opens its own session from `async_session_factory` — scheduler jobs cannot use the FastAPI `Depends` machinery).
- **Depends on:** `scrape_runner.py`, `database.py`.

### `backend/app/api/news.py`
- **Purpose:** `GET /news` (cursor pagination + filters) and `GET /news/{id}`.
- **Contains:** `router = APIRouter()`; `async def get_news(...)`, `async def get_news_item(item_id, db)`.
- **Depends on:** models, schemas, `get_db`.

### `backend/app/api/sources.py`
- **Purpose:** `GET /sources` — all sources with status.
- **Depends on:** models, schemas, `get_db`.

### `backend/app/api/scrape.py`
- **Purpose:** `POST /scrape/trigger` — runs a full cycle on demand, returns `ScrapeResult`.
- **Depends on:** `scrape_runner.py`, `get_db`.

### `backend/app/main.py` *(modify existing empty stub)*
- **Purpose:** FastAPI app factory: lifespan (start/stop scheduler), CORS middleware from `settings.cors_origins`, mount the three routers.
- **Depends on:** everything above.

### `backend/requirements.txt`
- **Purpose:** Pinned dependencies (see §6).

---

## 3. DATABASE SETUP

All commands run from `backend/` with the conda env active and `DATABASE_URL` exported (or in `.env`).

### 3.1 Alembic initialisation — use the **async template**

```bash
cd backend
alembic init -t async alembic
```

Then edit:
- `alembic.ini` — leave `sqlalchemy.url` blank (env.py injects it from env var; never commit a real URL)
- `alembic/env.py` —
  ```python
  import os
  from app.database import Base
  from app.models import Source, NewsItem  # noqa: F401 — registers tables on metadata

  config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
  target_metadata = Base.metadata
  ```

### 3.2 Generate + run the first migration

```bash
alembic revision --autogenerate -m "create sources and news_items tables"
# Inspect the generated file in alembic/versions/ — verify:
#   - unique index on news_items.url_hash
#   - index on news_items.published_at (descending)
#   - index on news_items.source_id
#   - GIN index on news_items.tags (postgresql_using="gin")
#   - unique constraint on sources.name
alembic upgrade head
```

Note: reading `alembic/versions/` to verify autogenerate output is required; *hand-editing* those files is forbidden by CLAUDE.md. If the GIN index doesn't autogenerate correctly, fix the **model** `__table_args__` and regenerate, don't patch the migration.

### 3.3 Seed script — all 12 sources

`backend/seed_sources.py`, idempotent via `ON CONFLICT (name) DO NOTHING`:

```python
"""Seed all 12 sources. Safe to run repeatedly."""
import asyncio
from sqlalchemy.dialects.postgresql import insert
from app.database import async_session_factory
from app.models import Source

SOURCES = [
    # Phase 1 — active
    {"name": "HuggingFace Daily Papers", "url": "https://huggingface.co/papers",
     "rss_url": "https://huggingface.co/api/daily_papers", "category": "api",
     "poll_interval_hours": 12, "is_active": True},
    {"name": "ArXiv cs.AI", "url": "https://arxiv.org/list/cs.AI/recent",
     "rss_url": "https://arxiv.org/rss/cs.AI", "category": "rss",
     "poll_interval_hours": 6, "is_active": True},
    {"name": "Anthropic News", "url": "https://www.anthropic.com/news",
     "rss_url": "https://www.anthropic.com/news.rss", "category": "rss",
     "poll_interval_hours": 24, "is_active": True},
    # Phase 2+ — seeded dormant (is_active=False):
    # TLDR AI (24h), Daily Dose of DS (24h, https://blog.dailydoseofds.com/feed),
    # Claude Code Releases (12h, https://releasebot.io/updates/anthropic/claude-code.rss),
    # The Batch (168h), Latent Space (168h, https://latent.space/feed),
    # AlphaSignal (168h), Ahead of AI (168h), Import AI (168h),
    # Papers with Code (168h, https://paperswithcode.com/api/v1/papers, category="api")
]

async def seed() -> None:
    """Insert all sources, skipping any that already exist by name."""
    async with async_session_factory() as session:
        for src in SOURCES:
            await session.execute(
                insert(Source).values(**src).on_conflict_do_nothing(index_elements=["name"])
            )
        await session.commit()

if __name__ == "__main__":
    asyncio.run(seed())
```

Run: `python seed_sources.py`. (The implementation session must fill in all 9 dormant rows explicitly — the SPEC §2 Phase-2 table has the URLs; Substack ones whose exact feed URL isn't in the spec get their best-known `https://<name>.substack.com/feed` form.)

---

## 4. DATA FLOW

```
                        APScheduler (6h / 12h / 24h jobs)          POST /scrape/trigger
                                      │                                     │
                                      └──────────────┬──────────────────────┘
                                                     ▼
                                        run_scrape_cycle(session)
                                                     │
                                 SELECT * FROM sources WHERE is_active
                                 [AND poll_interval_hours = X if scheduled]
                                                     │
                              asyncio.gather(*fetch tasks, return_exceptions=True)
                                                     │
                    ┌────────────────────────────────┼────────────────────────────────┐
                    ▼                                ▼                                ▼
          HuggingFaceScraper                  ArxivScraper                 AnthropicNewsScraper
          httpx GET (JSON API)              httpx GET → feedparser         httpx GET → feedparser
          15s timeout, 3 retries            15s timeout, 3 retries         15s timeout, 3 retries
                    │                                │                                │
                    └────────────── list[ScrapedItem] per source ─────────────────────┘
                                                     │
                                   for each item:  hash_url(item.url)
                                                     │
                                    url_hash already in DB (or seen
                                    earlier this cycle)?
                                        │ yes                │ no
                                        ▼                    ▼
                                   skip (count as     summarise(raw_content)
                                   skipped_duplicate)   Claude API, retried
                                   NO Claude call            │
                                                     assign_tags(title, raw, source)
                                                             │
                                                    INSERT NewsItem
                                                    (url, url_hash, title, summary,
                                                     published_at, scraped_at,
                                                     source_id, tags, raw_content)
                                                             │
                                          UPDATE sources SET last_scraped_at=now()
                                          (or last_error/last_error_at on failure)
                                                             │
                                                             ▼
                                                    PostgreSQL (Supabase)
                                                             │
                          ┌──────────────────────────────────┼─────────────────────┐
                          ▼                                  ▼                     ▼
                     GET /news                        GET /news/{id}          GET /sources
              filters + keyset cursor,               single item, 404       all rows + status
              ORDER BY published_at DESC                if missing
                          │
                          ▼
                   React frontend (later session)
```

The single most important invariant, enforced at one choke point in `run_scrape_cycle`: **the dedup check happens before `summarise()` is ever called.** A duplicate URL costs one SHA256 and one indexed DB lookup — never a Claude token.

---

## 5. FUNCTION SIGNATURES

All Python 3.12, all following CLAUDE.md conventions.

### `app/database.py`

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI dependency: yield an AsyncSession, always closed after the request."""
```

### `app/scrapers/base.py`

```python
def hash_url(url: str) -> str:
    """SHA256 of the normalised URL (stripped, trailing-slash- and query-free) — the dedup key."""

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=8))
async def fetch_with_retry(client: httpx.AsyncClient, url: str) -> httpx.Response:
    """GET url with a 15s timeout; 3 attempts with 2s→4s→8s backoff; raises after the third failure."""

def parse_rss(xml_text: str) -> list[ScrapedItem]:
    """Parse already-fetched RSS XML with feedparser into ScrapedItems (title, link, summary, published)."""

class BaseScraper(ABC):
    def __init__(self, source: Source) -> None: ...

    @abstractmethod
    async def fetch(self, client: httpx.AsyncClient) -> list[ScrapedItem]:
        """Fetch this scraper's feed and return normalised items; raises on unrecoverable fetch failure."""
```

### `app/scrapers/huggingface.py` / `arxiv.py` / `anthropic_news.py`

```python
class HuggingFaceScraper(BaseScraper):
    async def fetch(self, client: httpx.AsyncClient) -> list[ScrapedItem]:
        """GET the daily_papers JSON API; map paper.id → https://huggingface.co/papers/{id}."""

class ArxivScraper(BaseScraper):
    async def fetch(self, client: httpx.AsyncClient) -> list[ScrapedItem]:
        """GET the cs.AI RSS feed and delegate to parse_rss."""

class AnthropicNewsScraper(BaseScraper):
    async def fetch(self, client: httpx.AsyncClient) -> list[ScrapedItem]:
        """GET the Anthropic news RSS feed and delegate to parse_rss."""
```

### `app/services/summariser.py`

```python
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=8))
async def summarise(raw_content: str) -> str:
    """Call Claude (settings.anthropic_model) with the SPEC prompt; return the 80–120 word plain-text summary."""
```

### `app/services/tagger.py`

```python
def assign_tags(title: str, raw_content: str | None, source_name: str) -> list[str]:
    """Apply SPEC §5 keyword heuristics; multiple tags allowed; 'ai-engineering' fallback for HuggingFace."""
```

### `app/services/scrape_runner.py`

```python
async def run_scrape_cycle(
    session: AsyncSession, interval_hours: int | None = None
) -> ScrapeResult:
    """Scrape all active sources (optionally only one poll interval); dedup, summarise, tag, store; never raises."""

async def _process_source(
    source: Source, client: httpx.AsyncClient, session: AsyncSession,
    seen_hashes: set[str],
) -> tuple[int, int]:
    """Scrape one source and persist its new items; returns (new_items, skipped_duplicates); updates source status."""
```

### `app/services/scheduler.py`

```python
def start_scheduler() -> None:
    """Register the 6h/12h/24h interval jobs (max_instances=1, coalesce=True) and start AsyncIOScheduler."""

def shutdown_scheduler() -> None:
    """Stop the scheduler cleanly on app shutdown."""

async def _scheduled_scrape(interval_hours: int) -> None:
    """Job body: open a session from async_session_factory and run run_scrape_cycle for this interval."""
```

### `app/api/news.py`

```python
@router.get("/news", response_model=NewsFeedResponse)
async def get_news(
    source_id: int | None = None,
    tags: Annotated[list[str] | None, Query()] = None,
    from_date: date | None = None,
    to_date: date | None = None,
    cursor: int | None = None,
    limit: Annotated[int, Query(ge=1, le=50)] = 20,
    db: AsyncSession = Depends(get_db),
) -> NewsFeedResponse:
    """Paginated feed, newest first; all filters AND logic; keyset cursor = last item id."""

@router.get("/news/{item_id}", response_model=NewsItemOut)
async def get_news_item(item_id: int, db: AsyncSession = Depends(get_db)) -> NewsItemOut:
    """Single item by id; raises HTTPException(404) if not found."""
```

### `app/api/sources.py`

```python
@router.get("/sources", response_model=list[SourceOut])
async def get_sources(db: AsyncSession = Depends(get_db)) -> list[SourceOut]:
    """All sources, active and inactive, with last_scraped_at / last_error status."""
```

### `app/api/scrape.py`

```python
@router.post("/scrape/trigger", response_model=ScrapeResult)
async def trigger_scrape(db: AsyncSession = Depends(get_db)) -> ScrapeResult:
    """Manually run a full scrape cycle across all active sources; returns per-cycle counters."""
```

### `app/main.py`

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Start APScheduler on startup, shut it down on exit."""
```

---

## 6. ENVIRONMENT + DEPENDENCY SETUP

### `backend/requirements.txt` — exact pins, each justified

```
fastapi==0.115.12          # web framework (SPEC stack)
uvicorn[standard]==0.34.0  # ASGI server, CLAUDE.md run command
sqlalchemy==2.0.36         # ORM, 2.x async per CLAUDE.md
asyncpg==0.30.0            # async PostgreSQL driver for postgresql+asyncpg://
greenlet==3.1.1            # required by SQLAlchemy async internals (not always auto-installed)
alembic==1.14.0            # migrations (SPEC stack)
pydantic==2.10.6           # v2 schemas per CLAUDE.md
pydantic-settings==2.7.1   # Settings from env vars (never hardcode)
httpx==0.28.1              # all outbound HTTP (never requests)
apscheduler==3.11.0        # poll scheduling (SPEC §5)
tenacity==9.0.0            # retry with exponential backoff (CLAUDE.md scraper rules)
feedparser==6.0.11         # RSS parsing for ArXiv/Anthropic (SPEC §5)
anthropic==0.45.0          # Claude API client for summariser
python-dateutil==2.9.0.post0  # robust published_at parsing (HF ISO strings, RSS date variants)
```

Every entry maps to a SPEC/CLAUDE.md requirement; nothing extra. If pip resolution fails on any pin at implementation time, bump to the nearest compatible version and record it in the changelog — but keep it pinned.

### Setup commands, in order

```bash
# 1. Env (already created earlier)
conda activate ai-research-tracker

# 2. Dependencies
cd backend
pip install -r requirements.txt

# 3. Env vars — copy the example, fill in real values (never commit .env)
cp .env.example .env
# .env must define: DATABASE_URL, ANTHROPIC_API_KEY, CORS_ORIGINS
export $(grep -v '^#' .env | xargs)   # or use a dotenv loader in config.py

# 4. Migrations
alembic upgrade head

# 5. Seed
python seed_sources.py

# 6. Run
uvicorn app.main:app --reload --port 8000
```

`.env.example` (currently an empty stub — fill with):
```
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/dbname
ANTHROPIC_API_KEY=sk-ant-...
CORS_ORIGINS=http://localhost:5173
```

---

## 7. TESTING CHECKPOINTS

Run each checkpoint immediately after its step from §1. Do not proceed while one fails.

| After step | Command | Expected output |
|---|---|---|
| 1. config | `python -c "from app.config import settings; print(settings.anthropic_model)"` | `claude-sonnet-4-6` (env vars must be set first) |
| 2. database | `python -c "import asyncio; from app.database import engine; from sqlalchemy import text; asyncio.run(engine.connect().__aenter__().execute(text('SELECT 1')))"` — or a 5-line `scripts/check_db.py` | No exception → Supabase reachable via asyncpg |
| 3–4. models + migration | `alembic upgrade head` then `alembic current` | `head` revision shown; in Supabase dashboard: `sources` + `news_items` tables with all 4 indexes |
| 5. seed | `python seed_sources.py` **twice**, then `SELECT count(*), count(*) FILTER (WHERE is_active) FROM sources;` | `12, 3` after both runs (second run proves idempotency) |
| 7. tagger | `python -c "from app.services.tagger import assign_tags; print(assign_tags('Claude API update', None, 'Anthropic News'))"` | `['model-releases', 'platform-updates']` |
| 8. summariser | one-off script: `asyncio.run(summarise("<paste any abstract>"))` | Plain-text summary, 80–120 words, no bullets/markdown |
| 9–10. scrapers | one-off script instantiating each scraper against its real feed URL | Each returns a non-empty `list[ScrapedItem]` with populated url/title/published_at |
| 11–12. pipeline + API | `uvicorn app.main:app --port 8000` then `curl -X POST localhost:8000/scrape/trigger` | JSON matching SPEC §4: `sources_attempted: 3`, `new_items > 0`, `errors: []` |
| 12. dedup | `curl -X POST localhost:8000/scrape/trigger` **again immediately** | `new_items: 0`, `skipped_duplicates` ≈ previous `new_items` — proves no double Claude spend |
| 12. news API | `curl "localhost:8000/news?limit=5"` | 5 items, newest first, `next_cursor` set, `has_more: true` |
| 12. filters | `curl "localhost:8000/news?tags=research-paper&source_id=2"` | Only ArXiv items tagged research-paper |
| 12. 404 | `curl -i localhost:8000/news/999999` | `HTTP 404` with JSON detail |
| 12. sources | `curl localhost:8000/sources` | 12 rows; the 3 active ones have fresh `last_scraped_at` |
| 13. scheduler | Start app, check logs | Three jobs registered (6h/12h/24h) in startup logs; no immediate crash |
| 14. CORS | `curl -i -H "Origin: http://localhost:5173" localhost:8000/news` | `access-control-allow-origin: http://localhost:5173` header present |

---

## 8. WATCH OUT FOR

### Async mistakes
1. **feedparser is synchronous** — never give it a URL (it would do blocking network I/O inside the event loop). Fetch with `httpx.AsyncClient` first, then call `feedparser.parse(response.text)` on the in-memory string. Parsing a string is fast CPU work; that's acceptable inline.
2. **APScheduler jobs can't use `Depends(get_db)`** — that's FastAPI request machinery. Scheduler jobs must open their own session: `async with async_session_factory() as session:`.
3. **Use `AsyncIOScheduler`, not `BackgroundScheduler`** — the async coroutine jobs need the running event loop. Start it inside the FastAPI **lifespan** so the loop exists.
4. **Don't share one `AsyncSession` across `asyncio.gather` tasks** — AsyncSession is not concurrency-safe. Gather the *fetches* (pure httpx, no session), then process results and write to the DB **sequentially** with the one session. Fetch-concurrent, write-serial.
5. **`uvicorn --reload` restarts the whole process** — the scheduler restarts with it. Fine locally; just don't be surprised by duplicate "scheduler started" logs across reloads.

### Database / Supabase
6. **Supabase pooled connections (pgbouncer, port 6543) break asyncpg prepared statements.** Either connect to the direct port 5432, or pass `connect_args={"statement_cache_size": 0}` to `create_async_engine`. This is the single most common "works locally, dies on Supabase" failure.
7. **GIN index on `tags`** needs explicit `Index("ix_news_items_tags", NewsItem.tags, postgresql_using="gin")` in `__table_args__` — autogenerate won't invent it. Verify it's in the generated migration before `upgrade head`.
8. **Timezone-aware datetimes everywhere:** `DateTime(timezone=True)` columns, `datetime.now(timezone.utc)` for `scraped_at`. Mixing naive and aware datetimes raises `TypeError` at comparison time, not at insert time — it will bite during cursor pagination.
9. **Tags AND-logic filter** is PostgreSQL array containment: `NewsItem.tags.contains(tags_list)` → SQL `tags @> ARRAY[...]`. Don't loop `ANY()` per tag — that's OR logic.
10. **Cursor pagination ordering must be deterministic.** Spec: newest first by `published_at`, cursor = last item **id**. Two items can share `published_at`, so order by `(published_at DESC, id DESC)` and page with a tuple comparison: look up the cursor row's `published_at`, then `WHERE (published_at, id) < (:cursor_published_at, :cursor_id)`. Paging on `id < cursor` alone silently skips items whose ids and publish dates are out of order (backfills, slow sources).
11. **Seed idempotency requires the unique constraint on `sources.name`** — `ON CONFLICT (name)` errors without it. It's in the model spec above; don't drop it.

### Scraping / dedup
12. **Dedup within a single cycle, not just against the DB.** ArXiv RSS routinely repeats entries, and HF/ArXiv can surface the same paper in one batch. Keep a `seen_hashes: set[str]` for the cycle; check it *and* the DB before summarising. Otherwise a unique-constraint violation aborts the insert — or worse, you pay Claude twice before the DB stops you.
13. **`hash_url` must be the only hashing code path** — used by scrapers on ingest *and* by any future backfill. Any second implementation that normalises differently silently breaks dedup forever.
14. **ArXiv links redirect** (`http://` → `https://`, sometimes versioned paths). Create the shared client with `httpx.AsyncClient(follow_redirects=True, timeout=15.0)` — but hash the URL **from the feed entry**, not the post-redirect URL, so hashes stay stable across scrapes.
15. **`published_parsed` from feedparser is a UTC `struct_time`** — convert with `datetime.fromtimestamp(calendar.timegm(entry.published_parsed), tz=timezone.utc)`. Naive `datetime(*entry.published_parsed[:6])` loses tz-awareness (see #8). Some entries lack the field entirely → fall back to `datetime.now(timezone.utc)`.
16. **Skip items with empty `raw_content` instead of summarising the title alone** — the prompt expects real content; Claude on a bare title yields a hallucination-prone summary. Log and skip (open question from the model review — this is the recommended default; revisit if too many items get dropped).
17. **tenacity's `@retry` re-raises the original exception after attempt 3** wrapped behaviour: `_process_source` must catch `Exception` around the whole fetch, write `source.last_error` / `last_error_at`, and return — combined with `return_exceptions=True` at the gather level, one dead feed never kills the cycle.
18. **Cap items per cycle on first run.** The first-ever ArXiv scrape can return hundreds of entries → hundreds of Claude calls in one burst. Process at most ~50 new items per source per cycle (constant in `scrape_runner.py`); the rest are picked up next cycle. Protects both the API bill and rate limits.

### Claude API
19. **Use `anthropic.AsyncAnthropic`**, not the sync client — a sync client blocks the event loop for seconds per call.
20. **Set `max_tokens` ≈ 300** — 120 words needs ~180 tokens; 300 gives headroom without letting a runaway response spend money.
21. **Summariser retry should catch `anthropic.APIStatusError` / rate limits** via tenacity, same 3-attempt policy. If it still fails, skip the item *without storing it* — the url_hash won't be in the DB, so the next cycle retries it for free. Never store an item with an empty summary.

### Config
22. **`CORS_ORIGINS` is a comma-separated string in env** — split it in `config.py` before passing to `CORSMiddleware(allow_origins=[...])`. Passing the raw string "works" but matches nothing.
23. **`pydantic-settings` reads `.env` natively** — set `model_config = SettingsConfigDict(env_file=".env")` so local dev needs no manual `export`; Railway injects real env vars which take precedence.
