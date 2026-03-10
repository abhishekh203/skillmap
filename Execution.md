# WAR ROOM — Senior Engineer Execution Plan

> **Purpose**: Step-by-step execution guide to build the War Room digest engine on top of the NoNewsBackend.
> **Audience**: Senior engineer responsible for implementation.
> **Estimated effort**: 14-22 working days.
> **Prerequisite reading**: Skim the other docs in `/warroom/` for context, especially `01_BACKEND_REUSE_MAP.md` and `02_PILLAR_NEWSLETTER_MAPPING.md`.

---

## Codebase Quick Reference

```
cloud_functions/api_service/
├── main.py                   (192 lines)  — Flask app, blueprint registration, error handlers
├── config.py                 (178 lines)  — Env vars, constants, shared config
├── service_registry.py       (106 lines)  — HTTP session, DB/service init, rate limiter
├── database.py               (1811 lines) — StoryDatabase, NewsletterDatabase, etc.
├── story_service.py          (369 lines)  — Story fetching, GCS content, summary orchestration
├── newsletter_service.py     (2041 lines) — Newsletter generation, profile matching, batch gen
├── llm_service.py            (683 lines)  — OpenRouter/Gemini, 7 prompt types, embeddings
├── routes_stories.py         (463 lines)  — Story API endpoints
├── routes_newsletters.py     (894 lines)  — Newsletter CRUD + generation endpoints
├── routes_marketplace.py     (623 lines)  — Marketplace + subscription endpoints
├── utils_auth.py             (186 lines)  — JWT verification, @require_auth
├── utils_validation.py       (426 lines)  — Input validation, sanitization
├── utils_errors.py           (230 lines)  — Error hierarchy, logging
├── utils_helpers.py          (186 lines)  — CORS, helpers, response formatting
└── requirements.txt                       — Dependencies

db/migrations/                             — 42 SQL migration files (001-041 + schema.md)
```

**Tech stack**: Python/Flask, Supabase (PostgreSQL + pgvector), OpenRouter (Gemini 2.5 Flash Lite), GCS, Cloud Pub/Sub, Cloud Functions.

**Key pattern**: Services follow `init_*_service()` factory pattern (see `service_registry.py:75-76`). DB classes take `http_session` as constructor arg. All Supabase calls use REST API via `requests`.

---

## SPRINT 1: Database & Pillar Config (Days 1-3)

### Task 1.1 — Create `pillar_configs` table

**Create file**: `db/migrations/042_create_pillar_configs.sql`

```sql
CREATE TABLE IF NOT EXISTS pillar_configs (
    id SERIAL PRIMARY KEY,
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    icon TEXT,
    sort_order INTEGER DEFAULT 0,
    entity_filters TEXT[] DEFAULT '{}',
    source_ids INTEGER[],
    keywords TEXT[] DEFAULT '{}',
    pillar_embedding vector(768),
    max_stories INTEGER DEFAULT 10,
    hours_back INTEGER DEFAULT 24,
    refresh_interval_hours INTEGER DEFAULT 3,
    is_active BOOLEAN DEFAULT TRUE,
    platform TEXT DEFAULT 'warroom',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pillar_configs_slug ON pillar_configs(slug);
CREATE INDEX idx_pillar_configs_active ON pillar_configs(is_active) WHERE is_active = TRUE;
```

Run in Supabase SQL editor.

### Task 1.2 — Seed the 5 pillars

**Create file**: `db/migrations/043_seed_pillar_configs.sql`

Insert these rows (full SQL in `03_NEW_TABLES_AND_MIGRATIONS.md`):

| slug | name | entity_filters |
|------|------|---------------|
| `ali-digest` | President Ali | `{Irfaan Ali, President Ali, Dr. Ali, Dr. Irfaan Ali}` |
| `jagdeo-digest` | VP Jagdeo | `{Bharrat Jagdeo, VP Jagdeo, Vice President Jagdeo}` |
| `azruddin-digest` | Azruddin Mohamed | `{Azruddin Mohamed}` |
| `opposition-digest` | Opposition | `{Aubrey Norton, APNU+AFC, APNU, AFC, PNC}` |
| `live-guyana-digest` | Live Guyana + International | `{}` (empty — source-based) |

