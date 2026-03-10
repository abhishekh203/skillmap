# 05 — Digest Generation Engine

> Detailed implementation of the one-click digest engine — the core War Room feature.

---

## Architecture Overview

```
User clicks "Ali Digest" button
         ↓
Frontend: POST /api/warroom/digests/generate/ali-digest
         ↓
Backend: digest_service.generate_pillar_digest()
         ↓
    ┌─────────────────────┐
    │ 1. Load pillar config│
    │    from pillar_configs│
    └──────────┬──────────┘
               ↓
    ┌─────────────────────────────┐
    │ 2. Check for recent digest   │
    │    (within refresh_interval) │
    │    → If fresh, return cached │
    └──────────┬──────────────────┘
               ↓
    ┌──────────────────────────────────┐
    │ 3. RPC: match_stories_for_pillar │
    │    - pgvector semantic search     │
    │    - Entity filtering/reranking   │
    │    - Source-level filtering        │
    └──────────┬───────────────────────┘
               ↓
    ┌──────────────────────────────────┐
    │ 4. Story enrichment (per story)  │
    │    a. Fetch article content (GCS)│
    │    b. Concise summary (LLM)      │
    │    c. Quote extraction (LLM)     │
    │    d. Sentiment analysis (LLM)   │
    └──────────┬───────────────────────┘
               ↓
    ┌──────────────────────────────────┐
    │ 5. Digest-level content          │
    │    a. Introduction paragraph     │
    │    b. Overall sentiment note     │
    └──────────┬───────────────────────┘
               ↓
    ┌──────────────────────────────┐
    │ 6. Store in pillar_digests   │
    │    → Return to frontend      │
    └──────────────────────────────┘
```

---

## New File: `digest_service.py`

Location: `cloud_functions/api_service/digest_service.py`

```python
"""
Digest Service — One-Click Digest Generation for War Room Pillars

Reuses:
- story_service.py: get_story_data(), GCS content extraction
- llm_service.py: generate_with_gemini(), create_concise_prompt(),
                   create_quotes_with_sources_prompt()
- database.py: StoryDatabase.match_stories_for_pillar() (new RPC)
- newsletter_service.py: deduplication patterns
"""

class DigestService:
    def __init__(self, http_session, pillar_db, story_db, story_service, story_ai_db):
        self.http_session = http_session
        self.pillar_db = pillar_db
        self.story_db = story_db
        self.story_service = story_service
        self.story_ai_db = story_ai_db

    def generate_pillar_digest(self, pillar_slug, force=False):
        """Main entry point — generates a complete digest for one pillar."""
        ...

    def _search_pillar_stories(self, pillar):
        """Call match_stories_for_pillar RPC."""
        ...

    def _enrich_story(self, story):
        """Add summary, quotes, sentiment to a single story."""
        ...

    def _generate_digest_content(self, pillar, enriched_stories):
        """Compile the full digest JSON structure."""
        ...

    def _analyze_sentiment(self, text, entity_name=None):
        """Classify sentiment of text toward an entity."""
        ...

    def _generate_digest_intro(self, pillar, stories):
        """Generate the digest introduction paragraph."""
        ...

    def refresh_all_pillars(self, pillar_slugs=None, force=False):
        """Background refresh — called by Cloud Scheduler."""
        ...
```

---

## Step-by-Step Implementation

### Step 1: Load Pillar Config

```python
def generate_pillar_digest(self, pillar_slug, force=False):
    pillar = self.pillar_db.get_by_slug(pillar_slug)
    if not pillar:
        raise ResourceNotFoundError(f"Pillar '{pillar_slug}' not found")

    # Check for recent digest (cache hit)
    if not force:
        latest = self.pillar_db.get_latest_digest(pillar['id'])
        if latest and is_fresh(latest, pillar['refresh_interval_hours']):
            return latest

    # Continue to generation...
```

### Step 2: Search Stories

Reuses the pattern from `newsletter_service.py:472-480` but calls the new RPC.

