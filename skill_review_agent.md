# Proposal: Skill Review & Scoring System

## 1. Problem

Once a skill is written to Neo4j, it's permanently accepted. The pipeline's only post-creation check is structural validation (Stage 7: orphaned nodes, circular prerequisites). There is no quality gate that evaluates whether a skill is **well-formed**, **assessable**, **properly scoped**, or **correctly connected**.

We already have signals that could inform quality — embedding similarity scores, LLM decision confidence, prerequisite chain depth, age-range consistency — but none of them are captured or used after graph construction.

### Consequences

- **Low-quality skills accumulate silently** — vague titles, missing verbs, overlapping scope with neighbors
- **No way to prioritize remediation** — without scores, every skill looks equally valid
- **Prerequisite chains can be pedagogically wrong** without anyone noticing until a learner hits them
- **Source quality varies** — some CSVs produce better skills than others, but we can't quantify this

---

## 2. Solution Overview

Add a **Skill Review & Scoring** stage to the pipeline that evaluates every skill across 6 dimensions, produces a composite score, and assigns a review status.

### 6 Scoring Dimensions

| # | Dimension | What It Measures | Weight |
|---|-----------|-----------------|--------|
| 1 | **Clarity** | Is the skill title unambiguous and self-contained? | 20% |
| 2 | **Assessability** | Does it use a measurable verb (Bloom's taxonomy)? Can you write a test item for it? | 25% |
| 3 | **Granularity** | Is it atomic (one action) vs. compound (bundles multiple actions)? | 15% |
| 4 | **Prerequisite Coherence** | Are its prerequisites logically ordered and complete? No circular or missing links? | 20% |
| 5 | **Age-Range Plausibility** | Does the `age_range` make sense for the skill's complexity? | 10% |
| 6 | **Graph Placement** | Is it under the right Domain, Strand, and Concept? Does it overlap with sibling skills? | 10% |

### Composite Score

```
composite = Σ (dimension_score × weight)
```

Each dimension is scored **1–5** (integer). Composite ranges from **1.0 to 5.0**.

### Review Status

Statuses are evaluated **in priority order** — the first matching rule wins:

| Priority | Status | Condition | Meaning |
|----------|--------|-----------|---------|
| 1 | `flagged` | composite < 2.5 **or** any dimension = 1 | Likely needs rewrite or removal |
| 2 | `approved` | composite ≥ 4.0 **and** no dimension < 3 | Production-ready |
| 3 | `needs_review` | everything else | Human should check |

Implementation:
```python
def classify(composite: float, dim_scores: list[int]) -> str:
    if composite < 2.5 or min(dim_scores) == 1:
        return "flagged"
    if composite >= 4.0 and min(dim_scores) >= 3:
        return "approved"
    return "needs_review"
```

---

## 3. Scoring Rubric

### 3.1 Clarity (1–5)

| Score | Anchor |
|-------|--------|
| 5 | Title is a complete verb phrase; no jargon; a teacher reads it and immediately knows what to assess. Example: "Add fractions with unlike denominators" |
| 4 | Clear but slightly ambiguous scope. Example: "Work with unlike denominators" |
| 3 | Understandable but requires context from the parent concept to interpret |
| 2 | Vague or uses undefined terms. Example: "Fraction operations" |
| 1 | Incomprehensible, truncated, or just a noun phrase. Example: "Denominators" |

### 3.2 Assessability (1–5)

Uses Bloom's Taxonomy action verbs as the primary signal.

**Strong verbs** (score 4–5): identify, calculate, solve, compare, classify, construct, evaluate, analyze, derive, prove, apply, convert, estimate, measure, order, plot, simplify, factor, demonstrate

**Weak verbs** (score 2–3): understand, know, learn, explore, appreciate, recognize, be aware of

**No verb / noun-only** (score 1): "Fractions", "Linear equations", "Data"

| Score | Anchor |
|-------|--------|
| 5 | Strong Bloom's verb + specific object + clear condition. "Solve two-step linear equations with integer coefficients" |
| 4 | Strong verb + object but condition is implicit. "Solve two-step linear equations" |
| 3 | Weak verb or object is too broad. "Understand linear equations" |
| 2 | No clear action; reads like a topic, not a skill. "Linear equation concepts" |
| 1 | Cannot determine what a student would do. "Equations" |

### 3.3 Granularity (1–5)

| Score | Anchor |
|-------|--------|
| 5 | Single, atomic action. One verb, one object, one condition. "Multiply two 2-digit numbers" |
| 4 | Single action, slightly broad object. "Multiply multi-digit numbers" |
| 3 | Two closely related actions bundled. "Add and subtract fractions" |
| 2 | Multiple distinct actions. "Add, subtract, and compare fractions with unlike denominators" |
| 1 | Entire topic compressed into one skill. "Master all fraction operations" |

### 3.4 Prerequisite Coherence (1–5)

| Score | Anchor |
|-------|--------|
| 5 | All prerequisites are logically necessary and correctly ordered; no missing steps |
| 4 | Prerequisites are correct but one intermediate step could be added |
| 3 | One prerequisite is questionable (too advanced or not truly required) |
| 2 | Multiple prerequisites are wrong or missing; learning path has gaps |
| 1 | Circular dependency detected, or prerequisites are from an unrelated domain |

### 3.5 Age-Range Plausibility (1–5)

| Score | Anchor |
|-------|--------|
| 5 | Age range matches established curriculum standards (e.g., single-digit addition at ages 6–7) |
| 4 | Reasonable but spans slightly too wide (e.g., ages 6–10 for single-digit addition) |
| 3 | Off by 1–2 years from expected range |
| 2 | Off by 3+ years or range is implausibly wide (e.g., ages 5–15) |
| 1 | Age range is missing, or clearly wrong (e.g., calculus at age 6) |

### 3.6 Graph Placement (1–5)

Evaluates placement across all hierarchy levels: Domain, Strand, and Concept.

| Score | Anchor |
|-------|--------|
| 5 | Correct domain, strand, and concept; no overlap with sibling skills |
| 4 | Correct placement but mild overlap with one sibling (>0.85 embedding similarity) |
| 3 | Placement is defensible but a better concept or strand exists in the graph |
| 2 | Clearly belongs under a different concept or strand |
| 1 | Wrong domain entirely, or is a near-duplicate of another skill (>0.95 similarity) |

---

## 4. Architecture

### Pipeline Integration: New Stage Between Validation and Completed

The review stage runs **after validation** (currently Stage 7), on skills in Neo4j that have no existing `review_score` — or on-demand via CLI.

**Stage numbering change required:** The current `PipelineStage` enum uses `COMPLETED = 8`. Adding review requires inserting `SKILL_REVIEW = 8` and bumping `COMPLETED` to `9`:

```
Current enum:                  After this proposal:
  SCRAPING = 1                   SCRAPING = 1
  PARSING = 2 (pass-through)    PARSING = 2 (pass-through)
  EXTRACTION = 3                 EXTRACTION = 3
  HIERARCHY_BUILDING = 4         HIERARCHY_BUILDING = 4 (pass-through)
    (pass-through)
  GRAPH_CONSTRUCTION = 5         GRAPH_CONSTRUCTION = 5 (no-op)
    (no-op, merged into 3)
  PREREQUISITE_INFERENCE = 6     PREREQUISITE_INFERENCE = 6
  VALIDATION = 7                 VALIDATION = 7
  COMPLETED = 8                  SKILL_REVIEW = 8        ← NEW
                                 COMPLETED = 9            ← bumped
```

The `state_manager.py` `from_name()` mapping must also add `"review": cls.SKILL_REVIEW` and `"skill_review": cls.SKILL_REVIEW` so that `--restart-from review` works.

**Query for skills to score:** The review stage queries Neo4j directly (not the `curriculum_items` PostgreSQL table):
```cypher
MATCH (s:Skill)
WHERE s.review_score IS NULL
RETURN s.id, s.title, s.description, s.age_range
```

### Two Scoring Modes

**Rule-based scoring** (dimensions 3–6) — fast, no LLM cost:
- **Granularity**: Count verbs and conjunctions in the title; check character length.
  **Caveat:** Verb/conjunction counting produces false positives for skills that are pedagogically atomic but have compound titles (e.g., "Add and subtract integers on a number line" is a standard single skill despite having two verbs). Expect the LLM override to correct ~15–25% of rule-based granularity scores. This is acceptable — the rule score serves as a starting point, not a final answer.
- **Prerequisite Coherence**: Graph traversal — detect cycles, check ordering, count orphaned chains
- **Age-Range Plausibility**: Compare against grade-band lookup tables (e.g., ages 5–7 → K–2 skills)
- **Graph Placement**: Compute embedding similarity against sibling skills; flag >0.90 overlaps. Also check strand assignment by comparing against skills in neighboring strands.

**LLM-as-Judge scoring** (dimensions 1–2, plus override on 3–6) — batched for cost efficiency:
- **Clarity** and **Assessability** require semantic judgment that rules can't reliably provide
- The LLM also gets a chance to override rule-based scores when context matters (e.g., a seemingly compound title that is actually a single standard skill)

### Processing Flow

```
1. Query Neo4j: all skills with no review_score (or where rescore is requested)
     ↓
2. Rule-based scoring (dimensions 3–6)
   - Fetch skill + siblings + prerequisites from Neo4j
   - Compute granularity heuristics (verb count, conjunction count, length)
   - Run graph checks (cycle detection, prerequisite ordering)
   - Compare age_range against grade-band table
   - Compute sibling similarity from skill_embeddings
     ↓
3. Batch skills into groups of 10 for LLM-as-Judge
   - Each batch includes skill title, description, concept context, sibling list, rule-based pre-scores
     ↓
4. LLM-as-Judge call (one call per batch of 10)
   - Scores dimensions 1–2 (Clarity, Assessability)
   - Optionally overrides dimensions 3–6 with reasoning
     ↓
5. Compute composite score and review status
     ↓
6. Write scores to Neo4j (skill node properties) + detail to PostgreSQL
```

---

## 5. Storage Schema

### Neo4j: Skill Node Properties (new)

```cypher
// Added to every Skill node after scoring
skill.review_score       // Float: composite 1.0–5.0
skill.review_status      // String: "approved" | "needs_review" | "flagged"
skill.reviewed_at        // DateTime: when last scored
skill.review_version     // Integer: scoring rubric version (starts at 1)
```

These properties enable Cypher queries like:
```cypher
// Find all flagged skills
MATCH (s:Skill) WHERE s.review_status = 'flagged' RETURN s

// Average score by concept
MATCH (s:Skill)-[:BELONGS_TO]->(c:Concept)
RETURN c.title, avg(s.review_score) AS avg_score
ORDER BY avg_score ASC

// Skills needing review in a domain
MATCH (s:Skill)-[:BELONGS_TO]->(c:Concept)-[:BELONGS_TO]->(d:Domain)
WHERE d.title = 'Mathematics' AND s.review_status = 'needs_review'
RETURN s.title, s.review_score
```

### PostgreSQL: Skill Review Detail Table (new)

```sql
CREATE TABLE skill_reviews (
    id SERIAL PRIMARY KEY,
    neo4j_skill_id VARCHAR(50) NOT NULL,
    review_version INTEGER NOT NULL DEFAULT 1,

    -- Per-dimension scores (1–5)
    clarity_score INTEGER NOT NULL,
    assessability_score INTEGER NOT NULL,
    granularity_score INTEGER NOT NULL,
    prerequisite_coherence_score INTEGER NOT NULL,
    age_range_plausibility_score INTEGER NOT NULL,
    graph_placement_score INTEGER NOT NULL,

    -- Composite
    composite_score FLOAT NOT NULL,
    review_status VARCHAR(20) NOT NULL,  -- approved | needs_review | flagged

    -- Reasoning (from LLM + rules)
    dimension_reasoning JSONB NOT NULL,
    /*
      {
        "clarity": "Title is a complete verb phrase with specific object...",
        "assessability": "Uses Bloom's verb 'solve' with measurable condition...",
        "granularity": "Single verb detected, no conjunctions",
        "prerequisite_coherence": "3 prerequisites, correctly ordered, no cycles",
        "age_range_plausibility": "Ages [10,11] matches grade 5 expectations",
        "graph_placement": "Max sibling similarity: 0.82 (below threshold)"
      }
    */

    -- Metadata
    scoring_mode VARCHAR(20) NOT NULL,   -- "full" (rules+LLM) | "rules_only"
    llm_model VARCHAR(100),              -- e.g., "openai/gpt-4o-mini"
    created_at TIMESTAMP DEFAULT NOW(),

    UNIQUE (neo4j_skill_id, review_version)
);

CREATE INDEX idx_skill_reviews_skill ON skill_reviews(neo4j_skill_id);
CREATE INDEX idx_skill_reviews_status ON skill_reviews(review_status);
```

---

## 6. LLM-as-Judge Strategy

### Batched Prompt Design

Each LLM call reviews **10 skills at once**. The prompt includes:

1. The scoring rubric (dimensions 1–2 fully, dimensions 3–6 as override option)
2. For each skill in the batch:
   - Skill title, description, age_range
   - Parent concept title and domain title
   - Sibling skill titles (same concept)
   - Prerequisite skill titles (ordered)
   - Rule-based pre-scores for dimensions 3–6

### Response Schema

```json
{
  "reviews": [
    {
      "skill_id": "skill_abc123def4",
      "clarity_score": 5,
      "clarity_reasoning": "Complete verb phrase with specific object and condition",
      "assessability_score": 4,
      "assessability_reasoning": "Strong Bloom's verb 'solve', object could be more specific",
      "overrides": {
        "granularity_score": null,
        "granularity_reasoning": null,
        "prerequisite_coherence_score": null,
        "prerequisite_coherence_reasoning": null,
        "age_range_plausibility_score": 3,
        "age_range_plausibility_reasoning": "Age range [8,9] is low for this complexity; expect [10,11]",
        "graph_placement_score": null,
        "graph_placement_reasoning": null
      }
    }
  ]
}
```

`null` overrides mean the LLM agrees with the rule-based score.

### Cost Estimate

Assuming 180 skills (current seed data size):
- **Batches**: 180 / 10 = 18 LLM calls
- **Input tokens per batch**: ~2,000 (rubric) + ~300 per skill × 10 = ~5,000 tokens
- **Output tokens per batch**: ~150 per skill × 10 = ~1,500 tokens
- **Total**: ~90K input + ~27K output tokens
- **Cost at gpt-4o-mini rates** ($0.15/1M input, $0.60/1M output): ~$0.03

At 1,000 skills: ~$0.15. At 10,000 skills: ~$1.50. Cost is negligible.

**Production scale (1,720 IXL skills):**
- **Batches**: 1,720 / 10 = 172 LLM calls
- **Total**: ~860K input + ~258K output tokens
- **Cost at gpt-4o-mini rates**: ~$0.28
- **Cost at Gemini 2.0 Flash rates** ($0.10/1M input, $0.40/1M output): ~$0.19
- **Time estimate**: ~6 minutes at ~2s per call

The pipeline currently uses **Gemini 2.0 Flash via OpenRouter** (`google/gemini-2.0-flash-001`), making the actual cost even lower than the gpt-4o-mini estimates above. The `review_llm_override` config option (see Section 7) allows using a different model specifically for judging if needed.

**⚠ Model deprecation:** `gemini-2.0-flash` and `gemini-2.0-flash-001` are scheduled for **shutdown on June 1, 2026**. The recommended replacement is `gemini-2.5-flash`. The pipeline's `llm_model` setting in `src/config/settings.py` (or the `.env` file) must be updated before that date. This applies to both the extraction stage and the review/scoring stage.

### Agreement Calibration

Based on LLM-as-a-Judge research (OpenReview 2025), expect ~90% agreement with human expert ratings on a 1–5 scale when:
- Rubric anchors are concrete (not abstract)
- Each score point has an example
- The LLM sees sibling context (not just the skill in isolation)

Our rubric (Section 3) is designed to meet all three criteria.

---

## 7. What Changes

| Component | File | Change |
|-----------|------|--------|
| Orchestrator | `src/pipeline/orchestrator.py` | Add `_run_skill_review()` as the new SKILL_REVIEW stage; call after validation |
| New processor | `src/processors/skill_reviewer.py` | **New file.** Rule-based scoring + LLM-as-Judge batching + composite calculation + error handling (see Section 11) |
| Graph store | `src/storage/graph_store.py` | Add `update_skill_review_scores()` — sets `review_score`, `review_status`, `reviewed_at`, `review_version` on Skill nodes |
| Embedding store | `src/storage/embedding_store.py` | Add `skill_reviews` table creation in `_ensure_tables()`. Add `insert_skill_review()` and `get_skill_reviews()` methods |
| LLM client | `src/processors/llm_client.py` | No changes — existing `call_llm()` with JSON repair handles the batched judge prompt |
| CLI | `src/cli/main.py` | Add `review-skills` command for on-demand scoring. Add `--skip-review` flag to `run` command |
| Config | `src/config/settings.py` | Add `review_batch_size` (default 10), `review_llm_override` (optional separate model for judging) |
| State manager | `src/pipeline/state_manager.py` | Add `SKILL_REVIEW = 8`, bump `COMPLETED` to `9`. Add `"review"` and `"skill_review"` to `from_name()` mapping |

### What Does NOT Change

- Graph schema (Domain, Concept nodes) — only Skill nodes get new properties
- Pipeline stages 1–7 — review is additive, not modifying existing flow
- Embedding model or similarity search — reuses existing `skill_embeddings` for sibling comparison
- API and visualization — no changes required (but could consume scores in future)
- `sources.yaml` / `subjects.yaml` — untouched

---

## 8. CLI Commands

### Integrated with Pipeline

```bash
# Normal run — includes review as Stage 8
python main.py run --subject mathematics

# Skip review (faster iteration during development)
python main.py run --subject mathematics --skip-review

# Run only review on existing graph (no ingestion/extraction)
python main.py run --subject mathematics --restart-from review
```

### On-Demand Review

```bash
# Score all unreviewed skills
python main.py review-skills --subject mathematics

# Re-score everything (ignore existing scores)
python main.py review-skills --subject mathematics --rescore

# Rules-only mode (no LLM calls, faster, free)
python main.py review-skills --subject mathematics --rules-only

# Show review summary
python main.py review-skills --subject mathematics --summary
# Output:
#   Total skills: 180
#   Approved: 142 (78.9%)
#   Needs review: 31 (17.2%)
#   Flagged: 7 (3.9%)
#   Average composite: 4.12
#   Lowest dimension (avg): Prerequisite Coherence (3.41)

# List flagged skills
python main.py review-skills --subject mathematics --status flagged
```

---

## 9. Data Flow

```
Stage 8: Skill Review & Scoring
══════════════════════════════════════════════════════════════

Neo4j                          PostgreSQL
──────                         ──────────
MATCH (s:Skill)                skill_embeddings
WHERE s.review_score IS NULL   (sibling similarity lookup)
  │                                │
  ▼                                ▼
┌─────────────────────────────────────────────┐
│  Rule-Based Scorer                          │
│                                             │
│  For each skill:                            │
│   • Count verbs/conjunctions → granularity  │
│   • Graph traversal → prereq coherence      │
│   • Grade-band lookup → age plausibility    │
│   • Sibling cosine sim → graph placement    │
└─────────────────────┬───────────────────────┘
                      │ pre-scores (dims 3–6)
                      ▼
┌─────────────────────────────────────────────┐
│  LLM-as-Judge (batches of 10)              │
│                                             │
│  Input: skill + context + pre-scores        │
│  Output: dims 1–2 scores + optional         │
│          overrides for dims 3–6             │
└─────────────────────┬───────────────────────┘
                      │ all 6 dimension scores
                      ▼
┌─────────────────────────────────────────────┐
│  Composite Calculator                       │
│                                             │
│  composite = Σ(score × weight)              │
│  status = approved | needs_review | flagged │
└──────────┬──────────────────┬───────────────┘
           │                  │
           ▼                  ▼
      Neo4j Skill        PostgreSQL
      node update         skill_reviews
      ┌──────────┐       ┌──────────────┐
      │ +review_ │       │ per-dimension │
      │  score   │       │ scores +      │
      │ +review_ │       │ reasoning +   │
      │  status  │       │ metadata      │
      └──────────┘       └──────────────┘
```

---

## 10. Scope & Limitations

This stage is **diagnostic only**. It scores skills and assigns review statuses, but it does **not** automatically fix, remove, or rewrite flagged skills. Flagged and needs_review skills remain in the graph exactly as they are, and are used in prerequisite trees, visualizations, and vector search like any other skill.

The intended workflow is:
1. Pipeline runs → review stage scores all new/unscored skills.
2. Operators use the CLI (`--summary`, `--status flagged`) or Cypher queries to triage low-scoring skills.
3. Manual remediation: operator edits the graph directly, re-runs with `--clean`, or (in future) uses the Auto-Remediation extension.

Automated remediation is deliberately excluded from this proposal to keep the scope bounded. See Section 14 (Future Extensions) for the planned follow-up.

---

## 11. Error Handling

### LLM Judge Failures

When a batched LLM-as-Judge call fails (timeout, malformed JSON, API error), the reviewer follows this fallback strategy:

| Attempt | Action |
|---------|--------|
| 1 | Retry the full batch (use existing `llm_client` retry/backoff logic) |
| 2 | If retry fails, split the batch in half and retry each half independently |
| 3 | If a half-batch still fails, fall back to **rules-only scoring** for those skills (dimensions 1–2 get score `null` and status `needs_review`) |

Skills scored in rules-only fallback mode are recorded with `scoring_mode = 'rules_only'` in the `skill_reviews` table. They can be re-scored later with `review-skills --rescore`.

### Partial Batch Handling

If the LLM returns valid JSON but is missing scores for some skills in the batch (e.g., returns 8 reviews for a batch of 10), the missing skills are logged and queued for re-scoring in the next batch iteration.

### Neo4j / PostgreSQL Write Failures

If the score write to Neo4j or PostgreSQL fails after successful scoring:
- Log the error with the skill ID and computed scores.
- Do **not** mark the skill as reviewed (so it will be picked up again on re-run).
- Continue processing remaining skills.

---

## 12. Incremental Run Behavior

The pipeline supports incremental runs (e.g., `--rows 100`, then `--rows 200`). Review scores can become **stale** when the graph changes around an already-scored skill:

| Change | Affected Dimensions | Action |
|--------|-------------------|--------|
| New skill added to the same concept | Graph Placement (sibling overlap may change) | Mark sibling skills' `review_score` as stale |
| Skill's prerequisites change (edge added/removed) | Prerequisite Coherence | Mark affected skill's `review_score` as stale |
| Skill gets `update_existing` from a later CSV item | All dimensions (title/description/age_range may change) | Clear the skill's `review_score` (force re-score) |

### Staleness Detection

Add a `review_stale` boolean property to Skill nodes (default `false`). The graph construction step sets `review_stale = true` on affected neighbors when:
- A new skill is created in a concept that has already-reviewed siblings.
- A PREREQUISITE edge is added or removed involving an already-reviewed skill.
- An existing skill is updated via `update_existing`.

The review stage processes skills where `review_score IS NULL OR review_stale = true`, then resets `review_stale = false` after scoring.

### Conservative Default

On first implementation, **always re-score all skills** per run (simpler, and cost is negligible at ~$0.03 per 180 skills). The staleness optimization can be added later if graph sizes grow past ~5,000 skills and scoring latency becomes noticeable.

---

## 13. Rubric Version Migration

The `review_version` field (on both Neo4j Skill nodes and the `skill_reviews` PostgreSQL table) tracks which version of the scoring rubric was used. This enables safe rubric evolution.

### Rules

1. **Version is a monotonically increasing integer** starting at 1. Each change to the rubric (score anchors, dimension weights, or dimensions themselves) increments the version.

2. **Old and new versions coexist.** The `skill_reviews` table uses `UNIQUE (neo4j_skill_id, review_version)`, so a skill can have reviews from multiple rubric versions. The Neo4j Skill node always reflects the **latest** version's scores.

3. **`--rescore` re-scores using the current rubric version.** If the current version is 2 and a skill has a v1 review, `--rescore` creates a new v2 row in `skill_reviews` and updates the Skill node. The v1 row is preserved for audit.

4. **Rubric changes do not auto-invalidate existing scores.** Operators must explicitly run `review-skills --rescore` after a rubric update. The CLI can warn when skills have scores from an older version:
   ```
   ⚠ 142 skills scored with rubric v1 (current: v2). Run --rescore to update.
   ```

5. **Version history is documented** in a `REVIEW_CHANGELOG.md` (or inline in `skill_reviewer.py`) so operators know what changed between versions.

---

## 14. Future Extensions

These are **not part of this proposal** but are natural next steps:

### Auto-Remediation
When a skill is `flagged`, automatically generate a corrected version using the LLM (new title, updated age_range, fixed prerequisites) and present it as a suggested edit. Requires a `skill_corrections` queue table and a separate approval step.

### Human Review UI
Add a review dashboard to the web visualization where a curriculum expert can:
- Browse skills sorted by composite score (worst first)
- Override individual dimension scores
- Approve/reject LLM suggestions
- Track inter-rater agreement between LLM and human scores

### Source Quality Tracking
Aggregate review scores by source to identify which curriculum CSVs produce the best skills:
```cypher
MATCH (s:Skill)
UNWIND s.source_names AS source
RETURN source, avg(s.review_score) AS avg_quality, count(s) AS skill_count
ORDER BY avg_quality ASC
```

**Dependency:** This extension requires the `source_names` property on Skill nodes, which is described in a separate proposal (`docs/proposals/store-source-names-in-neo4j.md`) and is not yet implemented. Source Quality Tracking should only be built after that proposal lands.

### Feedback Loop to Extraction
Use review scores to improve the extraction prompt. If skills from a particular source consistently score low on assessability, inject guidance like "ensure every skill title starts with a Bloom's taxonomy verb" into the `SkillDecisionMaker` prompt for that source.

### Score-Based Visualization
Color-code skill cells in the territory map by review status: green (approved), yellow (needs review), red (flagged). Add a filter toggle to the search bar.
