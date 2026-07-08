# AI Research Tracker — SPEC.md

## 1. Problem Statement

AI research moves faster than any individual can track manually. New papers drop on ArXiv daily, models ship without warning, and Anthropic releases are scattered across blog posts and changelogs. This app solves that by scraping a curated list of AI research sources on a schedule, summarising each item using Claude API, and displaying them in a filterable feed — automatically, with no manual intervention after deployment.

**Target user:** A single developer (you) staying current with AI research and tooling while building with it.

**Core constraint:** Every design decision optimises for low maintenance and low API cost, not for scale.

---

## 2. Data Sources

### Phase 1 — Active (`is_active=True`)

| Source | Feed URL | Type | Poll Interval |
|---|---|---|---|
| HuggingFace Daily Papers | `https://huggingface.co/api/daily_papers` | JSON API | 12h |
| ArXiv cs.AI | `https://arxiv.org/rss/cs.AI` | RSS | 6h |
| Anthropic News | `https://www.anthropic.com/news.rss` | RSS | 24h |

### Phase 2+ — Seeded but inactive (`is_active=False`)

| Source | Feed URL | Type | Poll Interval |
|---|---|---|---|
| TLDR AI | Substack RSS | RSS | 24h |
| Daily Dose of DS | `https://blog.dailydoseofds.com/feed` | RSS | 24h |
| Claude Code Releases | `https://releasebot.io/updates/anthropic/claude-code.rss` | RSS | 12h |
| The Batch (DeepLearning.AI) | Substack RSS | RSS | 168h (weekly) |
| Latent Space | `https://latent.space/feed` | RSS | 168h |
| AlphaSignal | Research + GitHub trending | RSS | 168h |
| Ahead of AI (Raschka) | Substack RSS | RSS | 168h |
| Import AI (Jack Clark) | Substack RSS | RSS | 168h |
| Papers with Code | `https://paperswithcode.com/api/v1/papers` | API | 168h |

**Rule:** All 12 sources are seeded as rows in the `Source` table on first migration. Enabling a new source in Phase 2 requires only flipping `is_active=True` — no code changes, no new migrations.

---

## 3. Data Model

### `Source`

```python
class Source(Base):
    __tablename__ = "sources"

    id: int                          # PK
    name: str                        # "HuggingFace Daily Papers"
    url: str                         # homepage URL (for display)
    rss_url: str                     # actual feed URL polled
    category: str                    # "rss" or "api"
    poll_interval_hours: int         # 6, 12, 24, or 168
    is_active: bool                  # True = polled, False = dormant
    last_scraped_at: datetime | None
    last_error: str | None
    last_error_at: datetime | None
```

### `NewsItem`

```python
class NewsItem(Base):
    __tablename__ = "news_items"

    id: int
    url: str                         # canonical article/paper URL
    url_hash: str                    # SHA256(url) — deduplication key, unique index
    title: str
    summary: str                     # 80–120 word Claude summary (cached forever)
    published_at: datetime           # from source feed
    scraped_at: datetime             # when we fetched it
    source_id: int                   # FK → sources.id
    tags: list[str]                  # e.g. ["research-paper", "agents"]
    raw_content: str | None          # original text before summarisation
```

### Indexes
- `UNIQUE INDEX` on `news_items.url_hash` — fast deduplication lookups
- `INDEX` on `news_items.published_at DESC` — feed ordering
- `INDEX` on `news_items.source_id` — source filter queries
- `INDEX` on `news_items.tags` (GIN for PostgreSQL) — tag filter queries

---

## 4. API Endpoints

All endpoints are async. All responses are JSON. All errors return structured `HTTPException` with appropriate status codes.

### `GET /news`

Returns paginated list of news items, newest first.

**Query params:**
| Param | Type | Default | Description |
|---|---|---|---|
| `source_id` | int | None | Filter by source |
| `tags` | list[str] | None | Filter by one or more tags (AND logic) |
| `from_date` | date | None | Items published on or after this date |
| `to_date` | date | None | Items published on or before this date |
| `cursor` | int | None | Last item ID seen (for infinite scroll) |
| `limit` | int | 20 | Items per page (max 50) |

