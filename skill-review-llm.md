# Proposal: Skill Review, Scoring & Auto-Remediation System

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

Add a **four-pass, wave-based Skill Review** as a standalone command, separate from the ingestion pipeline. Skills are grouped by prerequisite depth (foundations first) and processed in waves. Within each wave, four sequential passes run:

1. **Pass 1 — Score**: Evaluate every skill using a **single LLM call per skill** across 6 dimensions. Read-only, zero graph risk.
2. **Pass 2 — Fix Metadata**: For failing skills, a **second LLM call** fixes title, description, and age_range. Self-contained, no graph structural changes.
3. **Pass 3 — Fix Structure**: For skills with prerequisite problems, a **third LLM call** proposes edge changes. This call sees the **already-fixed metadata** from Pass 2, producing better structural decisions. Changes go through deterministic validation gates before applying.
4. **Pass 4 — Verify**: Re-score all fixed skills. Revert any fix that made the score worse.

**Why separate metadata from structural fixes:** A single remediation call that simultaneously rewrites "Fraction stuff" into a proper title AND figures out prerequisites is reasoning about two things with stale input. Fixing the title first means the structural fixer asks "given this skill called 'Add fractions with unlike denominators', what should its prerequisites be?" — a much better prompt.

**Why wave-based:** Skills in the same wave have no prerequisite relationships with each other (by definition of depth grouping). Concurrent execution within a wave is safe. Wave barriers ensure prerequisites are fixed before their dependents are scored.

**Fully automated, no human review.** The entire graph was built by LLMs during extraction. If we trust the LLM to create prerequisite edges, it's consistent to trust it to fix them — with deterministic validation gates catching the dangerous cases (cycles, missing IDs, age inconsistencies).

### 6 Scoring Dimensions

