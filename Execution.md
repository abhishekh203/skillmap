# 03 — New Tables & Database Migrations

> Database changes needed to support War Room pillars, digest generation, and document export.

---

## Overview

The existing schema (42 migrations) covers articles, stories, entities, newsletters, and user profiles. War Room needs **3 new tables** and **1 new RPC function**.

---

## New Table 1: `pillar_configs`

Stores the configuration for each intelligence pillar.

```sql
-- Migration: 042_create_pillar_configs.sql

CREATE TABLE IF NOT EXISTS pillar_configs (
    id SERIAL PRIMARY KEY,

    -- Identity
    slug TEXT NOT NULL UNIQUE,          -- e.g., 'ali-digest', 'opposition-digest'
    name TEXT NOT NULL,                  -- e.g., 'President Ali Digest'
    description TEXT,                    -- Used to generate pillar embedding
    icon TEXT,                           -- Icon identifier for frontend
    sort_order INTEGER DEFAULT 0,       -- Display order in sidebar

    -- Entity Filters
    -- All entity names + aliases that should match for this pillar
    entity_filters TEXT[] DEFAULT '{}',  -- e.g., {'Irfaan Ali', 'President Ali', 'Dr. Ali'}

    -- Source Filters (optional — NULL means all sources)
    -- Array of source IDs from the sources table
    source_ids INTEGER[],               -- NULL = all sources, [1,2,3] = specific sources only

    -- Keyword Filters (optional — for additional text matching)
    keywords TEXT[] DEFAULT '{}',        -- e.g., {'president', 'Ali', 'head of state'}

    -- Semantic Matching
    -- Pre-generated embedding from pillar description (768-dim, Gemini embedding-001)
    pillar_embedding vector(768),

    -- Generation Settings
    max_stories INTEGER DEFAULT 10,      -- Max stories per digest (brief says 5-10)
    hours_back INTEGER DEFAULT 24,       -- How far back to search for stories
    refresh_interval_hours INTEGER DEFAULT 3,  -- How often to auto-regenerate

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    platform TEXT DEFAULT 'warroom',     -- 'warroom', 'civic_portal', etc.

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for quick lookups
CREATE INDEX idx_pillar_configs_slug ON pillar_configs(slug);
CREATE INDEX idx_pillar_configs_active ON pillar_configs(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_pillar_configs_platform ON pillar_configs(platform);

-- Auto-update timestamp trigger
CREATE TRIGGER update_pillar_configs_updated_at
    BEFORE UPDATE ON pillar_configs
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Seed Data

```sql
-- Migration: 043_seed_pillar_configs.sql

INSERT INTO pillar_configs (slug, name, description, entity_filters, keywords, sort_order, platform)
VALUES
    (
        'ali-digest',
        'President Ali',
        'News, statements, and coverage of the President of Guyana, Irfaan Ali',
        ARRAY['Irfaan Ali', 'President Ali', 'Dr. Ali', 'Dr. Irfaan Ali', 'Mohamed Irfaan Ali'],
        ARRAY['president', 'head of state', 'office of the president'],
        1,
        'warroom'
    ),
    (
        'jagdeo-digest',
        'VP Jagdeo',
        'News, statements, and coverage of Vice President Bharrat Jagdeo',
        ARRAY['Bharrat Jagdeo', 'VP Jagdeo', 'Vice President Jagdeo', 'Dr. Jagdeo'],
        ARRAY['vice president', 'VP'],
        2,
        'warroom'
    ),
    (
        'azruddin-digest',
        'Azruddin Mohamed',
        'Entity-filtered feed tracking political figure Azruddin Mohamed',
        ARRAY['Azruddin Mohamed', 'Mohamed Azruddin'],
        ARRAY['Azruddin'],
        3,
        'warroom'
    ),
    (
        'opposition-digest',
        'Opposition',
        'Aggregated coverage of all opposition channels and figures in Guyana',
        ARRAY['Aubrey Norton', 'APNU+AFC', 'APNU', 'AFC', 'PNC',
              'People''s National Congress', 'A Partnership for National Unity',
              'Alliance for Change', 'Leader of the Opposition',
              'David Granger', 'Khemraj Ramjattan'],
        ARRAY['opposition', 'shadow cabinet'],
        4,
        'warroom'
    ),
    (
        'live-guyana-digest',
        'Live Guyana News + International',
        'Domestic breaking news and foreign coverage of Guyana',
        ARRAY[]::TEXT[],  -- No entity filter — source-based pillar
        ARRAY['Guyana', 'Georgetown', 'breaking'],
        5,
        'warroom'
    );

-- NOTE: source_ids for 'live-guyana-digest' should be populated after
-- War Room RSS sources are added to the sources table.
-- Run: UPDATE pillar_configs SET source_ids = ARRAY[...] WHERE slug = 'live-guyana-digest';