**Response:**
```json
{
  "items": [...],
  "next_cursor": 142,
  "has_more": true
}
```

### `GET /news/{id}`

Returns a single news item by ID. Returns 404 if not found.

### `POST /scrape/trigger`

Manually triggers a full scrape cycle across all active sources. Used for local testing. Returns a summary of results per source.

**Response:**
```json
{
  "sources_attempted": 3,
  "sources_succeeded": 3,
  "sources_failed": 0,
  "new_items": 14,
  "skipped_duplicates": 3,
  "errors": []
}
```

### `GET /sources`

Returns all sources (active and inactive) with their current status.

**Response:**
```json
[
  {
    "id": 1,
    "name": "HuggingFace Daily Papers",
    "is_active": true,
    "last_scraped_at": "2025-07-01T06:00:00Z",
    "last_error": null
  }
]
```

---

## 5. Scraper Logic

### Poll cycle

APScheduler runs three jobs on separate schedules matching each source's `poll_interval_hours`. On each run:

1. Query DB for active sources matching that interval
2. Fetch all concurrently using `asyncio.gather(return_exceptions=True)`
3. For each result, check `return_exceptions` output — log failures, continue with successes
4. For each fetched item, compute `SHA256(url)` and check against `url_hash` in DB
5. If hash exists → skip (already summarised and stored)
6. If hash is new → call Claude API, store item with summary

### Deduplication

```python
import hashlib

def hash_url(url: str) -> str:
    # Strip trailing slashes and query params that vary by session
    normalized = url.strip().rstrip("/").split("?")[0]
    return hashlib.sha256(normalized.encode()).hexdigest()
```

Deduplication check happens **before** the Claude API call. URL hash is computed on ingest, checked against DB index, and only new items proceed to summarisation.

### Retry strategy

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=8)
)
async def fetch_with_retry(client: httpx.AsyncClient, url: str) -> httpx.Response:
    return await client.get(url, timeout=15.0)
```

Three attempts. Backoff: 2s → 4s → 8s. If all three fail, log the error to `Source.last_error` and `Source.last_error_at`, then continue the scrape cycle. One source failing never aborts the job.

### HuggingFace parser (API)

`GET https://huggingface.co/api/daily_papers` returns structured JSON. Extract:
- `paper.title`
- `paper.summary` (used as `raw_content`)
- `paper.id` → construct URL as `https://huggingface.co/papers/{id}`
- `publishedAt` → `published_at`

### RSS parser (ArXiv, Anthropic)

Use `feedparser` library. Extract:
- `entry.title`
- `entry.summary` or `entry.content[0].value` → `raw_content`
- `entry.link` → `url`
- `entry.published` → `published_at`

### Summarisation prompt

```
You are a technical summariser for an AI research feed read by engineers.

Summarise the following content in 80–120 words across these three parts:
1. What was built or discovered (be specific — name the model, method, or result)
2. Why it matters for AI engineering (practical implication, not hype)
3. One concrete thing an engineer could do with this today

Rules:
- Never use filler phrases: "this paper explores", "researchers have found", "in this work"
- Be direct. Start with the finding, not the context.
- Output plain text only. No bullet points, no headers, no markdown.

Content:
{raw_content}
```

### Tagging logic

Tags are assigned per item based on source + keyword heuristics applied to the title and raw_content. Applied after fetching, before storing.

```python
VALID_TAGS = [
    "research-paper",    # ArXiv, Papers with Code
    "model-releases",    # New models from any lab
    "platform-updates",  # API/SDK/tooling changes
    "ai-engineering",    # How to build with AI
    "agents",            # Agent frameworks, MCP, orchestration
    "open-source",       # Open weight models, repos
]
```

**Assignment rules:**
- `research-paper` → source is ArXiv or Papers with Code
- `model-releases` → title contains: "release", "launch", "gpt", "claude", "gemini", "llama", "mistral"
- `platform-updates` → title contains: "API", "SDK", "update", "v2", "changelog"
- `agents` → title contains: "agent", "MCP", "tool use", "orchestration", "agentic"
- `open-source` → title contains: "open", "weights", "open-source", "github"
- `ai-engineering` → default fallback if no other tag matches and source is HuggingFace

