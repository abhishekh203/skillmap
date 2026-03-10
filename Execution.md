# 08 — Sources & Entities Mapping

> Complete mapping of War Room sources and entities from the UI screenshots to existing database structures.

---

## Sources (from War Room "All Sources" Dropdown)

These are the news sources visible in the War Room UI (Domestic Guyana + International). Each needs a row in the `sources` table.

### Domestic Guyana (Live: Guyana)

| # | Source Name | Type | RSS URL (to be configured) | Status |
|---|------------|------|---------------------------|--------|
| 1 | Stabroek News | RSS | `https://www.stabroeknews.com/feed/` | Needs verification |
| 2 | Kaieteur News | RSS | `https://www.kaieteurnewsonline.com/feed/` | Needs verification |
| 3 | Guyana Chronicle | RSS | `https://guyanachronicle.com/feed/` | Needs verification |
| 4 | Demerara Waves | RSS | `https://demerarawaves.com/feed/` | Needs verification |
| 5 | News Room GY | RSS | `https://newsroom.gy/feed/` | Needs verification |
| 6 | Guyana Times | RSS | `https://guyanatimesgy.com/feed/` | Needs verification |
| 7 | Conversation Tree | RSS | TBD | Needs research |
| 8 | Guyana News & Information | RSS | TBD | Needs research |
| 9 | DPI Guyana (Dept. of Public Information) | RSS | `https://dpi.gov.gy/feed/` | Needs verification |
| 10 | Guyana Defence Force | RSS | TBD | Needs research |
| 11 | Guyana Revenue Authority (GRA) | RSS | TBD | Needs research (may not have RSS) |
| 12 | Caribbean360 | RSS | `https://www.caribbean360.com/feed` | Needs verification |

### International & Regional (Foreign: Guyana / broader coverage)

| # | Source Name | Type | RSS URL (to be configured) | Status |
|---|------------|------|---------------------------|--------|
| 13 | Caribbean National Weekly | RSS | TBD | Needs research |
| 14 | Jamaica Gleaner | RSS | TBD | Needs research |
| 15 | T&T Guardian | RSS | TBD | Needs research |
| 16 | Reuters | RSS | TBD | Needs research |
| 17 | BBC News | RSS | TBD | Needs research |
| 18 | Al Jazeera | RSS | TBD | Needs research |
| 19 | The Guardian | RSS | TBD | Needs research |
| 20 | NYT Americas | RSS | TBD | Needs research |
| 21 | OilNow | RSS | TBD | Needs research |
| 22 | Offshore Energy | RSS | TBD | Needs research |
| 23 | Energy Voice | RSS | TBD | Needs research |

### Notes
- All RSS URLs need verification — some sites may have changed feed URLs
- Sources without RSS may need web scraping (the `news_article_processor_worker` supports this via `source_type` config)
- The existing worker handles rate limiting (500ms between requests to same domain)

### Database Insert Format

```sql
-- Example: Add Stabrook News
INSERT INTO sources (name, source_type, description, config, active, platform)
VALUES (
    'Stabrook News',
    'rss',
    'Guyana domestic news source',
    '{"rss_url": "https://www.stabroeknews.com/feed/", "scrape_content": true}'::jsonb,
    true,
    'warroom'
);
```

### Source → Pillar Mapping

| Source | Pillar 1 (Ali) | Pillar 2 (Jagdeo) | Pillar 3 (Azruddin) | Pillar 4 (Opposition) | Pillar 5 (Live+Intl) |
|--------|:-:|:-:|:-:|:-:|:-:|
| Stabrook News | via entity | via entity | via entity | via entity | YES |
| Kaieteur News | via entity | via entity | via entity | via entity | YES |
| Guyana Chronicle | via entity | via entity | via entity | via entity | YES |
| Demerara Waves | via entity | via entity | via entity | via entity | YES |
| News Room GY | via entity | via entity | via entity | via entity | YES |
| Guyana Times | via entity | via entity | via entity | via entity | YES |
| Conversation Tree | via entity | via entity | via entity | via entity | YES |
| Guyana News & Info | via entity | via entity | via entity | via entity | YES |
| DPI Guyana | via entity | via entity | via entity | via entity | YES |
| Guyana Defence Force | via entity | via entity | via entity | via entity | YES |
| Guyana Revenue Auth | via entity | via entity | via entity | via entity | YES |
| Caribbean360 | via entity | via entity | via entity | via entity | YES (Intl) |

**Key**:
- "via entity" = articles from this source are included if they mention the pillar's entities
- "YES" = all articles from this source are included (Pillar 5 is source-based, not entity-based)

---

## Entities (from War Room "All Entities" Dropdown)

These are the entities visible in the War Room UI. They map to `story_entities.entity_name`.

### People