-- NOTE: pillar_embedding should be generated after creation using:
-- generate_profile_embedding(description) from llm_service.py
-- This can be done via a one-time script or admin endpoint.
```

---

## New Table 2: `pillar_digests`

Stores generated digest content for each pillar, per date/time.

```sql
-- Migration: 044_create_pillar_digests.sql

CREATE TABLE IF NOT EXISTS pillar_digests (
    id BIGSERIAL PRIMARY KEY,

    -- Reference
    pillar_id INTEGER NOT NULL REFERENCES pillar_configs(id) ON DELETE CASCADE,

    -- Timing & Expiration (Mirroring user_daily_newsletters)
    edition_date DATE NOT NULL,
    generated_at TIMESTAMPTZ NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NULL,

    -- Content & Story References
    summary_story_ids BIGINT[] NOT NULL DEFAULT '{}'::BIGINT[],
    ai_semantic_story_ids BIGINT[] NOT NULL DEFAULT '{}'::BIGINT[],
    total_stories_available INTEGER NULL DEFAULT 0,

    -- The actual compiled digest content
    -- (Replacing cached_llm_summary since digests need structured JSON)
    content JSONB NOT NULL DEFAULT '{}'::JSONB,

    -- Generation metadata
    generation_method TEXT NULL DEFAULT 'ai_semantic'::TEXT,
    generation_time_ms INTEGER,

    -- Status & Errors
    status TEXT DEFAULT 'active'::TEXT,
    error_message TEXT,

    -- Timestamps
    created_at TIMESTAMPTZ NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NULL DEFAULT NOW(),

    CONSTRAINT pillar_digests_multi_unique UNIQUE (pillar_id, edition_date)
);

-- Indexes (Mirroring user_daily_newsletters indexing strategy)
CREATE INDEX IF NOT EXISTS idx_pillar_digests_generated_at ON pillar_digests USING btree (generated_at DESC);
CREATE INDEX IF NOT EXISTS idx_pillar_digests_pillar_date ON pillar_digests USING btree (pillar_id, edition_date DESC);
CREATE INDEX IF NOT EXISTS idx_pillar_digests_date ON pillar_digests USING btree (edition_date DESC);
CREATE INDEX IF NOT EXISTS idx_pillar_digests_expires ON pillar_digests USING btree (expires_at) WHERE (expires_at IS NOT NULL);
CREATE INDEX IF NOT EXISTS idx_pillar_digests_status ON pillar_digests USING btree (status) WHERE status = 'active';

-- Auto-update timestamp trigger
CREATE TRIGGER trg_pillar_digests_updated_at
    BEFORE UPDATE ON pillar_digests
    FOR EACH ROW
    EXECUTE FUNCTION trigger_set_timestamp();
```


---

## New Table 3: `pillar_source_mapping` (Optional)

If pillars need many-to-many source relationships beyond what `source_ids[]` array provides.

```sql
-- Migration: 045_create_pillar_source_mapping.sql (OPTIONAL)
-- Only needed if source management per pillar becomes complex

CREATE TABLE IF NOT EXISTS pillar_source_mapping (
    id SERIAL PRIMARY KEY,
    pillar_id INTEGER NOT NULL REFERENCES pillar_configs(id) ON DELETE CASCADE,
    source_id INTEGER NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    priority INTEGER DEFAULT 0,            -- Higher = more important for this pillar
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(pillar_id, source_id)
);

CREATE INDEX idx_pillar_source_mapping_pillar ON pillar_source_mapping(pillar_id) WHERE is_active = TRUE;
```

---

## New RPC Function: `match_stories_for_pillar`

Extends existing `match_stories_for_newsletter` (migration `030`) with source-level filtering.

```sql
-- Migration: 046_create_match_stories_for_pillar.sql

