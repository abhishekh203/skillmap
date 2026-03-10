# Post-Extraction Stages — Prerequisite Clean-Up & Validation

> This document covers what happens after the extraction stage writes skills to Neo4j. Three remaining stages clean up, derive relationships, and validate the final graph.

---

## Pipeline Stages Recap

| Stage | Name | Where Work Happens |
|-------|------|--------------------|
| 1 | Ingestion | CSV → PostgreSQL + embeddings |
| 2 | Extraction | PostgreSQL → LLM → Neo4j (nodes + edges) |
| **3** | **Graph Construction** | **No-op (merged into extraction)** |
| **4** | **Prerequisite Inference** | **Transitive reduction + concept prereq derivation** |
| **5** | **Validation** | **Graph integrity checks** |

---

## Stage 3: Graph Construction (No-Op)

This stage does **nothing** — it just advances the document state in PostgreSQL.

**Why it exists:** Originally, the pipeline was designed with extraction (LLM calls) and graph construction (Neo4j writes) as separate stages. But the current design writes to Neo4j **immediately** after each LLM call in Stage 2 (so that the next item can see what the previous one created). This made Stage 3 redundant.

Rather than remove it (which would break the stage numbering and state machine), the code simply passes through:

```python
def _run_graph_construction_stage(self):
    # Just advance document state — actual work was done in Stage 2
    for doc in documents:
        self.state_manager.update_document_stage(
            doc['id'], PipelineStage.GRAPH_CONSTRUCTION, True
        )
    logger.info("Graph construction stage: already completed inline during extraction")
```