### Task 1.3 — Create `pillar_digests` table

**Create file**: `db/migrations/044_create_pillar_digests.sql`

```sql
CREATE TABLE IF NOT EXISTS pillar_digests (
    id SERIAL PRIMARY KEY,
    pillar_id INTEGER NOT NULL REFERENCES pillar_configs(id) ON DELETE CASCADE,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    edition_date DATE NOT NULL,
    content JSONB NOT NULL DEFAULT '{}',
    story_ids INTEGER[] DEFAULT '{}',
    story_count INTEGER DEFAULT 0,
    sentiment TEXT,
    docx_url TEXT,
    pdf_url TEXT,
    generation_method TEXT DEFAULT 'scheduled',
    generation_time_ms INTEGER,
    status TEXT DEFAULT 'active',
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pillar_digests_pillar_date ON pillar_digests(pillar_id, edition_date DESC);
CREATE UNIQUE INDEX idx_pillar_digests_unique ON pillar_digests(pillar_id, edition_date, generated_at);
```

### Task 1.4 — Create `match_stories_for_pillar` RPC

**Create file**: `db/migrations/046_create_match_stories_for_pillar.sql`

This is the critical new pgvector function. It extends the existing `match_stories_for_newsletter` (migration `030`) with **source-level filtering**.

Full SQL in `03_NEW_TABLES_AND_MIGRATIONS.md`. Key differences from existing RPC:
- Accepts `pillar_source_ids INTEGER[]` parameter
- Adds a source filter step: joins `story_articles → artifacts → raw_items` to check `source_id = ANY(pillar_source_ids)`
- For entity-based pillars (1-4), **requires at least one entity match** (not just semantic similarity)
- For source-based pillars (5), pure semantic + source filter

**Test after creation**:
```sql
-- Should return stories mentioning Ali
SELECT * FROM match_stories_for_pillar(
    query_embedding := (SELECT pillar_embedding FROM pillar_configs WHERE slug = 'ali-digest'),
    pillar_entities := ARRAY['Irfaan Ali', 'President Ali'],
    pillar_source_ids := ARRAY[]::INTEGER[],
    match_threshold := 0.4,
    match_count := 10,
    hours_back := 48
);
```

### Task 1.5 — Add War Room RSS sources

**Create file**: `scripts/seed_warroom_sources.py`

Add 12 sources to the `sources` table. Verify RSS URLs first. Use the existing `sources` table format:
```json
{"rss_url": "https://www.stabroeknews.com/feed/", "scrape_content": true}
```

Reference the source list in `08_SOURCES_AND_ENTITIES.md`.

### Task 1.6 — Generate pillar embeddings

**Create file**: `scripts/generate_pillar_embeddings.py`

```python
from llm_service import generate_profile_embedding
# For each pillar: embed description → UPDATE pillar_configs SET pillar_embedding = ...
```

Uses the existing `generate_profile_embedding()` at `llm_service.py:79` — same Gemini embedding-001 model (768-dim) used for stories and newsletters.

### Sprint 1 Checklist
- [ ] All 4 migrations executed without errors
- [ ] `SELECT count(*) FROM pillar_configs` = 5
- [ ] `SELECT count(*) FROM sources WHERE name ILIKE '%guyana%' OR name ILIKE '%kaieteur%'` returns expected count
- [ ] All pillars have non-NULL `pillar_embedding`
- [ ] `match_stories_for_pillar()` returns results (at least for `live-guyana-digest` if sources are active)

---

## SPRINT 2: Digest Generation Engine (Days 4-7)

### Task 2.1 — Add `PillarDatabase` class to `database.py`

**Modify file**: `cloud_functions/api_service/database.py`

Add a new class after the existing `StoryAIContentDatabase` class (~line 1800):