CREATE OR REPLACE FUNCTION match_stories_for_pillar(
    query_embedding vector(768),
    pillar_entities TEXT[] DEFAULT '{}',
    pillar_source_ids INTEGER[] DEFAULT '{}',
    match_threshold FLOAT DEFAULT 0.4,
    match_count INTEGER DEFAULT 50,
    hours_back INTEGER DEFAULT 24
)
RETURNS TABLE (
    id INTEGER,
    title TEXT,
    summary TEXT,
    article_count INTEGER,
    created_at TIMESTAMPTZ,
    last_article_date TIMESTAMPTZ,
    similarity FLOAT,
    entity_match_count INTEGER,
    matched_entities TEXT[]
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    WITH
    -- Step 1: Semantic similarity search (existing pattern from match_stories_for_newsletter)
    semantic_matches AS (
        SELECT
            s.id,
            s.title,
            s.summary,
            s.article_count,
            s.created_at,
            s.last_article_date,
            1 - (s.centroid_embedding_vec <=> query_embedding) AS similarity
        FROM stories s
        WHERE s.status = 'ACTIVE'
          AND s.last_article_date > NOW() - (hours_back || ' hours')::INTERVAL
          AND s.centroid_embedding_vec IS NOT NULL
          AND 1 - (s.centroid_embedding_vec <=> query_embedding) > match_threshold
        ORDER BY s.centroid_embedding_vec <=> query_embedding
        LIMIT match_count * 3  -- Overfetch for filtering
    ),

    -- Step 2: Source filter (only if pillar_source_ids is not empty)
    source_filtered AS (
        SELECT sm.*
        FROM semantic_matches sm
        WHERE
            -- If no source filter, keep all
            array_length(pillar_source_ids, 1) IS NULL
            OR
            -- Otherwise, check if story has articles from specified sources
            EXISTS (
                SELECT 1
                FROM story_articles sa
                JOIN artifacts a ON a.id = sa.artifact_id
                JOIN raw_items ri ON ri.id = a.raw_item_id
                WHERE sa.story_id = sm.id
                  AND ri.source_id = ANY(pillar_source_ids)
            )
    ),

    -- Step 3: Entity matching and scoring (existing pattern from reranked function)
    entity_scored AS (
        SELECT
            sf.*,
            COALESCE(entity_agg.match_count, 0) AS entity_match_count,
            COALESCE(entity_agg.matched, ARRAY[]::TEXT[]) AS matched_entities
        FROM source_filtered sf
        LEFT JOIN LATERAL (
            SELECT
                COUNT(*)::INTEGER AS match_count,
                ARRAY_AGG(DISTINCT se.entity_name) AS matched
            FROM story_entities se
            WHERE se.story_id = sf.id
              AND se.entity_name = ANY(pillar_entities)
        ) entity_agg ON TRUE
    )

    -- Step 4: Final scoring and ranking
    SELECT
        es.id,
        es.title,
        es.summary,
        es.article_count,
        es.created_at,
        es.last_article_date,
        -- Combined score: semantic similarity (70%) + entity match bonus (30%)
        CASE
            WHEN array_length(pillar_entities, 1) IS NULL OR array_length(pillar_entities, 1) = 0
            THEN es.similarity  -- No entity filter → pure semantic
            ELSE es.similarity * 0.7 + LEAST(es.entity_match_count::FLOAT / 3.0, 1.0) * 0.3
        END AS similarity,
        es.entity_match_count,
        es.matched_entities
    FROM entity_scored es
    -- For entity-based pillars, require at least one entity match
    WHERE
        array_length(pillar_entities, 1) IS NULL
        OR array_length(pillar_entities, 1) = 0
        OR es.entity_match_count > 0
    ORDER BY similarity DESC
    LIMIT match_count;
END;
$$;
```

---

## Column Addition to `sources` Table (Optional)

To tag sources with their platform:

```sql
-- Migration: 047_add_platform_to_sources.sql (OPTIONAL)

ALTER TABLE sources ADD COLUMN IF NOT EXISTS platform TEXT DEFAULT 'nonews';
-- Values: 'nonews', 'warroom', 'both'

CREATE INDEX idx_sources_platform ON sources(platform);

-- Tag existing War Room sources after they're added:
-- UPDATE sources SET platform = 'warroom' WHERE name IN ('Stabrook News', 'Kaieteur News', ...);
```

---

## Migration Execution Order

```
042_create_pillar_configs.sql          -- Pillar configuration table
043_seed_pillar_configs.sql            -- Initial 5 pillars
044_create_pillar_digests.sql          -- Digest storage
045_create_pillar_source_mapping.sql   -- Optional: source ↔ pillar mapping
046_create_match_stories_for_pillar.sql -- RPC for pillar story matching
047_add_platform_to_sources.sql        -- Optional: platform tagging
```

---

## Post-Migration Steps

1. **Add War Room RSS sources** to `sources` table (if not already present)
2. **Generate pillar embeddings** by running `generate_profile_embedding(description)` for each pillar
3. **Set `source_ids`** on `live-guyana-digest` pillar after sources are created
4. **Verify** `match_stories_for_pillar` returns results with test queries

---

## Existing Tables Used (No Changes Needed)

| Table | Usage in War Room |
|---|---|
| `sources` | Store War Room RSS feed URLs |
| `raw_items` | Ingested articles |
| `artifacts` | Processed articles with embeddings |
| `stories` | Clustered story groups |
| `story_articles` | Story ↔ article junction |
| `story_entities` | Extracted entities per story |
| `story_categories` | Category assignments |
| `story_ai_content` | LLM-generated content (quotes, reports, etc.) |