| # | Dimension | What It Measures | Weight |
|---|-----------|-----------------|--------|
| 1 | **Clarity** | Is the skill title unambiguous and self-contained? | 20% |
| 2 | **Assessability** | Does it use a measurable verb (Bloom's taxonomy)? Can you write a test item for it? | 25% |
| 3 | **Granularity** | Is it atomic (one action) vs. compound (bundles multiple actions)? | 15% |
| 4 | **Prerequisite Coherence** | Are its prerequisites logically ordered and complete? No circular or missing links? | 20% |
| 5 | **Age-Range Plausibility** | Does the `age_range` make sense for the skill's complexity? | 10% |
| 6 | **Graph Placement** | Is it under the right Domain → Strand → Concept hierarchy? Does it overlap with sibling skills? | 10% |

### Composite Score

```
composite = Σ (dimension_score × weight)
```

Each dimension is scored **1–5** (integer). Composite ranges from **1.0 to 5.0**.

### Review Status

Statuses are evaluated **in priority order** — the first matching rule wins:

| Priority | Status | Condition | Meaning |
|----------|--------|-----------|---------|
| 1 | `flagged` | composite < 2.5 **or** any dimension = 1 | Sent to auto-remediation; if fix fails or worsens score, remains flagged |
| 2 | `approved` | composite ≥ 4.0 **and** no dimension < 3 | Production-ready |
| 3 | `needs_fix` | everything else | Sent to auto-remediation |

After remediation + verification:

| Status | Meaning |
|--------|---------|
| `approved` | Originally passed, or was fixed and re-scored ≥ 4.0 |
| `auto_fixed` | LLM corrected the skill; re-score improved but didn't reach `approved` threshold |
| `flagged` | LLM could not fix the skill, or fix worsened the score |
| `error` | LLM call failed after retries |

Implementation:
```python
def classify(composite: float, dim_scores: list[int]) -> str:
    if composite < 2.5 or min(dim_scores) == 1:
        return "flagged"
    if composite >= 4.0 and min(dim_scores) >= 3:
        return "approved"
    return "needs_fix"
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

Evaluates placement across all 4 hierarchy levels: Domain → Strand → Concept → Skill.

**Note:** The graph uses a 4-level hierarchy (Domain → Strand → Concept → Skill), where Strand is optional — a Concept can BELONGS_TO a Domain directly if no Strand applies. The LLM evaluates placement considering all levels that exist.

| Score | Anchor |
|-------|--------|
| 5 | Correct domain, strand (if applicable), and concept; no overlap with sibling skills |
| 4 | Correct placement but mild overlap with one sibling (>0.85 embedding similarity) |
| 3 | Placement is defensible but a better concept or strand exists in the graph |
| 2 | Clearly belongs under a different concept or strand |
| 1 | Wrong domain entirely, or is a near-duplicate of another skill (>0.95 similarity) |

**Note on similarity threshold:** The pipeline uses a cosine similarity threshold of **0.85** (`embedding_similarity_threshold` in `settings.py`), not 0.7. The rubric anchors above are calibrated against this threshold.

---

## 4. Architecture

### Standalone Command, Not a Pipeline Stage

Review is a **separate command** that runs independently from the ingestion pipeline. It is NOT a pipeline stage.

**Why:** The ingestion pipeline (`python main.py run`) processes CSV files through stages 1-7 (scraping → extraction → graph construction → validation → completed). Review operates on the **entire subject's skill graph**, not on a single source document. Mixing them creates scoping problems — a new CSV's 300 skills might depend on 4700 old unreviewed skills, and wave ordering only works when ALL skills in the prerequisite chain are included.

**Separation of concerns:**
- `python main.py run` = ingest CSV data, build graph (stages 1-7, per source document)
- `python main.py review-skills` = review and fix skills (standalone, per subject, all skills)

```
Pipeline stages (unchanged):
  SCRAPING = 1
  PARSING = 2 (pass-through)
  EXTRACTION = 3
  HIERARCHY_BUILDING = 4 (pass-through)
  GRAPH_CONSTRUCTION = 5 (no-op)
  PREREQUISITE_INFERENCE = 6
  VALIDATION = 7
  COMPLETED = 8                ← no change
```

No `PipelineStage` enum changes. No `ProcessingState` enum changes. No `_stage_mapping` changes. The pipeline completes at stage 8 as before.

### State Manager Changes

Only one fix needed:

**`from_name()` mapping** — fix the existing missing `"prerequisite_inference"` key:
```python
name_to_stage = {
    "scraping": cls.SCRAPING,
    "parsing": cls.PARSING,
    "extraction": cls.EXTRACTION,
    "hierarchy_building": cls.HIERARCHY_BUILDING,
    "prerequisite_inference": cls.PREREQUISITE_INFERENCE,  # BUG FIX: currently missing
    "graph_construction": cls.GRAPH_CONSTRUCTION,
    "validation": cls.VALIDATION,
    "completed": cls.COMPLETED,
}
```

### Review Scope

The `review-skills` command always operates on a **full subject**. It reviews:
1. All skills with `review_score IS NULL` (never reviewed)
2. All skills with `review_stale = true` (context changed since last review)

All these skills — plus their full prerequisite chains — are included in wave computation. This ensures wave ordering is valid: every foundation skill is reviewed before its dependents.

### When to Run Review

```
Step 1: Ingest data (can be multiple runs)
  python main.py run --subject mathematics --sources "source_1"
  python main.py run --subject mathematics --sources "source_2"
  python main.py run --subject mathematics --sources "source_3"

Step 2: Review the full subject (once, after ingestion)
  python main.py review-skills --subject mathematics
  → Reviews ALL unreviewed + stale skills across the entire subject
  → Wave ordering covers the full prerequisite graph

Step 3: Add more data later
  python main.py run --subject mathematics --sources "source_4"
  → New 300 skills created, pipeline completes (stages 1-7)

Step 4: Review again
  python main.py review-skills --subject mathematics
  → Picks up 300 new unreviewed skills + any stale neighbors
  → Previously reviewed skills with review_score already set are skipped
  → Only ~350 skills to process, not 5000+
```

### Wave-Based Processing (Bottom-Up by Prerequisite Depth)

Skills are grouped into **waves by prerequisite depth** and processed bottom-up. This ensures prerequisites are always scored and fixed before the skills that depend on them.

**Depth computation (iterative, stored on Skill nodes):**

A single unbounded path query (`[:PREREQUISITE*]`) is too expensive for large graphs (5000+ skills with chains 20+ levels deep — half the graph gets stuck at the depth cap). Instead, compute depth iteratively, wave by wave, and store it as a `wave_depth` property on each Skill node.

**Step 1: Assign wave 0 (foundation skills with no prerequisites):**
```cypher
MATCH (s:Skill) WHERE NOT (s)-[:PREREQUISITE]->()
SET s.wave_depth = 0
RETURN count(s) AS wave_0_count
```

**Step 2: Assign subsequent waves (repeat until no more skills are assigned):**
```cypher
MATCH (s:Skill)-[:PREREQUISITE]->(p:Skill)
WHERE s.wave_depth IS NULL
WITH s, collect(p.wave_depth) AS prereq_depths
WHERE ALL(d IN prereq_depths WHERE d IS NOT NULL)
SET s.wave_depth = apoc.coll.max(prereq_depths) + 1
RETURN count(s) AS assigned
```

Each iteration of Step 2 computes the next wave. Run it in a loop until `assigned = 0`:

```python
def compute_wave_depths(graph_store):
    """Iteratively assign wave_depth to all skills."""
    # Clear existing depths for skills being reviewed
    graph_store.run("MATCH (s:Skill) WHERE s.review_score IS NULL OR s.review_stale = true REMOVE s.wave_depth")

    # Wave 0: no prerequisites
    result = graph_store.run("""
        MATCH (s:Skill) WHERE NOT (s)-[:PREREQUISITE]->() AND s.wave_depth IS NULL
        SET s.wave_depth = 0
        RETURN count(s) AS assigned
    """)
    logger.info(f"Wave 0: {result['assigned']} skills")

    # Subsequent waves: repeat until no more assignments
    wave = 1
    while True:
        result = graph_store.run("""
            MATCH (s:Skill)-[:PREREQUISITE]->(p:Skill)
            WHERE s.wave_depth IS NULL
            WITH s, collect(p.wave_depth) AS prereq_depths
            WHERE ALL(d IN prereq_depths WHERE d IS NOT NULL)
            SET s.wave_depth = apoc.coll.max(prereq_depths) + 1
            RETURN count(s) AS assigned
        """)
        if result['assigned'] == 0:
            break
        logger.info(f"Wave {wave}: {result['assigned']} skills")
        wave += 1

    # Fetch grouped results
    results = graph_store.run("""
        MATCH (s:Skill) WHERE s.wave_depth IS NOT NULL
        RETURN s.wave_depth AS depth, collect(s.id) AS skill_ids
        ORDER BY depth ASC
    """)
    return {r['depth']: r['skill_ids'] for r in results}
```

**Why iterative instead of single query:**
- The single path query (`[:PREREQUISITE*]`) explores every possible path for every skill — exponentially expensive with deep chains
- On production data (5000+ skills), chains go 30+ levels deep, causing timeouts or incorrect results when capped
- The iterative approach only looks at 1-hop prereqs per iteration — fast and guaranteed correct
- Each iteration takes milliseconds, and the total number of iterations = max chain depth

**`wave_depth` is stored as a Neo4j Skill property** so it can be:
- Reused across passes within a review run (no recomputation)
- Queried directly: `MATCH (s:Skill) WHERE s.wave_depth = 3 RETURN s`
- Cleared and recomputed at the start of each review run (in case prereq edges changed since last run)

**Concurrency safety within a wave:** Skills in the same wave have no prerequisite relationships with each other (by definition — if A depends on B, A is in a higher wave). So all skills in a wave can be scored, metadata-fixed, and structure-fixed in parallel without conflicts.

### Four-Pass Wave Execution

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def process_waves(skills_by_depth: dict[int, list], config):
    max_workers = config.review_concurrency  # default: 10

    for depth in sorted(skills_by_depth.keys()):
        wave = skills_by_depth[depth]
        logger.info(f"Wave {depth}: {len(wave)} skills")

        # ── Pass 1: Score (read-only, concurrent) ──────────────────
        scores = {}
        with ThreadPoolExecutor(max_workers=max_workers) as pool:
            futures = {pool.submit(score_skill, s): s for s in wave}
            for f in as_completed(futures):
                result = f.result()
                scores[result['skill_id']] = result

        # Identify skills needing fixes
        needs_metadata_fix = [
            s for s in scores.values()
            if s['status'] in ('needs_fix', 'flagged')
            and any(s[d] < 3 for d in ['clarity', 'assessability', 'granularity',
                                        'age_range_plausibility'])
        ]
        needs_structural_fix = [
            s for s in scores.values()
            if s['status'] in ('needs_fix', 'flagged')
            and s['prerequisite_coherence'] < 3
        ]

        if not needs_metadata_fix and not needs_structural_fix:
            write_all_scores(scores)
            continue

        # ── Pass 2: Fix metadata (concurrent) ─────────────────────
        # Title, description, age_range only. No graph structural changes.
        if needs_metadata_fix:
            with ThreadPoolExecutor(max_workers=max_workers) as pool:
                futures = {pool.submit(fix_metadata, s): s for s in needs_metadata_fix}
                for f in as_completed(futures):
                    fix = f.result()
                    if fix['changes_made']:
                        apply_metadata_to_neo4j(fix)  # title, desc, age_range
                        re_embed_skill(fix)            # new embedding for new title

        # ── Pass 3: Fix structure (concurrent within wave) ────────
        # Prerequisite edge add/remove. Sees already-fixed metadata.
        # Each proposed change goes through deterministic validation.
        if needs_structural_fix:
            # Re-fetch context — skills now have fixed titles from Pass 2
            refreshed = fetch_skills_with_context(
                [s['skill_id'] for s in needs_structural_fix]
            )
            with ThreadPoolExecutor(max_workers=max_workers) as pool:
                futures = {pool.submit(fix_structure, s): s for s in refreshed}
                for f in as_completed(futures):
                    fix = f.result()
                    if fix['changes_made']:
                        # Each edge change goes through validation gate
                        validated_changes = validate_all_edge_changes(fix)
                        if validated_changes:
                            apply_edge_changes_to_neo4j(validated_changes)

        # ── Pass 4: Verify (concurrent) ───────────────────────────
        # Re-score ALL fixed skills with fresh context.
        all_fixed_ids = set()
        if needs_metadata_fix:
            all_fixed_ids |= {s['skill_id'] for s in needs_metadata_fix
                              if s.get('fix_applied')}
        if needs_structural_fix:
            all_fixed_ids |= {s['skill_id'] for s in needs_structural_fix
                              if s.get('fix_applied')}

        if all_fixed_ids:
            fixed_skills = fetch_skills_from_neo4j(list(all_fixed_ids))
            with ThreadPoolExecutor(max_workers=max_workers) as pool:
                futures = {pool.submit(score_skill, s): s for s in fixed_skills}
                for f in as_completed(futures):
                    new_score = f.result()
                    old_score = scores[new_score['skill_id']]
                    if new_score['composite'] < old_score['composite']:
                        # Fix made it worse — revert everything
                        revert_skill(new_score['skill_id'], old_score)
                        scores[new_score['skill_id']]['status'] = 'flagged'
                    else:
                        scores[new_score['skill_id']] = new_score
                        scores[new_score['skill_id']]['status'] = (
                            'approved' if new_score['composite'] >= 4.0
                            and min(new_score['dim_scores']) >= 3
                            else 'auto_fixed'
                        )

        # Mark top-10 similar siblings stale if any titles changed
        mark_top10_siblings_stale(wave, all_fixed_ids)

        # Write final scores for this wave
        write_all_scores(scores)

    # ── After all waves ────────────────────────────────────────────
    if any_structural_fixes:
        graph_store.derive_concept_prerequisites()  # re-derive if edges changed
```

### Structural Fix Validation Gates

Before any prerequisite edge change is applied, it must pass **deterministic code checks** — not LLM checks, not human approval. These are fast, reliable, and catch the most common LLM mistakes:

```python
def validate_edge_change(change: dict, graph_store) -> tuple[bool, str]:
    """Deterministic validation. Returns (allowed, reason)."""

    skill_id = change['skill_id']
    target_id = change['target_skill_id']
    action = change['action']  # "add" or "remove"

    # 1. Target skill must exist
    if not graph_store.node_exists('Skill', target_id):
        return False, f"target skill {target_id} not found"

    # 2. No self-loop
    if target_id == skill_id:
        return False, "self-loop"

    # 3. No cycle (only for add operations)
    if action == 'add':
        if graph_store.would_create_cycle(skill_id, target_id):
            return False, "would create cycle"

    # 4. Age consistency: prerequisite age should be ≤ dependent age
    if action == 'add':
        prereq = graph_store.get_skill(target_id)
        dependent = graph_store.get_skill(skill_id)
        if prereq and dependent and prereq.get('age_range') and dependent.get('age_range'):
            if min(prereq['age_range']) > max(dependent['age_range']):
                return False, (
                    f"prereq min age {min(prereq['age_range'])} > "
                    f"dependent max age {max(dependent['age_range'])}"
                )

    # 5. Same domain: cross-domain prerequisites are almost always wrong
    if action == 'add':
        prereq_domain = graph_store.get_skill_domain(target_id)
        dependent_domain = graph_store.get_skill_domain(skill_id)
        if prereq_domain and dependent_domain and prereq_domain != dependent_domain:
            return False, f"cross-domain prereq ({prereq_domain} -> {dependent_domain})"

    return True, "ok"
```

**What each check catches:**
| Check | What it prevents | Frequency |
|-------|-----------------|-----------|
| Target exists | Hallucinated skill IDs | Common with LLMs |
| No self-loop | Skill depending on itself | Rare but catastrophic |
| No cycle | A→B→C→A circular chains | Occasional |
| Age consistency | 12-year-old skill requiring 15-year-old skill | Common LLM mistake |
| Same domain | Math skill requiring a Science skill | Occasional |

Edge changes that fail validation are **rejected individually** — the rest of the fix still applies. If ALL edge changes for a skill are rejected, the skill keeps its original prerequisites and is marked based on Pass 4 re-score.

### Bounded Context Per Skill (1-Hop, Not Full Trees)

During extraction, 1 curriculum item fans out to 20 matches → 18 trees → ~1200 skills of context via bidirectional DFS. **The review stage does NOT need this.** Each scoring dimension requires only shallow context:

| Dimension | Context Needed | NOT Needed |
|-----------|---------------|------------|
| Clarity | Just the skill itself | — |
| Assessability | Just the skill itself | — |
| Granularity | Just the skill itself | — |
| Prerequisite Coherence | Direct prerequisites + direct dependents (1 hop) | Full transitive tree |
| Age-Range Plausibility | Skill's age_range + direct prerequisites' age_ranges | Deep chains |
| Graph Placement | Top-10 similar sibling skills (same concept) | Full tree |

**Context bounds:**
- **Direct prerequisites**: 1 hop via `(s)-[:PREREQUISITE]->(p)` — typically 1–3 skills
- **Direct dependents**: 1 hop via `(d)-[:PREREQUISITE]->(s)` — typically 1–5 skills
- **Siblings**: Top 10 most similar siblings **by embedding similarity** (fetched from `skill_embeddings` in PostgreSQL, then hydrated from Neo4j). For concepts with 30+ skills, include count: "...and 22 more siblings"
- **Parent hierarchy**: Domain → Strand (if any) → Concept — always 2–3 nodes

**Total context per skill: ~20–25 skills max**, regardless of graph size.

**Sibling selection query** (PostgreSQL, not Neo4j — because embeddings live in pgvector):
```sql
-- Find top-10 most similar siblings for a skill
SELECT se2.neo4j_skill_id, se2.skill_title, se2.embedding_text,
       1 - (se1.embedding <=> se2.embedding) AS similarity
FROM skill_embeddings se1
JOIN skill_embeddings se2
  ON se2.concept_title = se1.concept_title
  AND se2.neo4j_skill_id <> se1.neo4j_skill_id
WHERE se1.neo4j_skill_id = $skill_id
ORDER BY se1.embedding <=> se2.embedding ASC
LIMIT 10
```

Then hydrate from Neo4j:
```cypher
MATCH (s:Skill) WHERE s.id IN $sibling_ids
RETURN s.id, s.title, s.description, s.age_range
```

**Full context fetch (Neo4j portion):**
```cypher
MATCH (s:Skill {id: $skill_id})
MATCH (s)-[:BELONGS_TO]->(c:Concept)
OPTIONAL MATCH (c)-[:BELONGS_TO]->(st:Strand)-[:BELONGS_TO]->(d:Domain)
OPTIONAL MATCH (c)-[:BELONGS_TO]->(d2:Domain)
WITH s, c, st, COALESCE(d, d2) AS domain

// Direct prerequisites (1 hop)
OPTIONAL MATCH (s)-[pr:PREREQUISITE]->(prereq:Skill)
WITH s, c, st, domain, collect({
  id: prereq.id, title: prereq.title, age_range: prereq.age_range,
  order: pr.order, reasoning: pr.reasoning
}) AS prerequisites

// Direct dependents (1 hop)
OPTIONAL MATCH (dep:Skill)-[dr:PREREQUISITE]->(s)
WITH s, c, st, domain, prerequisites, collect({
  id: dep.id, title: dep.title
}) AS dependents

// Total sibling count
OPTIONAL MATCH (sib:Skill)-[:BELONGS_TO]->(c) WHERE sib.id <> s.id
WITH s, c, st, domain, prerequisites, dependents, count(sib) AS sibling_count

RETURN s, c, st, domain, prerequisites, dependents, sibling_count
```

### Cascade Handling

When a skill's **title changes** in Pass 2, its neighbors may need re-scoring:

**Within a single run (handled by wave ordering):**
- A fixed skill in Wave N has its dependents in Wave N+1 or higher. Those waves haven't been scored yet, so they automatically see the fixed version. No stale marking needed.
- A fixed skill's **top-10 similar siblings** (same concept, same wave) may have already been scored in this wave. Their Graph Placement scores could be affected. Mark only these top-10 siblings as `review_stale = true` for the next run — NOT all siblings in the concept.

**Why only top-10, not all siblings?** A concept like "Counting" has 222 skills. If one skill's title changes, marking all 221 siblings stale would be too aggressive — most have low similarity and their Graph Placement score wouldn't be affected. The top-10 most similar siblings (the ones actually sent to the LLM during scoring) are the only ones whose scores could change.

**Across runs:**
- The `review_stale` flag catches anything missed.
- Convergence is fast: typically **2–3 runs** to reach a stable state.

```
Run 1: Score all → Fix ~20% → Mark stale siblings → Done
         Dependents in higher waves already see fixed prereqs.
Run 2: Re-score ~5% stale siblings → Fix a few → Mark stale → Done
Run 3: Re-score ~1% stale → Almost nothing changes → Stable
```

### Concept-Level Prerequisite Re-Derivation

Concept → Concept PREREQUISITE edges are derived bottom-up from skill prerequisites (in `graph_store.derive_concept_prerequisites()`). If Pass 3 adds or removes any skill prerequisite edges, concept prereqs may become stale.

**Resolution:** After all waves complete, re-run `derive_concept_prerequisites()`. This mirrors how the existing pipeline runs it after graph construction (Stage 6: Prerequisite Inference). Only runs if any structural fixes were actually applied.

### Skill ID Immutability

Skill IDs are deterministic SHA1 hashes: `skill_` + SHA1(`domain|strand|concept|skill_title`)[:12]. If auto-remediation changes a skill's title, the hash would produce a different ID — but we cannot change a node's ID without delete+recreate.

**Rule: Skill IDs are immutable.** Auto-remediation updates title, description, age_range, and prerequisite edges on the existing node but never changes the node ID. The ID reflects the skill's identity at creation time, not its current title. This is consistent with how `update_existing` works in the extraction stage.

### Embedding Drift After Remediation

When Pass 2 changes a skill's title, the skill's embedding must be recomputed and upserted in `skill_embeddings`. This means future extraction runs (for new CSV sources) will match against the improved embedding.

**This is desirable** — a better title produces a more accurate embedding, leading to better deduplication and prerequisite matching for future items.

### Remediation Scope Limits

Passes 2 and 3 **can** change: title, description, age_range, prerequisite edges (add/remove).

Passes 2 and 3 **cannot** change:
- **Parent concept, strand, or domain** — moving a skill between concepts requires BELONGS_TO edge rewiring, concept prerequisite re-derivation, and potentially ID migration. This is graph restructuring, out of scope.
- **Skill ID** — immutable.

**Consequence:** If a skill's only problem is Graph Placement (score 1 or 2 — wrong concept), remediation cannot fix it. The skill stays `flagged`.

### Revert Mechanism

Before applying any fix, the original values are snapshotted in the `metadata_fix` and `structural_fix` JSONB columns. If Pass 4 (Verify) shows the composite score is worse or any dimension dropped, revert:

```python
def revert_skill(skill_id: str, original: dict):
    """Restore pre-fix state from snapshot."""
    graph_store.update_skill(
        skill_id=skill_id,
        title=original['original_title'],
        description=original['original_description'],
        age_range=original['original_age_range'],
    )
    # Restore original prerequisites if structural changes were made
    if original.get('prerequisite_changes'):
        for change in original['prerequisite_changes']:
            if change['action'] == 'add':
                # We added this edge — remove it
                graph_store.remove_prerequisite_edge(skill_id, change['target_skill_id'])
            elif change['action'] == 'remove':
                # We removed this edge — re-add it
                graph_store.create_prerequisite_relationship(
                    skill_id, change['target_skill_id'],
                    order=change.get('original_order', 1),
                    reasoning=change.get('original_reasoning', ''),
                )
    # Re-embed with original title
    re_embed_skill(skill_id, original['original_title'])
```

---

## 5. Storage Schema

### Neo4j: Skill Node Properties (new)

```cypher
// Added to every Skill node after scoring
skill.review_score       // Float: composite 1.0-5.0
skill.review_status      // String: "approved" | "auto_fixed" | "flagged" | "error"
skill.reviewed_at        // DateTime: when last scored
skill.review_version     // Integer: scoring rubric version (starts at 1)
skill.review_stale       // Boolean: true when graph context changed since last review
skill.wave_depth         // Integer: prerequisite depth (0 = foundation, computed iteratively)
```

These properties enable Cypher queries like:
```cypher
// Find all flagged skills
MATCH (s:Skill) WHERE s.review_status = 'flagged' RETURN s

// Average score by concept
MATCH (s:Skill)-[:BELONGS_TO]->(c:Concept)
RETURN c.title, avg(s.review_score) AS avg_score
ORDER BY avg_score ASC

// Skills that were auto-fixed
MATCH (s:Skill) WHERE s.review_status = 'auto_fixed'
RETURN s.title, s.review_score

// Flagged skills under a domain (handles optional Strand)
MATCH (s:Skill)-[:BELONGS_TO]->(c:Concept)
WHERE s.review_status = 'flagged'
  AND ( (c)-[:BELONGS_TO]->(:Domain {title: 'Mathematics'})
     OR (c)-[:BELONGS_TO]->(:Strand)-[:BELONGS_TO]->(:Domain {title: 'Mathematics'}) )
RETURN s.title, s.review_score

// Source quality tracking (source_names already exists on all node types)
MATCH (s:Skill)
UNWIND s.source_names AS source
RETURN source, avg(s.review_score) AS avg_quality, count(s) AS skill_count
ORDER BY avg_quality ASC
```

### PostgreSQL: Skill Review Detail Table (new)

```sql
CREATE TABLE skill_reviews (
    id SERIAL PRIMARY KEY,
    neo4j_skill_id VARCHAR(50) NOT NULL,
    review_version INTEGER NOT NULL DEFAULT 1,

    -- Per-dimension scores (1-5)
    clarity_score INTEGER NOT NULL,
    assessability_score INTEGER NOT NULL,
    granularity_score INTEGER NOT NULL,
    prerequisite_coherence_score INTEGER NOT NULL,
    age_range_plausibility_score INTEGER NOT NULL,
    graph_placement_score INTEGER NOT NULL,

    -- Composite
    composite_score FLOAT NOT NULL,
    review_status VARCHAR(20) NOT NULL,  -- approved | auto_fixed | flagged | error

    -- Reasoning (from LLM)
    dimension_reasoning JSONB NOT NULL,
    /*
      {
        "clarity": "Title is a complete verb phrase with specific object...",
        "assessability": "Uses Bloom's verb 'solve' with measurable condition...",
        "granularity": "Single atomic action with one verb and one object",
        "prerequisite_coherence": "3 prerequisites, correctly ordered, no cycles",
        "age_range_plausibility": "Ages [10,11] matches grade 5 expectations for this complexity",
        "graph_placement": "Correctly placed under Number Sense > Fraction Addition; no significant sibling overlap"
      }
    */

    -- Auto-remediation: metadata fix (Pass 2)
    metadata_fix JSONB,
    /*
      {
        "original_title": "Fraction stuff",
        "new_title": "Add fractions with unlike denominators",
        "original_description": "Learn about fractions",
        "new_description": "Add two fractions with different denominators by finding a common denominator",
        "original_age_range": [5, 15],
        "new_age_range": [10, 11]
      }
    */

    -- Auto-remediation: structural fix (Pass 3)
    structural_fix JSONB,
    /*
      {
        "prerequisite_changes": [
          {"action": "add", "target_skill_id": "skill_abc123", "order": 1,
           "reasoning": "...", "validation_result": "ok"},
          {"action": "remove", "target_skill_id": "skill_def456",
           "reasoning": "...", "validation_result": "ok"},
          {"action": "add", "target_skill_id": "skill_ghost",
           "reasoning": "...", "validation_result": "target skill not found", "rejected": true}
        ]
      }
    */

    -- Verification (Pass 4)
    pre_fix_composite FLOAT,              -- composite before any fixes
    post_fix_composite FLOAT,             -- composite after fixes + re-score
    fix_reverted BOOLEAN DEFAULT false,   -- true if re-score was worse and we reverted

    -- Metadata
    llm_model VARCHAR(100),
    review_run_id VARCHAR(100),           -- links reviews from the same pipeline run
    wave_depth INTEGER,                   -- prerequisite depth level
    created_at TIMESTAMP DEFAULT NOW(),

    UNIQUE (neo4j_skill_id, review_version)
);

CREATE INDEX idx_skill_reviews_skill ON skill_reviews(neo4j_skill_id);
CREATE INDEX idx_skill_reviews_status ON skill_reviews(review_status);
CREATE INDEX idx_skill_reviews_run ON skill_reviews(review_run_id);
```

---

## 6. LLM Prompt Design

### Pass 1: Scoring Prompt (Per Skill)

Each LLM call reviews **1 skill** with its bounded graph context. The prompt includes:

1. The complete scoring rubric (all 6 dimensions with anchors)
2. The skill being scored:
   - Skill title, description, age_range
   - Parent hierarchy: Domain → Strand (if applicable) → Concept
3. Bounded graph context (1-hop only):
   - Direct prerequisite skill titles (ordered, with reasoning) — typically 1–3
   - Direct dependent skill titles — typically 1–5
   - Top 10 most similar sibling skills (by embedding similarity, from pgvector) with titles + age_ranges
   - Total sibling count if >10

### Pass 1: Response Schema

```json
{
  "skill_id": "skill_abc123def4",
  "clarity_score": 5,
  "clarity_reasoning": "Complete verb phrase with specific object and condition",
  "assessability_score": 4,
  "assessability_reasoning": "Strong Bloom's verb 'solve', object could be more specific",
  "granularity_score": 5,
  "granularity_reasoning": "Single atomic action — one verb, one object, one condition",
  "prerequisite_coherence_score": 4,
  "prerequisite_coherence_reasoning": "Prerequisites are correct and ordered; one intermediate step could be added between 'Identify fractions' and 'Add fractions with like denominators'",
  "age_range_plausibility_score": 5,
  "age_range_plausibility_reasoning": "Ages [10,11] aligns with grade 5 curriculum standards for fraction addition",
  "graph_placement_score": 4,
  "graph_placement_reasoning": "Correctly placed under Number Sense > Fraction Addition concept; mild overlap with sibling 'Add fractions with like denominators' but distinct enough"
}
```

### Pass 2: Metadata Fix Prompt (Per Failing Skill)

Only called for skills where Clarity, Assessability, Granularity, or Age-Range Plausibility scored < 3. The prompt includes:

1. The skill's current title, description, age_range
2. The Phase 1 scores and reasoning for the failing dimensions
3. The parent hierarchy and sibling context
4. **Instruction: Fix only title, description, and/or age_range. Do NOT propose prerequisite changes.**

### Pass 2: Response Schema

```json
{
  "skill_id": "skill_abc123def4",
  "changes_made": true,
  "updated_title": "Add fractions with unlike denominators",
  "updated_description": "Add two fractions with different denominators by finding a common denominator",
  "updated_age_range": [10, 11],
  "fix_reasoning": "Original title was a noun phrase ('Fraction operations'). Rewrote as a specific verb-phrase skill. Narrowed age range from [5,15] to [10,11] based on curriculum standards."
}
```

### Pass 3: Structural Fix Prompt (Per Skill with Prereq Problems)

Only called for skills where Prerequisite Coherence scored < 3. This call sees the **already-fixed metadata** from Pass 2 (updated title/description). The prompt includes:

1. The skill's current state (with any Pass 2 fixes already applied)
2. The Prerequisite Coherence score and reasoning from Pass 1
3. The skill's current direct prerequisites and dependents
4. The parent hierarchy and sibling context
5. **Instruction: Propose only prerequisite edge add/remove operations. Do NOT change title, description, or age_range.**

### Pass 3: Response Schema

```json
{
  "skill_id": "skill_abc123def4",
  "changes_made": true,
  "prerequisite_changes": [
    {
      "action": "add",
      "target_skill_id": "skill_xyz789",
      "order": 1,
      "reasoning": "Must understand equivalent fractions before adding unlike denominators"
    },
    {
      "action": "remove",
      "target_skill_id": "skill_def456",
      "reasoning": "This is a peer skill in the same concept, not a prerequisite"
    }
  ],
  "fix_reasoning": "Added missing intermediate prerequisite. Removed incorrect same-concept dependency."
}
```

### Cost Estimate

**Per-skill token usage (based on ~15 context skills: 1.78 prereqs + 1.78 dependents + 10 siblings + hierarchy):**
- **Pass 1 (scoring)**: ~2,500 input + ~300 output
- **Pass 2 (metadata fix, ~20% of skills)**: ~1,500 input + ~200 output
- **Pass 3 (structural fix, ~10% of skills)**: ~2,000 input + ~250 output
- **Pass 4 (re-score, ~25% of skills)**: ~2,500 input + ~300 output

**At production scale (4,083 skills, avg 1.78 prereqs/dependents per skill):**

| Pass | Calls | Input Tokens | Output Tokens |
|------|-------|-------------|--------------|
| Pass 1 (score all) | 4,083 | 10.2M | 1.2M |
| Pass 2 (20% metadata fix) | 817 | 1.2M | 0.16M |
| Pass 3 (10% structural fix) | 408 | 0.82M | 0.10M |
| Pass 4 (25% verify) | 1,021 | 2.55M | 0.31M |
| **Total** | **6,329** | **~14.8M** | **~1.8M** |

**Cost comparison across models (via OpenRouter):**

| Model | Input Rate | Output Rate | Input Cost | Output Cost | **Total** |
|-------|-----------|-------------|-----------|-------------|-----------|
| Gemini 2.5 Flash | $0.30/M | $2.50/M | $4.44 | $4.50 | **~$8.94** |
| Gemini 2.5 Pro | $1.25/M | $10.00/M | $18.50 | $18.00 | **~$36.50** |
| Gemini 3.1 Pro Preview | $2.00/M | $12.00/M | $29.60 | $21.60 | **~$51.20** |

**Time estimate**: ~15 min at 10 concurrent workers (with wave barriers) for 4,083 skills.

The pipeline currently uses **Gemini 3.1 Pro Preview via OpenRouter** for skill extraction. The `review_llm_model` config option (see Section 7) allows using a different model for review/remediation — Gemini 2.5 Flash offers the best cost-to-quality ratio for review tasks.

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
| New processor | `src/processors/skill_reviewer.py` | **New file.** Entry point for the `review-skills` CLI command. Pass 1: LLM scoring with bounded context. Pass 2: metadata fix (title/desc/age). Pass 3: structural fix (prereq edges) with validation gates. Pass 4: re-score + revert-if-worse. Wave orchestration with ThreadPoolExecutor |
| Graph store | `src/storage/graph_store.py` | Add `update_skill_review_scores()`. Add `get_skill_with_bounded_context()` (1-hop prereqs/deps + hierarchy). Add `apply_metadata_fix()` (title/desc/age update). Add `apply_structural_fix()` (prereq edge changes). Add `compute_wave_depths()` (iterative wave computation, stores `wave_depth` on Skill nodes). Add `would_create_cycle()` (cycle detection for validation gate). Add `get_skill_domain()` (domain lookup for cross-domain check) |
| Embedding store | `src/storage/embedding_store.py` | Add `skill_reviews` table creation in `ensure_tables()`. Add `insert_skill_review()` and `get_skill_reviews()`. Add `get_similar_siblings()` (pgvector similarity query for sibling context) |
| LLM client | `src/processors/llm_client.py` | No changes — existing `call_llm()` with JSON repair handles all three prompt types |
| CLI | `src/cli/main.py` | Add `review-skills` command as a standalone Click command (not a pipeline stage). Flags: `--subject`, `--rescore`, `--score-only`, `--summary`, `--status` |
| Config | `src/config/settings.py` | Add `review_llm_model`, `review_llm_base_url`, `review_llm_api_key` (optional overrides). Add `review_concurrency` (default: 10). Add `review_llm_config` property |
| State manager | `src/pipeline/state_manager.py` | **Bug fix only:** add missing `"prerequisite_inference"` key to `from_name()`. No new stages or states |

### What Does NOT Change

- **Pipeline stages 1–8** — review is a standalone command, not modifying the ingestion pipeline at all
- **`PipelineStage` enum, `ProcessingState` enum, `_stage_mapping`** — untouched (no new stages or states)
- **Orchestrator** (`src/pipeline/orchestrator.py`) — untouched
- Graph schema (Domain, Strand, Concept nodes) — only Skill nodes get new properties
- Embedding model or similarity search — reuses existing `skill_embeddings` for sibling comparison
- API and visualization — no changes required (but could consume scores in future)
- `sources.yaml` / `subjects.yaml` — untouched

---

## 8. CLI Commands

Review is a **standalone command**, completely separate from the ingestion pipeline.

### Pipeline (unchanged)

```bash
# Normal ingestion pipeline — stages 1-7, no review
python main.py run --subject mathematics
python main.py run --subject mathematics --sources "ixl.com/math"
python main.py run --subject mathematics --rows 100
```

### Review (standalone)

```bash
# Review all unreviewed + stale skills in the subject
python main.py review-skills --subject mathematics

# Re-score everything (ignore existing scores)
python main.py review-skills --subject mathematics --rescore

# Score only, skip auto-remediation (diagnostic mode)
python main.py review-skills --subject mathematics --score-only

# Show review summary
python main.py review-skills --subject mathematics --summary
# Output:
#   Total skills: 5000
#   Reviewed: 4650 (93.0%)
#     Approved: 3720 (80.0%)
#     Auto-fixed: 558 (12.0%)
#       Metadata only: 420
#       Structural only: 46
#       Metadata + structural: 92
#     Flagged: 232 (5.0%)
#       Fix worsened score (reverted): 93
#       Graph placement (unfixable): 46
#       All fixes rejected by validation: 23
#       LLM couldn't fix: 70
#     Error: 140 (3.0%)
#   Unreviewed: 350 (7.0%)
#   Average composite: 4.12
#   Lowest dimension (avg): Prerequisite Coherence (3.41)
#   Waves processed: 8 (max depth: 7)
#   Structural fixes: 150 proposed, 120 applied, 30 rejected
#     Rejections: 18 hallucinated IDs, 7 would create cycle, 5 age inconsistent
#   Source quality:
#     in.ixl.com/maths: avg 4.31 (1200 skills)
#     ixl.com/math: avg 3.89 (1500 skills)
#     embibe.com/maths: avg 4.05 (1000 skills)

# List flagged skills
python main.py review-skills --subject mathematics --status flagged

# List auto-fixed skills with change details
python main.py review-skills --subject mathematics --status auto_fixed
```

### Typical Workflow

```bash
# 1. Ingest multiple sources
python main.py run --subject mathematics --sources "source_1"
python main.py run --subject mathematics --sources "source_2"

# 2. Review everything
python main.py review-skills --subject mathematics

# 3. Add more data later
python main.py run --subject mathematics --sources "source_3"

# 4. Review again (only new + stale skills processed)
python main.py review-skills --subject mathematics
```

---

## 9. Data Flow

```
review-skills --subject mathematics
======================================================================

Step 0 — Select Skills & Compute Waves
----------------------------------------------------------------------
Select ALL skills in the subject that need review:
  - review_score IS NULL  (never reviewed)
  - review_stale = true   (context changed since last review)

Compute prerequisite depth for selected skills -> group into waves:
  Wave 0: [skill_a, skill_b, skill_c, ...]   (foundations, no prereqs)
  Wave 1: [skill_d, skill_e, ...]             (depend on wave 0 only)
  Wave 2: [skill_f, ...]                      (depend on wave 0+1)
  ...

For each wave (sequential, with barrier between waves):
======================================================================

Pass 1 — Score (read-only, concurrent within wave)
----------------------------------------------------------------------
  For each skill (10 threads):
    1. Fetch bounded context:
       - 1-hop prereqs + dependents from Neo4j
       - Top-10 similar siblings from pgvector (by embedding similarity)
       - Domain/Strand/Concept hierarchy
    2. LLM scores 6 dimensions (1-5) + reasoning
    3. Compute composite, classify: approved | needs_fix | flagged

  ─── BARRIER: all skills in wave scored ───

  Skills classified "approved" -> done for this wave.

Pass 2 — Fix Metadata (concurrent within wave)
----------------------------------------------------------------------
  For skills with Clarity, Assessability, Granularity, or Age-Range < 3:
    1. LLM proposes: new title, description, age_range
       (sees current state, does NOT propose prereq changes)
    2. Apply to Neo4j
    3. Re-embed in pgvector (new title = new embedding)

  ─── BARRIER: all metadata fixes applied ───

Pass 3 — Fix Structure (concurrent within wave)
----------------------------------------------------------------------
  For skills with Prerequisite Coherence < 3:
    1. Re-fetch context (sees already-fixed metadata from Pass 2)
    2. LLM proposes: prereq edge add/remove operations
    3. Each edge change -> deterministic validation gate:
       - Target exists? No self-loop? No cycle?
       - Age consistent? Same domain?
    4. Apply only validated changes to Neo4j
    5. Log rejected changes with reason

  ─── BARRIER: all structural fixes applied ───

Pass 4 — Verify (concurrent within wave)
----------------------------------------------------------------------
  For ALL skills that were fixed in Pass 2 or Pass 3:
    1. Re-score with fresh context (1 LLM call per skill)
    2. Compare new composite vs original composite:
       - Better or same -> keep fix, assign: approved | auto_fixed
       - Worse -> REVERT all changes, mark: flagged

  Mark same-wave siblings of fixed skills as review_stale = true.
  Write final scores for this wave.

  ─── BARRIER: wave complete, next wave starts ───

======================================================================

After all waves:
----------------------------------------------------------------------
  1. If any structural fixes were applied:
     -> Re-derive concept prerequisites (derive_concept_prerequisites())
  2. Write all review records to PostgreSQL skill_reviews table
  3. Log summary: approved/auto_fixed/flagged/error counts, wave stats
```

---

## 10. Scope

Review is a **standalone command**, fully automated, no human intervention required. It scores skills bottom-up by prerequisite depth, fixes metadata first, then fixes structure with validation gates, and verifies every fix. The entire process is LLM-driven with deterministic safety checks.

**Review is separate from the ingestion pipeline.** The ingestion pipeline (`python main.py run`) handles stages 1-7 (CSV → graph). Review (`python main.py review-skills`) runs independently on the full subject's skill graph afterward.

The workflow:
1. Run `review-skills --subject mathematics`.
2. All unreviewed + stale skills in the subject are selected.
3. Skills are grouped into waves by prerequisite depth.
4. Each wave runs 4 passes: Score → Fix Metadata → Fix Structure → Verify.
5. Wave barriers ensure prerequisites are fixed before their dependents are scored.
6. After all waves: concept prerequisites are re-derived if structural changes were made.

**Guardrails:**
- **Metadata fixes** (title, description, age_range) are self-contained and don't affect graph structure. Safe to apply directly, verified by re-score.
- **Structural fixes** (prerequisite edges) go through 5 deterministic validation checks before applying. Invalid changes are rejected individually.
- **Revert-if-worse**: Any fix that decreases the composite score is fully reverted (metadata + structural), and the skill is marked `flagged`.
- **Immutable IDs**: Skill IDs never change, regardless of title changes.
- **No concept moves**: Remediation cannot move a skill to a different concept/strand/domain.
- **Incremental**: Only unreviewed + stale skills are processed. Previously approved skills are skipped.
- **Score-only mode**: `--score-only` skips Passes 2-4 entirely for diagnostic-only runs.
- **Bounded context**: 1-hop prereqs/deps + top-10 siblings. No risk of token limit blowout.
- **Audit trail**: Every score, fix proposal, validation result, revert, and re-score is recorded in `skill_reviews`.

---

## 11. Error Handling

### LLM Call Failures (Pass 1 — Scoring)

| Attempt | Action |
|---------|--------|
| 1 | Retry with existing `llm_client` retry/backoff logic (up to 3 retries with exponential backoff) |
| 2 | If all retries fail, mark the skill as `review_status = 'error'` and continue |

Failed skills are logged and retried on the next run (`review_score IS NULL`).

### Malformed LLM Response

If the LLM returns valid JSON but is missing dimension scores:
- Log a warning with the skill ID and missing dimensions.
- Retry once.
- If still incomplete, mark as `error` and continue.

### LLM Call Failures (Pass 2 / Pass 3)

If a metadata or structural fix call fails:
- The skill keeps its Pass 1 scores unchanged.
- It remains `needs_fix` or `flagged` (not worsened).
- Logged for retry on next run.

### Structural Fix Validation Rejections

Individual edge changes that fail validation are rejected and logged, but the rest of the fix still applies. If ALL edge changes are rejected:
- The skill's prerequisites are unchanged.
- The skill is scored based on Pass 4 re-score (which may still improve from metadata fixes alone).

### Revert Safety

If Pass 4 shows the composite score is worse:
- All metadata changes are reverted (original title, description, age_range restored).
- All structural changes are reverted (added edges removed, removed edges re-added).
- Original embedding is restored.
- Skill is marked `flagged` with `fix_reverted = true` in the review record.

### Neo4j / PostgreSQL Write Failures

If the score write to Neo4j or PostgreSQL fails after successful scoring:
- Log the error with the skill ID and computed scores.
- Do **not** mark the skill as reviewed (picked up again on re-run).
- Continue processing remaining skills.

---

## 12. Incremental Behavior

### When the graph changes between reviews

The ingestion pipeline may add/update skills between review runs. Review scores can become **stale**:

| Change | Affected Dimensions | Action |
|--------|-------------------|--------|
| New skill added to the same concept (pipeline run) | Graph Placement (sibling overlap may change) | Mark top-10 similar siblings' `review_stale = true` |
| Skill's prerequisites change (pipeline run) | Prerequisite Coherence | Mark affected skill's `review_stale = true` |
| Skill gets `update_existing` from a later CSV item | All dimensions | Clear `review_score` (force re-score) |

These stale markings happen during the **ingestion pipeline** (stages 1-7). The next `review-skills` run picks them up automatically.

### Staleness from auto-remediation (within a review run)

Within a review run, when a skill is fixed:
- **Dependents** (higher wave): not yet scored → they see the fixed version naturally.
- **Top-10 similar siblings** (same wave, already scored): mark `review_stale = true` for next review run. Not all siblings in the concept — only the 10 most similar by embedding.
- **Prerequisites** (lower wave, already scored): their scores remain valid.

### Convergence

| Review run | What happens | Expected stale % |
|------------|-------------|-------------------|
| 1 | Score all unreviewed → Fix ~20% → Mark stale siblings | ~5% marked stale |
| 2 | Re-score stale siblings → Fix a few | ~1% marked stale |
| 3 | Almost nothing left | Stable |

At ~$10 per full run, convergence in 2–3 review runs costs ~$20–30 total for a stable, high-quality graph.

### Typical lifecycle

```
Pipeline run 1: ingest source_1 → 1000 skills created
Pipeline run 2: ingest source_2 → 1500 skills created
Review run 1:   review-skills   → 2500 skills scored, ~500 fixed, ~125 stale
Review run 2:   review-skills   → 125 stale re-scored, ~25 fixed, ~6 stale
Pipeline run 3: ingest source_3 → 300 new skills, ~50 existing siblings marked stale
Review run 3:   review-skills   → 350 skills to process (300 new + 50 stale)
```

---

## 13. Rubric Version Migration

The `review_version` field (on both Neo4j Skill nodes and the `skill_reviews` PostgreSQL table) tracks which version of the scoring rubric was used.

### Rules

1. **Version is a monotonically increasing integer** starting at 1. Each change to the rubric increments the version.

2. **Old and new versions coexist.** The `skill_reviews` table uses `UNIQUE (neo4j_skill_id, review_version)`, so a skill can have reviews from multiple rubric versions. The Neo4j Skill node always reflects the latest version.

3. **`--rescore` re-scores using the current rubric version.** Old version rows are preserved for audit.

4. **Rubric changes do not auto-invalidate existing scores.** The CLI warns:
   ```
   Warning: 142 skills scored with rubric v1 (current: v2). Run --rescore to update.
   ```

5. **Version history is documented** in `REVIEW_CHANGELOG.md` or inline in `skill_reviewer.py`.
