# 01 — Backend Reuse Map

> Component-by-component analysis of what can be reused from NoNewsBackend for War Room.

---

## Overall Reuse Score: ~70-75%

---

## 1. RSS Ingestion Pipeline — 95% Reusable

### Existing Component
- **File**: `cloud_functions/news_article_processor_worker/main.py`
- **What it does**: Fetches RSS feeds, extracts article content (newspaper3k + trafilatura fallback), stores in GCS, deduplicates via content hash, publishes to Pub/Sub

### War Room Mapping
- War Room sources (Stabrook News, Kaieteur News, etc.) are RSS feeds — **identical use case**
- Each War Room source just needs a row in the `sources` table with its RSS URL
- Content extraction, GCS upload, deduplication — all work as-is

### Changes Needed
- **None for the worker itself**
- Just add War Room RSS source URLs to the `sources` table (admin/migration)
- Optionally: tag sources with a `platform` field to distinguish War Room vs No News sources (see [03_NEW_TABLES_AND_MIGRATIONS.md](./03_NEW_TABLES_AND_MIGRATIONS.md))

---

## 2. Story Clustering — 90% Reusable

### Existing Component
- **File**: `cloud_functions/story_clustering_worker/main.py`
- **What it does**: Groups articles into stories using pgvector similarity (0.75-0.77 threshold) + entity Jaccard overlap (10%+)
- **Embedding**: Gemini embedding-001, 768 dimensions, HNSW index

### War Room Mapping
- War Room needs articles grouped into stories — **exact same requirement**
- Entity overlap validation catches Guyana-specific entities (Ali, Jagdeo, APNU+AFC) — works perfectly
- Dynamic thresholds (Day 1: 0.75, Day 2: 0.77) — appropriate for fast-moving political news

### Changes Needed
- **None** — clustering is entity-agnostic and embedding-based, works on any domain
- Consider: May want slightly lower thresholds for political news (stories diverge faster) — **tunable later**

---

## 3. Story Merge Processor — 100% Reusable

### Existing Component
- **File**: `cloud_functions/story_merge_processor/main.py`
- **What it does**: Runs every 30 min, finds similar stories missed by real-time clustering, merges them (0.85 threshold)

### War Room Mapping
- Identical need — prevents duplicate stories about same political event
- No changes needed

---

## 4. Entity Extraction — 85% Reusable

### Existing Component
- **Table**: `story_entities` (entity_name, entity_type, frequency, relevance)
- **Extracted during**: Story clustering worker
- **Used by**: Newsletter matching (entity reranking in `search_stories_semantic_reranked`)

### War Room Mapping
- War Room entities from screenshots: Irfaan Ali, Bharrat Jagdeo, Aubrey Norton, Azruddin Mohamed, PPP/C, APNU+AFC, ExxonMobil Guyana, GECOM, Georgetown, Linden, Region 4, Stabroek Market
- These will naturally appear in `story_entities` as articles are processed
- Entity matching is string-based — handles proper nouns well

### Changes Needed
- **Entity normalization**: May need aliases (e.g., "President Ali" = "Irfaan Ali" = "Dr. Ali")
- Add a `pillar_entity_aliases` mapping table or config (see [03_NEW_TABLES_AND_MIGRATIONS.md](./03_NEW_TABLES_AND_MIGRATIONS.md))

---

## 5. LLM Service — 90% Reusable

### Existing Component
- **File**: `cloud_functions/api_service/llm_service.py`
- **Model**: OpenRouter → Gemini 2.5 Flash Lite
- **Functions**: `generate_with_gemini()`, `generate_profile_embedding()`, 7 prompt types

### War Room Mapping
The following existing prompts directly serve War Room digest needs:

| War Room Digest Section | Existing LLM Prompt | File Reference |
|---|---|---|
| Top Stories (2-3 sentence summary) | `create_concise_prompt()` | llm_service.py |
| Key Quotes | `create_quotes_with_sources_prompt()` + `parse_quotes_with_sources()` | llm_service.py |
| Detailed Report | `create_detailed_story_prompt()` | llm_service.py |
| Suggested Questions | `create_suggested_questions_prompt()` | llm_service.py |

### Changes Needed (New Prompts)
- **Sentiment Analysis prompt**: New — "Analyze overall tone: positive/neutral/negative" (see [05_DIGEST_GENERATION_ENGINE.md](./05_DIGEST_GENERATION_ENGINE.md))
- **Digest Introduction prompt**: New — brief AI-generated overview paragraph for each pillar digest
- **Max tokens**: May need to increase from 1200 for digest-level summaries

---

## 6. Newsletter System — 80% Reusable (KEY COMPONENT)

### Existing Component
- **File**: `cloud_functions/api_service/newsletter_service.py` (97KB)
- **Core method**: `generate_user_newsletter()` — takes `profile_embedding`, `profile_entities`, `profile_categories`, searches stories via pgvector, deduplicates, stores results

### War Room Mapping
**Each War Room pillar IS a newsletter** with pre-configured parameters:

| Newsletter Concept | War Room Equivalent |
|---|---|
| `user_newsletters` table | `pillars` config table |
| `profile_embedding` | Pillar description embedding |
| `profile_entities` | Pillar entity filter list |
| `newsletter_slug` | Pillar slug (e.g., `ali-digest`) |
| `generate_user_newsletter()` | Digest generation per pillar |
| `generate-daily-newsletters` batch endpoint | Background pillar digest refresh (Cloud Scheduler) |

### What Works As-Is
- `search_stories_semantic_reranked()` — pgvector + entity reranking + category filtering
- Story deduplication (milestone-based)
- Batch generation with ThreadPoolExecutor
- Newsletter storage in `user_daily_newsletters`

### Changes Needed
- New `pillar_configs` table with hardcoded entity/source filters per pillar
- New `generate_pillar_digest()` method that wraps `generate_user_newsletter()` with pillar-specific params
- Source-level filtering (current system filters by entity/category but not by specific source IDs)
- See [02_PILLAR_NEWSLETTER_MAPPING.md](./02_PILLAR_NEWSLETTER_MAPPING.md) for full details

---

## 7. Database Layer — 80% Reusable

### Existing Component
- **File**: `cloud_functions/api_service/database.py` (79KB)
- **Classes**: StoryDatabase, NewsletterDatabase, SubscriptionDatabase, TemplateDatabase, AnalyticsDatabase, StoryAIContentDatabase

### War Room Mapping
- `StoryDatabase.get_story_by_id()`, `fetch_stories_by_ids()`, `search_stories_semantic_reranked()` — all reusable
- `StoryDatabase.get_source_urls_for_story()` — needed for digest source attribution
- `NewsletterDatabase` CRUD operations — reusable for pillar digest storage

### Changes Needed
- New `PillarDatabase` class or extend `NewsletterDatabase` with pillar-specific queries
- New RPC function: `match_stories_for_pillar()` — like `match_stories_for_newsletter` but with source_id filtering
- See [03_NEW_TABLES_AND_MIGRATIONS.md](./03_NEW_TABLES_AND_MIGRATIONS.md)

---

## 8. API Service (Flask) — 70% Reusable

### Existing Component
- **File**: `cloud_functions/api_service/main.py` + route files
- **Infrastructure**: Flask, Blueprints, CORS, rate limiting, JWT auth, error handling, health check

### War Room Mapping
- Flask app structure, error handling, auth — fully reusable
- Rate limiting config — reusable
- Service registry pattern — reusable
- CORS — need to add War Room domain (`v0-guyana-news-war-room.vercel.app` + production domain)

### Changes Needed
- New Blueprint: `routes_warroom.py` or `routes_digests.py` for pillar digest endpoints
- Add War Room CORS origins to `config.py`
- New endpoints (see [04_API_ENDPOINTS.md](./04_API_ENDPOINTS.md))

---

## 9. Authentication — 100% Reusable

### Existing Component
- **File**: `cloud_functions/api_service/utils_auth.py`
- JWT verification via Supabase, `@require_auth` decorator

### War Room Mapping
- Same Supabase project = same auth
- Same JWT verification
- No changes needed

---

## 10. Validation & Error Handling — 100% Reusable

### Existing Components
- `utils_validation.py` — input validation, sanitization
- `utils_errors.py` — error hierarchy, logging
- `utils_helpers.py` — CORS, logging, response formatting

### War Room Mapping
- All reusable without modification

---

## 11. Export / Document Generation — 0% (NEW)

### Existing
- **No export functionality exists** in the current codebase
- No Word, PDF, CSV (despite CSV button in War Room UI), or email export

### War Room Needs
- Word (.docx) generation
- PDF generation
- Email delivery
- CSV export (seen in screenshot toolbar)

### Must Build From Scratch
- See [06_DOCUMENT_EXPORT.md](./06_DOCUMENT_EXPORT.md) for full implementation details

---

## 12. Sentiment Analysis — 0% (NEW)

### Existing
- No sentiment analysis in current codebase
- The War Room UI shows sentiment tags (FAVORABLE, NEUTRAL, LOW/MED) but these aren't generated by this backend

### War Room Needs
- Per-article sentiment (already shown in UI — may come from frontend or separate service)
- Per-digest overall sentiment note

### Must Build
- New LLM prompt for sentiment classification
- Simple: "Classify this content as positive/neutral/negative toward [entity]"
- See [05_DIGEST_GENERATION_ENGINE.md](./05_DIGEST_GENERATION_ENGINE.md)

---

## Summary Table

| Component | Reuse % | Changes Needed |
|---|---|---|
| RSS Ingestion Worker | 95% | Add War Room source URLs to DB |
| Story Clustering Worker | 90% | None (maybe tune thresholds later) |
| Story Merge Processor | 100% | None |
| Entity Extraction | 85% | Add entity alias mapping |
| LLM Service | 90% | Add sentiment + digest intro prompts |
| Newsletter Service | 80% | Add pillar-specific generation wrapper |
| Database Layer | 80% | Add pillar tables + source-filtered RPC |
| API Service (Flask) | 70% | Add War Room routes + CORS |
| Authentication | 100% | None |
| Validation/Errors | 100% | None |
| Document Export | 0% | Build from scratch |
| Sentiment Analysis | 0% | Build from scratch |
| **Overall** | **~70-75%** | |
