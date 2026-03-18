# Video Metadata Enrichment Pipeline — Specification

## Overview

Enriches existing video records in `content.resources` with pedagogical metadata via LLM (text-only). Follows the image enhancement pipeline pattern exactly.

**Goal:** Batch-enrich all un-enriched video records by merging new fields into `content.resources.metadata` (JSONB `||` update).

---

## New Fields Added to Video Metadata

| Field | Type | Values / Notes |
|---|---|---|
| `blooms_level` | enum | `Remember` \| `Understand` \| `Apply` \| `Analyse` \| `Create` |
| `cognitive_focus` | string[] | `recognition`, `reasoning`, `spatial_understanding`, `pattern_spotting`, `comparison`, `sequencing`, `calculation`, `interpretation`, `critical_thinking` |
| `skills_practiced` | string[] | e.g. `["unit_conversion", "graph_interpretation"]` |
| `contextual_usage` | string[] | `introduction`, `detailed_explanation`, `reference`, `assessment`, `worked_example` |
| `complexity` | enum | `easy` \| `medium` \| `hard` |
| `topics` | string[] | Enriched array from single `topic` field |
| `prerequisite_concepts` | string[] | Required prior knowledge |
| `related_concepts` | string[] | Connected topics |
| `grade_level_range` | object | `{"min": int, "max": int}` |
| `curriculum_alignment` | object | `{"board": str, "chapter": str}` |
| `learning_objectives` | string[] | `"Student will be able to..."` (2–4 items) |
| `key_concepts` | string[] | Main concepts explicitly covered |
| `summary` | string | 2–4 sentence AI-generated rich description for semantic search |
| `is_enriched` | bool | `true` after successful enrichment (idempotency flag) |

---

## File Structure

```
plan/
└── video-metadata-enrichment-spec.md        ← this file

src/pipeline/video_enrichment/
├── __init__.py
├── run_pipeline.py                           ← CLI entry (mirrors image run_pipeline.py)
├── video_orchestrator.py                     ← mirrors ImageOrchestrator
├── metadata_enricher.py                      ← mirrors MetadataExtractor (text-only, single phase)
├── prompts/
│   ├── __init__.py
│   └── enrichment_prompt.py                  ← ENRICHMENT_SYSTEM_PROMPT + build_user_message()
└── utils/
    ├── __init__.py
    ├── db_connector.py                       ← VideoPipelineDB(BaseDBClient)
    ├── metadata_validator.py                 ← validate_enrichment() + normalize_enrichment()
    └── progress_tracker.py                   ← adapted from image version (no phase2 stats)
```

---

## Reference Files (Do Not Modify)

| File | Used For |
|---|---|
| `src/pipeline/image_enhancement/run_pipeline.py` | Exact CLI structure to mirror |
| `src/pipeline/image_enhancement/image_orchestrator.py` | ThreadPoolExecutor, test_mode, error handling |
| `src/pipeline/image_enhancement/metadata_extractor.py` | LLM call pattern, parse, validate, return tuple |
| `src/pipeline/image_enhancement/utils/db_connector.py` | `BaseDBClient` extension pattern |
| `src/pipeline/image_enhancement/utils/llm_utils.py` | `ImagePipelineLLMClient` — reuse directly |
| `src/pipeline/image_enhancement/utils/metadata_validator.py` | Validation structure to mirror |
| `src/pipeline/image_enhancement/prompts/metadata_prompt.py` | Prompt structure to mirror |
| `pipeline/common/db.py` | `BaseDBClient` base class |

---

## SQL Queries

### Fetch un-enriched videos
```sql
SELECT id, title, metadata
FROM content.resources
WHERE resource_type = 'video'
  AND is_deleted = false
  AND (metadata->>'is_enriched' IS NULL OR metadata->>'is_enriched' = 'false')
ORDER BY id ASC;
```

### Count un-enriched videos (dry-run)
```sql
SELECT COUNT(*)
FROM content.resources
WHERE resource_type = 'video'
  AND is_deleted = false
  AND (metadata->>'is_enriched' IS NULL OR metadata->>'is_enriched' = 'false');
```

### Fetch single video by ID (--video-id test mode)
```sql
SELECT id, title, metadata
FROM content.resources
WHERE id = %s::uuid;
```

### Merge enriched fields back (JSONB merge)
```sql
UPDATE content.resources
SET metadata = metadata || %s::jsonb,
    updated_at = NOW()
WHERE id = %s::uuid;
```