```python
class PillarDatabase:
    """Database operations for War Room pillars and digests."""

    def __init__(self, http_session):
        self.http_session = http_session

    def get_pillar_by_slug(self, slug: str) -> Optional[Dict]:
        """Fetch pillar config. Query: pillar_configs?slug=eq.{slug}"""

    def get_all_active_pillars(self) -> List[Dict]:
        """Fetch all active pillars. Query: pillar_configs?is_active=eq.true&order=sort_order"""

    def get_latest_digest(self, pillar_id: int, edition_date: str = None) -> Optional[Dict]:
        """Fetch most recent digest. Query: pillar_digests?pillar_id=eq.{id}&order=generated_at.desc&limit=1"""

    def get_digest_history(self, pillar_id: int, days: int = 7) -> List[Dict]:
        """Fetch digest history for a pillar."""

    def store_digest(self, pillar_id, edition_date, content, story_ids, sentiment, generation_method, generation_time_ms) -> Optional[Dict]:
        """Insert into pillar_digests. POST to Supabase REST API."""

    def match_stories_for_pillar(self, query_embedding, pillar_entities, pillar_source_ids, match_threshold, match_count, hours_back) -> Optional[List[Dict]]:
        """Call the match_stories_for_pillar RPC. POST to /rest/v1/rpc/match_stories_for_pillar"""
```

**Follow the exact same patterns** used in `StoryDatabase` and `NewsletterDatabase`:
- Use `self.http_session.get/post()` with `get_supabase_headers()`
- Use `DATABASE_TIMEOUT` for timeouts
- Return `None` on failure, `[]` on empty, `list[dict]` on success
- Wrap everything in try/except with `logging.error()`

### Task 2.2 — Add 3 new LLM prompts to `llm_service.py`

**Modify file**: `cloud_functions/api_service/llm_service.py`

Add after existing prompt functions (~line 680):

```python
def create_sentiment_prompt(article_text, entity_name=None):
    """Classify overall tone as POSITIVE/NEUTRAL/NEGATIVE."""
    # See 05_DIGEST_GENERATION_ENGINE.md for full prompt

def create_digest_intro_prompt(pillar_name, story_headlines):
    """Generate 2-3 sentence executive briefing intro."""
    # See 05_DIGEST_GENERATION_ENGINE.md for full prompt

def create_digest_sentiment_prompt(pillar_name, story_sentiments):
    """Generate overall sentiment note for entire digest."""
    # See 05_DIGEST_GENERATION_ENGINE.md for full prompt
```

**Important**: Keep `max_tokens=1200` (existing config). These prompts are short — output will be well under limit.

### Task 2.3 — Create `digest_service.py`

**Create file**: `cloud_functions/api_service/digest_service.py`

This is the **core new file** (~400 lines). Key class:

```python
class DigestService:
    def __init__(self, http_session, pillar_db, story_db, story_service, story_ai_db):
        ...

    def generate_pillar_digest(self, pillar_slug: str, force: bool = False) -> dict:
        """
        Main entry point. One-click digest generation.

        Flow:
        1. Load pillar from pillar_db.get_pillar_by_slug()
        2. Check cache: pillar_db.get_latest_digest() — if fresh & not force, return it
        3. Search stories: pillar_db.match_stories_for_pillar()
        4. Enrich each story (see _enrich_story below)
        5. Generate digest-level content (intro, overall sentiment)
        6. Store: pillar_db.store_digest()
        7. Return digest content
        """

    def _enrich_story(self, story: dict) -> dict:
        """
        For each matched story, build the digest entry.

        OPTIMIZATION — check story_ai_content FIRST:
        1. story_ai_db.get_ai_content(story_id)  ← existing table/method
        2. If concise_summary exists → use it (skip LLM call)
        3. If quotes_data exists → use it (skip LLM call)
        4. Fallback: use story['summary'] from clustering (always available)
        5. Only NEW LLM call: sentiment (batch all stories in one call)

        Also fetch source URLs:
        - story_service.get_source_urls_for_story(story_id)  ← reuse existing
        """

    def _batch_analyze_sentiment(self, stories: list) -> list:
        """
        Single LLM call to classify sentiment for all stories at once.
        Reduces 10 LLM calls → 1 call.
        """

    def _generate_digest_intro(self, pillar: dict, stories: list) -> str:
        """1 LLM call: create_digest_intro_prompt() → generate_with_gemini()"""

    def _generate_overall_sentiment(self, pillar: dict, story_sentiments: list) -> dict:
        """1 LLM call: create_digest_sentiment_prompt() → generate_with_gemini()"""

    def refresh_all_pillars(self, pillar_slugs=None, force=False) -> dict:
        """
        Called by Cloud Scheduler endpoint.
        Loop through all active pillars → generate_pillar_digest() each.
        Sequential — only 5 pillars, no need for threading.
        Return summary of results.
        """
```