| Entity | Type | Pillar | Aliases |
|--------|------|--------|---------|
| Irfaan Ali | PERSON | 1 (President Ali) | President Ali, Dr. Ali, Dr. Irfaan Ali, Mohamed Irfaan Ali |
| Bharrat Jagdeo | PERSON | 2 (VP Jagdeo) | VP Jagdeo, Vice President Jagdeo, Dr. Jagdeo |
| Aubrey Norton | PERSON | 4 (Opposition) | Leader of the Opposition, Norton |
| Azruddin Mohamed | PERSON | 3 (Azruddin) | Mohamed Azruddin |

### Organizations

| Entity | Type | Pillar | Aliases |
|--------|------|--------|---------|
| PPP/C | ORG | — | People's Progressive Party, PPP, PPP Civic |
| APNU+AFC | ORG | 4 (Opposition) | APNU, AFC, PNC, A Partnership for National Unity, Alliance for Change |
| ExxonMobil Guyana | ORG | — | ExxonMobil, Exxon, Esso Exploration |
| GECOM | ORG | — | Guyana Elections Commission |

### Locations

| Entity | Type | Pillar | Notes |
|--------|------|--------|-------|
| Georgetown | LOC | — | Capital city |
| Linden | LOC | — | Second largest town |
| Region 4 | LOC | — | Most populated region |
| Stabroek Market | LOC | — | Landmark/commercial area |

### How Entities Flow Through the System

```
Article ingested (news_article_processor_worker)
    ↓
Article clustered into story (story_clustering_worker)
    ↓
Entities extracted → stored in story_entities table
    ↓
Pillar digest generation queries story_entities
    ↓
match_stories_for_pillar() RPC:
    - Filters stories where story_entities overlaps with pillar.entity_filters
    - Boosts ranking for stories with more entity matches
```

### Entity Extraction Quality

The current system extracts entities during story clustering. Quality depends on:
1. **NER accuracy** — proper nouns like "Irfaan Ali" are well-detected
2. **Alias coverage** — "President Ali" vs "Dr. Ali" may be stored as different entities
3. **Entity normalization** — currently no deduplication of aliases

### Recommendation: Entity Alias Resolution

For War Room, the simplest approach is to include ALL aliases in the pillar's `entity_filters` array:

```python
# In pillar_configs table:
entity_filters = [
    'Irfaan Ali',        # Exact name
    'President Ali',     # Title + last name
    'Dr. Ali',           # Honorific + last name
    'Dr. Irfaan Ali',    # Full with honorific
    'Mohamed Irfaan Ali' # Full legal name
]
```

The `match_stories_for_pillar` RPC checks `story_entities.entity_name = ANY(pillar_entities)`, so including all aliases ensures comprehensive matching without needing a separate alias resolution system.

---

## Live Feed Counts (from UI Screenshots)

The UI shows article counts per source feed:

| Source | Count (at time of screenshot) |
|--------|------|
| Stabrook News | 25 |
| Kaieteur News | 25 |
| Guyana Times | 10 |
| Dept. of Public Information (DPI) | 5 |
| Conversation Tree | 10 |
| News Room Guyana | 10 |
| Guyana Chronicle | 10 |
| Guyana Revenue Authority (GRA) | 10 |
| Guyana Defence Force | visible |

This indicates active RSS feeds with regular publishing — good signal that the ingestion pipeline will have steady input.

---

## Toolbar Features Mapping (from UI Screenshots)

The War Room UI toolbar shows: `CSV | EMAIL | BRIEF | NARRATIVES`

| Feature | Backend Support | Notes |
|---------|----------------|-------|
| CSV | NEW — `export_service.py` | Simple tabular export |
| EMAIL | NEW — SendGrid integration | See [06_DOCUMENT_EXPORT.md](./06_DOCUMENT_EXPORT.md) |
| BRIEF | REUSE — `create_detailed_story_prompt()` | Existing detailed report generation |
| NARRATIVES | REUSE — `create_opposite_sides_prompt()` | Existing debate/narrative generation |

---

## Filter Dropdowns → Backend Queries

| UI Filter | Backend Implementation |
|-----------|----------------------|
| All Topics | `story_categories` table — filter by category slug |
| All Tones | NEW — sentiment field (not yet in DB, needs Phase 7) |
| All (time range: 1H, 6H, 24H, 7D, Custom) | `stories.last_article_date > NOW() - interval` |
| All Sources | `story_articles → artifacts → raw_items → sources` join |
| All Entities | `story_entities.entity_name` filter |
| All Types | TBD — may map to `story_categories` or a new type field |

---

## Data Volume Estimates

Based on 12 sources publishing ~10-25 articles each per day:

| Metric | Estimate |
|--------|----------|
| Articles/day | 120-300 |
| Stories/day (after clustering) | 30-80 |
| Stories per pillar digest | 5-15 |
| Entities per story | 3-8 |
| Storage per day | ~5-10 MB (articles in GCS) |

This is well within the existing system's capacity (designed for thousands of articles/day).