**Time taken:** < 0.1 seconds (it's just a database state update)

---

## Stage 4: Prerequisite Inference

This is the graph clean-up stage. It does two important things:

### Step 1: Transitive Reduction

**Problem:** During extraction, the LLM sometimes creates redundant edges. For example:

```
Before:
  Addition → Subtraction → Multiplication → Division
  Addition → Division    ← REDUNDANT! (already implied through Subtraction → Multiplication)
```

The LLM might say *"Division depends on Addition"* — which is technically true, but the dependency is already captured through the chain. Having both the direct edge AND the transitive path creates visual clutter in the frontend.

**Solution:** The Cypher query finds and removes these redundant shortcuts:

```cypher
MATCH (a:Skill)-[direct:PREREQUISITE]->(c:Skill)
WHERE EXISTS {
    MATCH (a)-[:PREREQUISITE]->(:Skill)-[:PREREQUISITE*1..10]->(c)
}
DELETE direct
```

**In plain English:** *"If there's already a path from A to C through other skills, delete the direct A→C edge."*

```
After:
  Addition → Subtraction → Multiplication → Division
  (direct Addition → Division edge removed ✅)
```

### Step 2: Concept Prerequisite Derivation

**Problem:** The LLM creates prerequisites at the **Skill** level, but the frontend also needs prerequisites at the **Concept** level for its overview view.

**Solution:** Automatically derive Concept→Concept prerequisites from Skill→Skill prerequisites:

```cypher
MATCH (s1:Skill)-[:PREREQUISITE]->(s2:Skill)        -- Skill A depends on Skill B
MATCH (s1)-[:BELONGS_TO]->(c1:Concept)               -- Skill A belongs to Concept X
MATCH (s2)-[:BELONGS_TO]->(c2:Concept)               -- Skill B belongs to Concept Y
WHERE c1.id <> c2.id                                  -- Different concepts
MERGE (c1)-[r:PREREQUISITE]->(c2)                     -- Create Concept X → Concept Y
```

**In plain English:** *"If any skill in 'Fractions' depends on any skill in 'Number System', then 'Fractions' depends on 'Number System' at the concept level."*

**Example:**
```
Skill level:
  "Add fractions" (in Fractions) → "Multiply integers" (in Multiplication)
  "Compare fractions" (in Fractions) → "Compare integers" (in Integers)

Derived concept level:
  Fractions → Multiplication
  Fractions → Integers
```

### Stage 4 Output

```python
{
    'skill_prerequisites': 245,        # Total skill-level prerequisite edges
    'concept_prerequisites': 38,       # Derived concept-level prerequisite edges
    'transitive_removed': 12,          # Redundant edges cleaned up
    'total_relationships': 283,        # Total across both levels
}
```

---

## Stage 5: Validation

The final stage runs **5 integrity checks** on the Neo4j graph to catch any data quality issues:

### Check 1: Orphaned Concepts
```cypher
MATCH (c:Concept)
WHERE NOT (c)-[:BELONGS_TO]->(:Domain)
  AND NOT (c)-[:BELONGS_TO]->(:Strand)
RETURN count(c)
```
*"Are there any concepts not connected to a domain or strand?"*
This would mean a concept is floating in the graph with no parent — likely a bug.

### Check 2: Orphaned Strands
```cypher
MATCH (st:Strand)
WHERE NOT (st)-[:BELONGS_TO]->(:Domain)
RETURN count(st)
```
*"Are there any strands not connected to a domain?"*

### Check 3: Orphaned Skills
```cypher
MATCH (s:Skill)
WHERE NOT (s)-[:BELONGS_TO]->(:Concept)
RETURN count(s)
```
*"Are there any skills not connected to a concept?"*

### Check 4: Missing Age Ranges
```cypher
MATCH (s:Skill)
WHERE s.age_range IS NULL OR size(s.age_range) = 0
RETURN count(s)
```
*"Are there any skills where the LLM didn't assign an age range?"*

### Check 5: Circular Prerequisites
```cypher
MATCH path = (s:Skill)-[:PREREQUISITE*]->(s)
RETURN count(path)
```
*"Are there any circular dependencies?"* (e.g., A depends on B, B depends on C, C depends on A). This would create an impossible learning path and is always a bug.

### After All Checks

The validation stage marks all source documents as **COMPLETED**:
```python
self.state_manager.update_document_stage(doc['id'], PipelineStage.COMPLETED, True)
```

This means the document has successfully gone through all 5 stages and the processing pipeline is done.

### Validation Output

```python
{
    'validated_count': 2,
    'total_documents': 2,
    'graph_validation': {
        'issues': [],              # Empty = no problems found ✅
        'node_counts': {
            'Domain': 1,
            'Strand': 8,
            'Concept': 45,
            'Skill': 312,
        },
        'is_valid': True,          # All checks passed
    },
}
```

---

## The Complete Pipeline Log

When you run the full pipeline, the logs show all 5 stages in sequence:

```
Stage 1: Ingestion
  Stored 5 curriculum items with embeddings for doc 3

Stage 2: Extraction (LLM decision + immediate Neo4j write per item)
  [1/5] Addition of Like Terms → skill_a1b2c3 (new)
  [2/5] Subtraction of Like Terms → skill_d4e5f6 (new)
  [3/5] Additive Inverse → skill_g7h8i9 (new)
  [4/5] Solving Word Problems → skill_j1k2l3 (new)
  [5/5] Adjoint of a Matrix → skill_m4n5o6 (new)
  Extraction complete: 5 new, 0 updated, 0 skipped, 0 failed

Stage 3: Graph construction: already completed inline during extraction

Stage 4: Prerequisite inference
  Step 1/2: Transitive reduction removed 2 redundant edges
  Step 2/2: Derived 8 concept prerequisite edges

Stage 5: Validation
  Graph validation: is_valid=True

Pipeline completed in 45.3s
  Documents: 2
  Errors:    0
```

---

## Summary

| Stage | What It Does | Time | Key Files |
|-------|-------------|------|-----------|
| 3 | Nothing (pass-through) | <0.1s | `orchestrator.py` |
| 4 | Remove redundant edges + derive concept prereqs | ~1-5s | `graph_store.py` (transitive_reduction, derive_concept_prerequisites) |
| 5 | Check for orphans, missing data, circular deps | ~1-2s | `graph_store.py` (validate_graph_integrity) |

After Stage 5, the processing state of every source document is set to `COMPLETED`, and the Knowledge Graph is ready for the frontend to query.