**Performance target**: Generate one pillar digest in < 10 seconds (mostly DB reads + 2-3 LLM calls).

**Key reuse points**:
- `story_service.get_source_urls_for_story()` at `story_service.py:183` — already fetches source URLs
- `story_ai_db.get_ai_content()` in `database.py` — already fetches pre-computed summaries/quotes
- `generate_with_gemini()` at `llm_service.py:55` — existing LLM generation function
- Story summary fallback: `story['summary']` is always populated by the clustering worker

### Task 2.4 — Register new services in `service_registry.py`

**Modify file**: `cloud_functions/api_service/service_registry.py`

Add at line 21 (imports):
```python
from database import PillarDatabase
from digest_service import DigestService, init_digest_service
```

Add after line 76 (service init):
```python
# War Room services
pillar_db = PillarDatabase(http_session)
digest_service = init_digest_service(http_session, pillar_db, story_db, story_service, story_ai_db)
```

### Sprint 2 Checklist
- [ ] `digest_service.generate_pillar_digest('ali-digest')` returns valid JSON
- [ ] Digest contains: header, introduction, stories[], quotes[], sentiment
- [ ] Each story has: headline, source, time_ago, summary, article_count
- [ ] Digest stored in `pillar_digests` table after generation
- [ ] Second call with `force=False` returns cached result (no re-generation)
- [ ] Generation time < 15 seconds per pillar
- [ ] All 5 pillars generate successfully (some may have 0 stories if sources aren't active yet — that's OK)

---

## SPRINT 3: API Endpoints (Days 8-10)

### Task 3.1 — Create `routes_warroom.py`

**Create file**: `cloud_functions/api_service/routes_warroom.py`

Follow the exact pattern of `routes_newsletters.py` and `routes_stories.py`:

```python
"""
War Room Routes — Pillar digests, story filtering, and export endpoints.
"""
from flask import Blueprint, request, jsonify
from service_registry import digest_service, pillar_db, limiter
from utils_auth import require_auth
from utils_validation import validate_string, validate_integer, ValidationError
from utils_errors import APIError, ResourceNotFoundError
from utils_helpers import cors_handler

warroom_bp = Blueprint('warroom', __name__)
```

**Implement these endpoints** (full specs in `04_API_ENDPOINTS.md`):

| Priority | Endpoint | Auth | Rate Limit |
|----------|----------|------|------------|
| P0 | `GET /api/warroom/pillars` | Yes | 60/min |
| P0 | `POST /api/warroom/digests/generate/<slug>` | Yes | 10/min |
| P0 | `GET /api/warroom/digests/<slug>/latest` | Yes | 60/min |
| P1 | `GET /api/warroom/digests/<slug>/history` | Yes | 30/min |
| P1 | `POST /api/warroom/digests/refresh` | Internal | 5/min |
| P2 | `POST /api/warroom/digests/<slug>/export` | Yes | 10/min |
| P2 | `POST /api/warroom/digests/<slug>/email` | Yes | 5/min |
| P2 | `GET /api/warroom/stories` | Yes | 60/min |

**Input validation**: Use existing `validate_string()`, `validate_integer()` from `utils_validation.py:1-426`.
**Error handling**: Use existing `@cors_handler`, `APIError` hierarchy from `utils_errors.py`.
**Auth**: Use existing `@require_auth` decorator from `utils_auth.py`.

### Task 3.2 — Register blueprint in `main.py`

**Modify file**: `cloud_functions/api_service/main.py`

At line 40 (imports), add:
```python
from routes_warroom import warroom_bp
```

At line 65 (blueprint registration), add:
```python
app.register_blueprint(warroom_bp)
```

### Task 3.3 — Add CORS origins to `config.py`

**Modify file**: `cloud_functions/api_service/config.py`