### Verify enrichment
```sql
SELECT
  metadata->>'is_enriched'    AS is_enriched,
  metadata->>'blooms_level'   AS blooms_level,
  metadata->>'complexity'     AS complexity,
  metadata->>'summary'        AS summary
FROM content.resources
WHERE resource_type = 'video'
LIMIT 5;
```

---

## Input Fields Used for LLM Enrichment

All sourced from `content.resources.metadata` (existing flat schema):

| Field | Source Key | Notes |
|---|---|---|
| `title` | `title` (top-level column) | Video title |
| `description` | `metadata->>'description'` | Long description |
| `short_description` | `metadata->>'short_description'` | Brief description |
| `subject` | `metadata->>'subject'` | e.g. "Mathematics" |
| `chapter` | `metadata->>'chapter'` | e.g. "Quadratic Equations" |
| `topic` | `metadata->>'topic'` | e.g. "Completing the Square" |
| `unit` | `metadata->>'unit'` | e.g. "Algebra" |
| `exam` | `metadata->>'exam'` | e.g. "JEE", "NEET" |
| `goal` | `metadata->>'goal'` | Learning goal |
| `type` | `metadata->>'type'` | Content type |
| `video_type` | `metadata->>'video_type'` | e.g. "concept", "solved_example" |
| `duration` | `metadata->>'duration'` | Video length in seconds |

Null fields rendered as `'N/A'` in the user message.

---

## Sample Input Record

```json
{
  "id": "3f2a1b4c-...",
  "title": "Introduction to Quadratic Equations",
  "metadata": {
    "subject": "Mathematics",
    "chapter": "Quadratic Equations",
    "topic": "Standard Form",
    "unit": "Algebra",
    "exam": "JEE",
    "goal": "Understand the standard form ax² + bx + c = 0",
    "description": "This video introduces quadratic equations, explaining the standard form and identifying coefficients a, b, and c with worked examples.",
    "short_description": "Learn the standard form of a quadratic equation.",
    "video_type": "concept",
    "type": "video",
    "duration": 480
  }
}
```

---

## Sample LLM Output

```json
{
  "blooms_level": "Understand",
  "cognitive_focus": ["recognition", "reasoning", "pattern_spotting"],
  "skills_practiced": ["equation_identification", "coefficient_recognition"],
  "contextual_usage": ["introduction", "detailed_explanation"],
  "complexity": "easy",
  "topics": ["Quadratic Equations", "Standard Form", "Coefficients", "Algebra"],
  "prerequisite_concepts": ["linear_equations", "basic_algebra", "exponents"],
  "related_concepts": ["factoring", "quadratic_formula", "discriminant", "parabola"],
  "grade_level_range": {"min": 9, "max": 11},
  "curriculum_alignment": {"board": "CBSE", "chapter": "Quadratic Equations"},
  "learning_objectives": [
    "Student will be able to identify the standard form ax² + bx + c = 0",
    "Student will be able to extract coefficients a, b, and c from a given equation",
    "Student will be able to distinguish quadratic from linear equations"
  ],
  "key_concepts": ["standard form", "coefficients", "degree 2 polynomial"],
  "summary": "This video introduces quadratic equations by presenting the standard form ax² + bx + c = 0 and walking through the identification of coefficients a, b, and c. Suitable for students in grades 9–11 encountering algebra for JEE preparation, the video uses worked examples to build recognition skills. It provides a foundation for subsequent topics like factoring and the quadratic formula."
}
```

---

## Module Descriptions

### `prompts/enrichment_prompt.py`
- `ENRICHMENT_SYSTEM_PROMPT`: Full schema definition with all 13 fields, enum constraints, and output format instructions.
- `build_user_message(video_record: dict) -> str`: Formats the 12 text fields from the video record into a structured prompt. Null values rendered as `'N/A'`.

### `utils/metadata_validator.py`
- `REQUIRED_ENRICHMENT_FIELDS`: List of all 13 new fields (excluding `is_enriched`).
- Enum constants: `VALID_BLOOMS_LEVELS`, `VALID_COMPLEXITY`, `VALID_CONTEXTUAL_USAGE`, `VALID_COGNITIVE_FOCUS`.
- `validate_enrichment(data: Dict) -> Tuple[bool, List[str]]`: Checks required fields and enum values.
- `normalize_enrichment(data: Dict) -> Dict`: Adds `is_enriched: True` and ensures all fields present.