One item can have multiple tags.

---

## 6. Frontend Screens

### Tech stack
- React 18 + Vite
- shadcn/ui components (Card, Badge, Skeleton, Dropdown, Checkbox)
- Tailwind CSS
- Dark mode only (no toggle)

### Layout — Card feed with left sidebar

```
┌─────────────────────────────────────────────────────┐
│  AI Research Tracker          [last scraped: 2h ago] │
├──────────────┬──────────────────────────────────────┤
│              │                                       │
│  SOURCES     │  ┌─────────────────────────────────┐ │
│  □ HuggingFace│  │ [HuggingFace]  2h ago           │ │
│  □ ArXiv     │  │ Title of the paper or post      │ │
│  □ Anthropic │  │ Summary text here, 2 lines       │ │
│              │  │ visible before expand...         │ │
│  TAGS        │  │ [research-paper] [agents]        │ │
│  □ research  │  └─────────────────────────────────┘ │
│  □ agents    │                                       │
│  □ models    │  ┌─────────────────────────────────┐ │
│  □ open-src  │  │ [ArXiv]  5h ago                 │ │
│              │  │ ...                              │ │
│  DATE RANGE  │  └─────────────────────────────────┘ │
│  From: ____  │                                       │
│  To:   ____  │  [loading skeleton on scroll]         │
│              │                                       │
└──────────────┴──────────────────────────────────────┘
```

### Feed card (default state)
- Source badge (colored pill: HuggingFace=blue, ArXiv=green, Anthropic=purple)
- Time since scraped ("2h ago")
- Title (bold, 1–2 lines)
- Summary preview (first 2 lines, truncated with ellipsis)
- Tag badges (clickable → adds tag to active filters)
- "Read more" link → opens detail modal

### Detail modal
- Full title
- Full 80–120 word summary
- Tags (clickable)
- Published date
- Source name + link to original
- Raw content (collapsed by default, expandable)

### Sidebar filters
- **Sources:** Checkbox per source (checked = shown)
- **Tags:** Checkbox per tag from `VALID_TAGS`
- **Date range:** Two date inputs (from / to)
- All filters are AND logic
- Active filters shown as dismissible pills above the feed

### Infinite scroll
- Initial load: 20 items
- On scroll to bottom: fetch next 20 using cursor from last item ID
- Loading state: skeleton cards (shadcn Skeleton component)
- End state: "You're all caught up." message when `has_more=false`

### Empty states
- No items matching filters: "No items match your filters. Try removing one."
- Feed empty (first run): "Nothing scraped yet. Trigger a scrape to get started."

---

## 7. Deployment

| Component | Platform | Config |
|---|---|---|
| Backend | Railway | Dockerfile |
| Frontend | Vercel | `VITE_API_URL` env var |
| Database | Supabase (free tier) | AsyncSession only, run Alembic migrations on deploy |

### Environment variables

**Backend (Railway):**
```
DATABASE_URL=postgresql+asyncpg://...
ANTHROPIC_API_KEY=sk-ant-...
CORS_ORIGINS=https://your-app.vercel.app
```

**Frontend (Vercel):**
```
VITE_API_URL=https://your-backend.railway.app
```

### Deployment checklist
1. Push to GitHub
2. Railway: connect repo, set env vars, deploy backend
3. Supabase: run `alembic upgrade head` against production DB
4. Seed `Source` table with all 12 sources (migration or seed script)
5. Vercel: connect repo, set `VITE_API_URL`, deploy frontend
6. Fix CORS: add Vercel URL to `CORS_ORIGINS` in Railway
7. Hit `POST /scrape/trigger` to verify end-to-end pipeline
8. Confirm `GET /news` returns summarised items in browser

---

## 8. Out of Scope (Phase 1)

The following are explicitly excluded. Do not implement them unless this spec is updated:

- User authentication or multi-user support
- Email digests or push notifications
- Mobile-specific layouts
- Search (by title or content)
- Light mode / theme toggle
- Bookmarking or saving items
- Admin UI for managing sources
- Custom scrape intervals per user
- Consecutive failure tracking on sources
- Rate limiting on API endpoints
- Pagination (infinite scroll only)