At line 51-56 (`ALLOWED_ORIGINS`), add:
```python
'https://v0-guyana-news-war-room.vercel.app',   # War Room staging
# 'https://warroom.gy',                          # War Room production (add when ready)
```

Add new constants after line 106:
```python
# War Room Configuration
WARROOM_DIGEST_REFRESH_HOURS = 3
WARROOM_MAX_STORIES_PER_DIGEST = 10
WARROOM_EXPORT_BUCKET = os.environ.get('WARROOM_EXPORT_BUCKET', 'warroom-exports')
```

### Sprint 3 Checklist
- [ ] `curl GET /api/warroom/pillars` (with auth header) returns 5 pillars
- [ ] `curl POST /api/warroom/digests/generate/ali-digest` returns full digest
- [ ] `curl GET /api/warroom/digests/ali-digest/latest` returns cached digest
- [ ] Rate limiting works (11th request in 1 minute gets 429)
- [ ] Invalid slug returns 404
- [ ] Missing auth returns 401
- [ ] CORS preflight from War Room domain succeeds
- [ ] Frontend can call endpoints and render digest data

---

## SPRINT 4: Document Export (Days 11-13)

### Task 4.1 — Add dependencies

**Modify file**: `cloud_functions/api_service/requirements.txt`

Add:
```
python-docx>=1.1.0
fpdf2>=2.8.0
```

### Task 4.2 — Create `export_service.py`

**Create file**: `cloud_functions/api_service/export_service.py`

~350 lines. Three generators + GCS upload:

```python
class ExportService:
    def __init__(self, gcs_bucket_name):
        self.bucket_name = gcs_bucket_name

    def generate_docx(self, digest_content: dict) -> bytes:
        """python-docx: Title → Header → Intro → Stories → Quotes → Sentiment → Footer"""

    def generate_pdf(self, digest_content: dict) -> bytes:
        """fpdf2: Same layout as Word but in PDF"""

    def generate_csv(self, stories: list) -> bytes:
        """csv module: Rank, Headline, Source, Time, Articles, Sentiment, Summary"""

    def upload_to_gcs(self, file_bytes, filename, content_type) -> str:
        """Upload to GCS, return signed URL (24h expiry)"""

    def export_digest(self, digest_content, format='docx') -> dict:
        """Main entry: generate → upload → return {download_url, format, file_size}"""
```

Full implementation in `06_DOCUMENT_EXPORT.md`. Follow the document template layout exactly as specified.

### Task 4.3 — Wire export endpoint

**Modify file**: `cloud_functions/api_service/routes_warroom.py`

Add the `POST /api/warroom/digests/<slug>/export` handler that:
1. Fetches latest digest from DB (or specific `digest_id` if provided)
2. Calls `export_service.export_digest(content, format)`
3. Returns `{download_url, expires_at, format, file_size_bytes}`

### Task 4.4 — Create GCS bucket

```bash
gsutil mb -l asia-south1 gs://warroom-exports/
gsutil lifecycle set '{"rule":[{"action":{"type":"Delete"},"condition":{"age":30}}]}' gs://warroom-exports/
```

### Sprint 4 Checklist
- [ ] `POST /api/warroom/digests/ali-digest/export` with `format=docx` returns download URL
- [ ] Download URL works (not expired, correct content-type)
- [ ] `.docx` file opens correctly in Microsoft Word / Google Docs
- [ ] `.pdf` file renders correctly in browser / PDF viewer
- [ ] CSV export contains correct columns and data
- [ ] Document layout matches the template in `06_DOCUMENT_EXPORT.md`
- [ ] GCS files auto-delete after 30 days

---

## SPRINT 5: Background Refresh + Email (Days 14-17)

### Task 5.1 — Background auto-refresh

Already built in Sprint 2 (`digest_service.refresh_all_pillars()`).
Already has endpoint from Sprint 3 (`POST /api/warroom/digests/refresh`).

**Create Cloud Scheduler job**:
```bash
gcloud scheduler jobs create http warroom-digest-refresh \
    --schedule="0 */3 * * *" \
    --uri="https://YOUR_CLOUD_RUN_URL/api/warroom/digests/refresh" \
    --http-method=POST \
    --headers="Content-Type=application/json" \
    --body='{"force": false}' \
    --oidc-service-account-email=YOUR_SA@PROJECT.iam.gserviceaccount.com \
    --time-zone="America/Guyana"
```