### `utils/db_connector.py`
Extends `BaseDBClient` (5 levels up: `pipeline/common/db.py`):
- `get_unenriched_videos() -> List[Dict]`
- `get_video_by_id(resource_id: str) -> Optional[Dict]`
- `update_video_metadata(resource_id: str, enriched_fields: Dict) -> None`
- `count_unenriched_videos() -> int`

### `utils/progress_tracker.py`
Adapted from image version:
- Removes `phase2_total_tokens` / `image_gen_tokens` stats.
- Banner: `VIDEO ENRICHMENT PIPELINE - FINAL REPORT`.
- Tracks: `total`, `processed`, `failed`, `skipped`, `total_tokens`, `prompt_tokens`, `completion_tokens`.

### `metadata_enricher.py`
`MetadataEnricher` class — mirrors `MetadataExtractor` but text-only:
- Reuses `ImagePipelineLLMClient` directly.
- `enrich(video_record: Dict) -> Tuple[Dict, Dict]`: builds text-only messages, calls LLM, validates, normalizes, returns `(enrichment, usage)`.

### `video_orchestrator.py`
`VideoOrchestrator` — mirrors `ImageOrchestrator`:
- No `phase2_model`, no `gcs_manager`.
- `run_pipeline()`: fetches un-enriched videos from DB, processes concurrently with `ThreadPoolExecutor`.
- `_process_video()`: single LLM phase. In `test_mode`, prints JSON and skips DB write. Otherwise calls `db.update_video_metadata()`.
- `--dry-run`: skips LLM, prints count and exits.

### `run_pipeline.py`
CLI entry — mirrors image `run_pipeline.py`:
- `validate_environment()`: checks `OPENROUTER_API_KEY`, `SUPABASE_DB_*` (no `GCP_CREDENTIALS_JSON`).
- `argparse`: `--max-workers`, `--test-mode`, `--debug`, `--dry-run`, `--video-id`.
- Optional env vars: `ENRICHMENT_MODEL` (default: `google/gemini-2.5-flash`), `ENRICHMENT_TEMPERATURE` (default: `0`).

---

## Key Differences from Image Pipeline

| Aspect | Image Pipeline | Video Enrichment |
|---|---|---|
| Input source | PNG files from folder | Video records from DB |
| LLM input | base64 image + text | Text only (12 metadata fields) |
| LLM phases | 2 (metadata + style) | 1 (enrichment only) |
| DB operation | INSERT new records | UPDATE JSONB `\|\|` merge |
| GCS upload | Yes (3 versions) | No |
| Idempotency | Check by filename in DB | Filter by `is_enriched` in fetch query |
| Env vars | Includes `GCP_CREDENTIALS_JSON` | No GCP vars needed |

---

## Environment Variables

### Required
```
OPENROUTER_API_KEY
SUPABASE_DB_HOST
SUPABASE_DB_NAME
SUPABASE_DB_USER
SUPABASE_DB_PASSWORD
```

### Optional
```
ENRICHMENT_MODEL        # default: google/gemini-2.5-flash
ENRICHMENT_TEMPERATURE  # default: 0
SUPABASE_DB_PORT        # default: 5432 (from BaseDBClient)
```

---

## CLI Usage

```bash
# Dry run — count un-enriched videos, no LLM calls
python src/pipeline/video_enrichment/run_pipeline.py --dry-run

# Test single video — LLM call, print JSON, no DB write
python src/pipeline/video_enrichment/run_pipeline.py --test-mode --video-id <uuid>

# Test all — LLM calls for all, print stats, no DB write
python src/pipeline/video_enrichment/run_pipeline.py --test-mode

# Full run — enrich all, update DB
python src/pipeline/video_enrichment/run_pipeline.py

# Full run with more workers
python src/pipeline/video_enrichment/run_pipeline.py --max-workers 10

# Debug logging
python src/pipeline/video_enrichment/run_pipeline.py --debug --test-mode
```

---

## Verification Checklist

1. **Dry run**: prints count of un-enriched videos, zero LLM calls made.
2. **Single video test**: `--test-mode --video-id <uuid>` → prints enriched JSON, no DB write.
3. **Batch test**: `--test-mode` → all videos enriched via LLM, printed stats, no DB write.
4. **Full run**: enriches all, updates DB, final report with token counts.
5. **Re-run idempotency**: run again → "No un-enriched videos found. Nothing to do."
6. **DB verification**:
```sql
SELECT
  metadata->>'is_enriched',
  metadata->>'blooms_level',
  metadata->>'summary'
FROM content.resources
WHERE resource_type = 'video'
LIMIT 5;
```