```python
def _search_pillar_stories(self, pillar):
    # All pillars search ALL War Room sources — no source_ids restriction
    # Entity-based pillars (Opposition, Azruddin, Ali & Jagdeo): filter by entity_filters
    # Semantic-based pillars (Live: Guyana, Foreign: Guyana): pure embedding match
    stories_data = self.story_db.match_stories_for_pillar(
        query_embedding=pillar['pillar_embedding'],
        pillar_entities=pillar['entity_filters'] or [],
        pillar_source_ids=[],  # Empty = all sources
        match_threshold=0.4,
        match_count=pillar['max_stories'] or 10,
        hours_back=pillar['hours_back'] or 24
    )
    return stories_data
```

### Step 3: Enrich Each Story

Reuses existing `story_service.py` methods:

```python
def _enrich_story(self, story):
    # Reuse: fetch article content from GCS
    # (same as story_service.py:get_story_data + extract_article_content)
    story_data = self.story_service.get_story_data(story['id'])
    article_content = self.story_service.extract_article_content(story_data)

    # Reuse: concise summary (llm_service.py:create_concise_prompt)
    summary = self.story_service.generate_summary(
        story_id=story['id'],
        summary_type='concise'
    )

    # Reuse: quote extraction (llm_service.py:create_quotes_with_sources_prompt)
    quotes = self.story_service.generate_summary(
        story_id=story['id'],
        summary_type='quotes'
    )

    # NEW: per-story sentiment
    sentiment = self._analyze_sentiment(
        text=article_content,
        entity_name=None  # Or pillar's primary entity
    )

    return {
        'story_id': story['id'],
        'headline': story['title'],
        'source': primary_source_name,
        'source_url': primary_source_url,
        'time': time_ago(story['created_at']),
        'summary': summary,
        'quotes': quotes,
        'sentiment': sentiment,
        'article_count': story['article_count']
    }
```

### Step 4: New LLM Prompts

#### Sentiment Analysis Prompt (NEW)

Add to `llm_service.py`:

```python
def create_sentiment_prompt(article_text, entity_name=None):
    """Classify overall sentiment of article text."""
    entity_clause = f" toward {entity_name}" if entity_name else ""
    return f"""Analyze the overall tone{entity_clause} in this news content.

Classify as one of: POSITIVE, NEUTRAL, NEGATIVE

Then provide a brief (1 sentence) explanation.

Format your response exactly as:
SENTIMENT: [POSITIVE/NEUTRAL/NEGATIVE]
NOTE: [1 sentence explanation]

Content:
{article_text[:3000]}"""
```

#### Digest Introduction Prompt (NEW)

```python
def create_digest_intro_prompt(pillar_name, story_headlines):
    """Generate a 2-3 sentence overview for the digest."""
    headlines_text = "\n".join(f"- {h}" for h in story_headlines)
    return f"""Write a 2-3 sentence executive briefing introduction for today's {pillar_name} intelligence digest.

Today's top stories:
{headlines_text}

Requirements:
- Professional, briefing-style tone
- Highlight the most significant development
- Mention the overall theme or trend
- Maximum 3 sentences
- Do not use bullet points"""
```

#### Digest-Level Sentiment Prompt (NEW)

```python
def create_digest_sentiment_prompt(pillar_name, story_sentiments):
    """Generate overall sentiment note for the entire digest."""
    sentiments_text = "\n".join(
        f"- {s['headline']}: {s['sentiment']}" for s in story_sentiments
    )
    return f"""Based on these story sentiments for {pillar_name}, write a brief (2-3 sentence)
sentiment analysis note summarizing the overall media tone.

Stories:
{sentiments_text}

Format:
OVERALL: [POSITIVE/NEUTRAL/NEGATIVE]
NOTE: [2-3 sentence analysis]"""
```

---

## Performance Optimization

### Problem: LLM calls per story are slow
Generating summary + quotes + sentiment for 10 stories = 30 LLM calls = ~60 seconds

### Solutions

#### 1. Use Pre-Computed AI Content (Preferred)
The `story_ai_content` table already stores generated summaries, quotes, and reports per story. **Check this table first** before calling LLM:

```python
def _enrich_story(self, story):
    # Check if AI content already exists (from previous API calls or pre-generation)
    ai_content = self.story_ai_db.get_ai_content(story['id'])

    if ai_content:
        summary = ai_content.get('concise_summary') or story.get('summary', '')
        quotes = ai_content.get('quotes_data', [])
    else:
        # Fallback: use story.summary from clustering (always available)
        summary = story.get('summary', '')
        quotes = []

    # Only sentiment is truly new — one LLM call per story
    sentiment = self._analyze_sentiment(summary)

    return {...}
```

This reduces LLM calls from 30 → 10 (just sentiment) or even 0 if sentiment is pre-computed.

#### 2. Batch Sentiment in One LLM Call
Instead of 10 sentiment calls, batch all stories into one prompt:

```python
def _batch_sentiment(self, stories):
    stories_text = "\n\n".join(
        f"STORY {i+1}: {s['title']}\n{s['summary']}"
        for i, s in enumerate(stories)
    )
    prompt = f"""Classify sentiment for each story. Reply as JSON array:
[{{"story": 1, "sentiment": "POSITIVE/NEUTRAL/NEGATIVE"}}]

{stories_text}"""

    result = generate_with_gemini(prompt)
    return parse_sentiment_batch(result)
```

**This reduces total LLM calls to 2-3** (batch sentiment + digest intro + overall sentiment).

#### 3. Background Pre-Generation
Cloud Scheduler refreshes digests every 1-6 hours. When user clicks the button, they get the **pre-generated** digest instantly (just a DB read).

```
Cloud Scheduler (every 3h)
  → POST /api/warroom/digests/refresh
    → For each active pillar:
      → generate_pillar_digest(slug, force=True)
      → Store in pillar_digests

User clicks button
  → GET /api/warroom/digests/ali-digest/latest
    → Return from pillar_digests (instant)
```

The "generate" endpoint is the **fallback** — used when user wants real-time fresh data.

---

## Digest Content JSON Schema

```json
{
  "header": {
    "pillar_name": "President Ali",
    "pillar_slug": "ali-digest",
    "date": "March 10, 2026",
    "time": "1:15 PM",
    "generated_at": "2026-03-10T13:15:00Z"
  },
  "introduction": "Today's coverage of President Ali is dominated by...",
  "stories": [
    {
      "story_id": 1234,
      "rank": 1,
      "headline": "President Ali announces new oil revenue plan",
      "sources": [
        { "name": "Stabrook News", "url": "https://..." },
        { "name": "Kaieteur News", "url": "https://..." }
      ],
      "primary_source": "Stabrook News",
      "time_ago": "2h",
      "published_at": "2026-03-10T11:00:00Z",
      "summary": "President Irfaan Ali unveiled a new framework for distributing oil revenues across Guyana's regions. The plan emphasizes infrastructure development and education funding.",
      "article_count": 5,
      "sentiment": "positive"
    }
  ],
  "quotes": [
    {
      "quote": "This is a historic moment for our nation and its people.",
      "person": "President Irfaan Ali",
      "title": "President of Guyana",
      "source": "Stabrook News",
      "story_id": 1234
    }
  ],
  "sentiment": {
    "overall": "positive",
    "note": "Coverage is predominantly positive, focusing on economic development and infrastructure promises. Some neutral coverage from opposition-leaning sources provides balance."
  },
  "metadata": {
    "total_stories": 8,
    "total_articles": 34,
    "sources_represented": ["Stabrook News", "Kaieteur News", "Guyana Chronicle"],
    "time_range_hours": 24,
    "generation_time_ms": 4500
  }
}
```

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Pillar not found | 404 ResourceNotFoundError |
| No stories matched | Return digest with empty stories array + message |
| LLM timeout | Use fallback summary (story.summary from clustering) |
| Embedding missing | 400 BusinessLogicError — admin must generate pillar embedding |
| RPC timeout | 503 DatabaseError with retry guidance |
| Export fails | Return digest content (JSON) even if document generation fails |

---

## Monitoring & Logging

Reuse existing patterns from `utils_errors.py`:

```python
log_operation_start("generate_pillar_digest", pillar_slug=slug)

# ... generation logic ...

log_operation_success("generate_pillar_digest",
    pillar_slug=slug,
    story_count=len(stories),
    generation_time_ms=elapsed,
    sentiment=overall_sentiment
)
```

Track in `pillar_digests.generation_time_ms` for performance monitoring.