Runs every 3 hours. Adjust `--schedule` based on need (the brief says 1-6 hours).

### Task 5.2 — Email delivery (if using SendGrid)

**Modify file**: `cloud_functions/api_service/requirements.txt` — add `sendgrid>=6.11.0`
**Modify file**: `cloud_functions/api_service/config.py` — add `SENDGRID_API_KEY`, `WARROOM_EMAIL_FROM`
**Modify file**: `cloud_functions/api_service/export_service.py` — add `send_digest_email()` method
**Modify file**: `cloud_functions/api_service/routes_warroom.py` — wire `POST /api/warroom/digests/<slug>/email`

Full SendGrid implementation in `06_DOCUMENT_EXPORT.md`.

### Sprint 5 Checklist
- [ ] Cloud Scheduler job visible in GCP console
- [ ] Job fires every 3 hours and returns 200
- [ ] All 5 pillars get fresh digests after each run
- [ ] `GET /api/warroom/digests/ali-digest/latest` returns digest < 3h old
- [ ] Email endpoint sends correctly to test recipient
- [ ] Email contains formatted HTML body + PDF/DOCX attachment

---

## SPRINT 6: Story Filtering + Polish (Days 18-22)

### Task 6.1 — War Room story list endpoint

**Modify file**: `cloud_functions/api_service/routes_warroom.py`

Implement `GET /api/warroom/stories` with filters:
- `sources` — filter by source name/ID (join through `story_articles → artifacts → raw_items → sources`)
- `entities` — filter by `story_entities.entity_name`
- `time_range` — 1h/6h/24h/7d/custom
- `limit`, `offset` — pagination

This powers the main War Room story grid (visible in screenshots).

### Task 6.2 — Sentiment field on stories (optional)

If the War Room frontend needs per-story sentiment tags (FAVORABLE/NEUTRAL shown in screenshots), either:
- **Option A**: Compute on-the-fly via LLM (slow, expensive)
- **Option B**: Pre-compute during clustering/categorization (add sentiment to `story_ai_content`)
- **Option C**: Let the frontend handle sentiment classification client-side

**Recommendation**: Option B — add a background job that periodically computes sentiment for new stories and stores in `story_ai_content.sentiment` (new column).

### Task 6.3 — Admin endpoints (optional)

If needed:
- `PUT /api/warroom/pillars/<slug>` — update pillar config (entity_filters, source_ids, etc.)
- `POST /api/warroom/pillars` — create new pillar

Low priority — can be done via Supabase dashboard initially.

---

## Dependency Graph

```
Sprint 1 (DB + Config)
    ↓
Sprint 2 (Digest Engine)  ←── depends on Sprint 1
    ↓
Sprint 3 (API Endpoints)  ←── depends on Sprint 2
    ↓
    ├── Sprint 4 (Export)  ←── depends on Sprint 3
    ├── Sprint 5 (Refresh + Email)  ←── depends on Sprint 3 + Sprint 4
    └── Sprint 6 (Filtering + Polish)  ←── depends on Sprint 3
```

Sprints 4, 5, 6 can run **in parallel** if multiple engineers are available.

---

## Files Summary — What Gets Created vs Modified

### NEW FILES (6 files, ~1,600 lines)

| File | Sprint | ~Lines | Purpose |
|------|--------|--------|---------|
| `db/migrations/042_create_pillar_configs.sql` | 1 | 30 | Pillar config table |
| `db/migrations/043_seed_pillar_configs.sql` | 1 | 40 | 5 pillar seed data |
| `db/migrations/044_create_pillar_digests.sql` | 1 | 40 | Digest storage table |
| `db/migrations/046_create_match_stories_for_pillar.sql` | 1 | 80 | pgvector RPC with source filtering |
| `cloud_functions/api_service/digest_service.py` | 2 | 400 | Core digest generation engine |
| `cloud_functions/api_service/export_service.py` | 4 | 350 | Word/PDF/CSV/email export |
| `cloud_functions/api_service/routes_warroom.py` | 3 | 500 | All War Room API endpoints |
| `scripts/seed_warroom_sources.py` | 1 | 60 | Add RSS sources to DB |
| `scripts/generate_pillar_embeddings.py` | 1 | 40 | One-time embedding generation |

### MODIFIED FILES (5 files, ~350 lines added)

| File | Sprint | Lines Added | What Changes |
|------|--------|-------------|-------------|
| `database.py` | 2 | +200 | Add `PillarDatabase` class at bottom |
| `llm_service.py` | 2 | +80 | Add 3 new prompt functions at bottom |
| `service_registry.py` | 2 | +20 | Import + init PillarDatabase, DigestService |
| `main.py` | 3 | +5 | Import + register `warroom_bp` blueprint |
| `config.py` | 3 | +15 | CORS origins + War Room constants |
| `requirements.txt` | 4 | +3 | python-docx, fpdf2, sendgrid |

### UNTOUCHED FILES (no modifications needed)

- `newsletter_service.py` — not modified, logic reused via `story_service` and `database` layers
- `story_service.py` — not modified, methods called directly by `digest_service`
- `routes_stories.py` — not modified, existing endpoints reused as-is by War Room frontend
- `routes_newsletters.py` — not modified
- `routes_marketplace.py` — not modified
- `utils_auth.py` — not modified, `@require_auth` reused
- `utils_validation.py` — not modified, validators reused
- `utils_errors.py` — not modified, error classes reused
- `utils_helpers.py` — not modified, helpers reused
- All worker files (`news_article_processor_worker`, `story_clustering_worker`, etc.) — not modified

---

## Environment Variables to Add

```bash
# Add to Cloud Run environment / .env
WARROOM_EXPORT_BUCKET=warroom-exports          # GCS bucket for document exports
SENDGRID_API_KEY=SG.xxxxx                       # Only if using email (Sprint 5)
WARROOM_EMAIL_FROM=digest@warroom.gy           # Only if using email (Sprint 5)
```

Existing env vars (`SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `OPENROUTER_API_KEY`, etc.) are already set and shared.

---

## Testing Strategy

### Unit Testing (per sprint)
- Sprint 2: Call `digest_service.generate_pillar_digest()` directly — verify JSON structure
- Sprint 3: `curl` each endpoint — verify status codes, auth, rate limits
- Sprint 4: Generate test documents — open and verify layout

### Integration Testing
- Full flow: Cloud Scheduler → refresh endpoint → all pillars → verify DB has fresh digests
- Frontend integration: War Room UI calls API → renders digest → export works

### Load Testing (optional)
- Simulate 5 concurrent "generate" requests (one per pillar)
- Verify generation time stays < 15s per pillar
- Verify no DB connection exhaustion (connection pooling in `service_registry.py:33-56`)

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| RSS URLs are wrong/stale | No articles ingested | Verify each URL manually before Sprint 1 is done |
| Entity names don't match extracted entities | Pillar digests have 0 stories | Check `story_entities` table for actual entity names after ingestion runs |
| LLM sentiment prompt gives inconsistent results | Bad sentiment labels | Test prompt with 20+ articles, iterate on prompt wording |
| fpdf2 can't handle Unicode/special chars | PDF rendering breaks | Test with Guyanese proper nouns, special characters early |
| Cloud Function timeout (60s) | Digest generation fails | Pre-compute via background refresh; on-demand is fallback only |
| pgvector RPC too slow with source join | Query timeout | Add index on `raw_items(source_id)` if not exists; test with EXPLAIN ANALYZE |

---

## Definition of Done

The War Room digest engine is **complete** when:

1. All 5 pillars visible in `/api/warroom/pillars`
2. Clicking any pillar returns a digest with stories, quotes, and sentiment
3. Digests auto-refresh every 3 hours via Cloud Scheduler
4. User can download digest as Word (.docx) or PDF
5. Document format matches the template in `06_DOCUMENT_EXPORT.md`
6. War Room frontend successfully integrated with all endpoints
7. Response time < 500ms for cached digests, < 15s for fresh generation
8. All endpoints have auth, rate limiting, and error handling
