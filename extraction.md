# Extraction Deep Dive — What Happens Step by Step

> This document traces exactly what happens during Stage 2 (Extraction). After ingestion fills PostgreSQL with raw curriculum rows + embeddings, extraction sends each row to an LLM and writes the result directly into the Neo4j Knowledge Graph.

---

## Overview

Extraction processes every "pending" curriculum item **one at a time** in a loop:

```
For each pending curriculum_item:
  1. Vector search Neo4j → find similar existing skills
  2. DFS from matches → get full prerequisite trees
  3. Build a prompt → send to LLM (GPT-4 / Claude / Gemini)
  4. LLM returns a structured JSON decision
  5. Immediately write nodes + edges to Neo4j
  6. Mark the item as "completed" in PostgreSQL
```

**Why one at a time?** Because each item is written to Neo4j *immediately* after the LLM call. This means item #2 can "see" what item #1 created via vector search — enabling accurate deduplication and prerequisite discovery. Batching would lose this sequential context.

---

## Step 1: Get Pending Items

The orchestrator fetches all curriculum items with `status = 'pending'` from PostgreSQL:

```python
pending_items = self.embedding_store.get_curriculum_items_by_status('pending')
# Returns list of dicts, each containing:
# {id, source_document_id, subject, unit, topic, concept,
#  grade_hint, item_type, embedding_text, embedding}
```

These are the rows that ingestion stored in `pipeline.curriculum_items` — each has a pre-computed 1024-dimensional embedding vector ready to use.

---

## Step 2: Vector Search — Find Similar Skills (`graph_store.py`)

For each item, the system uses its **embedding vector** to search Neo4j for the most similar skills that already exist in the graph:

```python
similar_skills = self.graph_store.find_similar_skills(
    query_embedding=embedding,     # The 1024-float vector from ingestion
    threshold=0.85,                # Only return skills with ≥85% similarity
    limit=200,                     # Return at most 200 matches
)
```

**What this does internally:**
- Computes **cosine similarity** between the new item's embedding and every existing Skill node's embedding in Neo4j
- Returns a list of matches, each with: `{neo4j_skill_id, title, similarity_score}`

**Example result:**
```python
similar_skills = [
    {'neo4j_skill_id': 'skill_a1b2c3', 'title': 'Add fractions with like denominators', 'similarity': 0.93},
    {'neo4j_skill_id': 'skill_d4e5f6', 'title': 'Subtract fractions with like denominators', 'similarity': 0.88},
]
```

**Why this matters:** These matches become the "context" for the LLM. Without them, the LLM wouldn't know what's already in the graph and would create duplicates or miss prerequisites.

---

## Step 3: DFS — Get Full Prerequisite Trees (`graph_store.py`)

The vector search gives us "seed" skills. But the LLM needs to see the **complete learning progression** — not just the matches, but all their prerequisites and dependents too.

```python
seed_ids = [s['neo4j_skill_id'] for s in similar_skills]
# ['skill_a1b2c3', 'skill_d4e5f6']

tree_data = self.graph_store.get_full_prerequisite_trees(seed_ids)
```

This performs a **bidirectional DFS (Depth-First Search)** from each seed skill, following PREREQUISITE edges in both directions:
- **Upstream:** What skills does this skill depend on? (prerequisites)
- **Downstream:** What skills depend on this skill? (dependents)

**Example tree_data result:**
```python
tree_data = {
    'trees': [
        {
            'seed_id': 'skill_a1b2c3',
            'skill_ids': ['skill_x1', 'skill_x2', 'skill_a1b2c3', 'skill_x3']
        }
    ],
    'nodes': {
        'skill_x1': {'title': 'Identify fractions on a number line', 'concept': 'Fractions', ...},
        'skill_x2': {'title': 'Compare fractions', 'concept': 'Fractions', ...},
        'skill_a1b2c3': {'title': 'Add fractions with like denominators', ...},
        'skill_x3': {'title': 'Add fractions with unlike denominators', ...},
    },
    'edges': [
        {'from_id': 'skill_x2', 'to_id': 'skill_x1', 'order': 1},   # Compare requires Identify
        {'from_id': 'skill_a1b2c3', 'to_id': 'skill_x2', 'order': 1}, # Add-like requires Compare
        {'from_id': 'skill_x3', 'to_id': 'skill_a1b2c3', 'order': 1}, # Add-unlike requires Add-like
    ]
}
```

**Visualized as a tree:**
```
Level 0 (Foundation):  Identify fractions on a number line
       ↓ PREREQUISITE
Level 1:               Compare fractions
       ↓ PREREQUISITE
Level 2:               Add fractions with like denominators  ← [seed match, 93% similar]
       ↓ PREREQUISITE
Level 3:               Add fractions with unlike denominators
```

The LLM sees this **entire tree** and must decide: where does the new skill fit?

---

## Step 4: Build the LLM Prompt (`graph_decision_maker.py`)

The `GraphDecisionMaker` constructs a structured prompt with two parts:

### 4a. System Prompt (same for every call)

The system prompt explains the graph hierarchy rules to the LLM:

```
You are building a hierarchical knowledge graph with four levels:
  Domain  >  Strand  >  Concept  >  Skill

DOMAIN — Broad area of knowledge. (Mathematics, Science)
         Almost always use_existing.

STRAND — A narrower sub-area. (Algebra, Geometry)
         Group related concepts under the same strand.

CONCEPT — An abstract learning area. Titles MUST be noun-based.
          Good: "Fractions"  Bad: "Adding Fractions"

SKILL — A specific, assessable action. Titles MUST start with a verb.
        Good: "Add fractions with unlike denominators"

PREREQUISITE TREE POSITIONING — decide where the new skill fits:
  1. AT THE END:    NEW_SKILL depends on existing skills
  2. IN THE MIDDLE: insert between two existing skills
  3. AS FOUNDATION: other skills should depend on it

AGE RANGE — Infer age range using Indian education system:
  Class 1 ≈ age 6, Class 2 ≈ age 7, ..., Class 12 ≈ age 17
```

### 4b. User Prompt (built per item)

```
=== INFORMATION TO PROCESS ===
Mathematics > Algebra > Fractions > Add fractions with unlike denominators
Grade level: Class 5

Note: the last segment is a concept label — rephrase it as a
specific, action-oriented skill sentence (verb phrase).

=== RELATED PREREQUISITE TREES (4 skills across 1 tree(s)) ===

=== PREREQUISITE TREE 1 (from matched skill: "Add fractions with like denominators") ===

--- Level 0 (Foundation Skills) ---
  [skill_x1] "Identify fractions on a number line"
    Domain: Mathematics | Strand: Arithmetic | Concept: Fractions
    age_range: [7, 8]

--- Level 1 ---
  [skill_x2] "Compare fractions"    ★ 88% similar
    Prerequisites: skill_x1 (order 1)

--- Level 2 ---
  [skill_a1b2c3] "Add fractions with like denominators"    ★ 93% similar
    Prerequisites: skill_x2 (order 1)

Decide how this information integrates into the graph. Return JSON only.
```

---

## Step 5: LLM Returns a Structured JSON Decision

The LLM responds with a JSON object that has decisions at **every level** of the graph:

```json
{
  "decision_summary": "This is a new skill for adding fractions with unlike denominators, building on existing fraction addition skills.",

  "domain": {
    "action": "use_existing",
    "existing_id": "domain_abc123",
    "title": "Mathematics",
    "description": "The study of numbers, quantities, and shapes."
  },

  "strand": {
    "action": "use_existing",
    "existing_id": "strand_def456",
    "title": "Arithmetic",
    "description": "Basic operations with numbers and fractions."
  },

  "concept": {
    "action": "use_existing",
    "existing_id": "concept_ghi789",
    "title": "Fractions",
    "description": "Understanding and operating with parts of a whole."
  },

  "skill": {
    "action": "new",
    "existing_id": null,
    "title": "Add fractions with unlike denominators",
    "description": "Calculate the sum of two fractions that have different denominators by finding a common denominator.",
    "age_range": [10, 11]
  },

  "edge_operations": [
    {
      "action": "add",
      "from_skill_id": "NEW_SKILL",
      "to_skill_id": "skill_a1b2c3",
      "order": 1,
      "reasoning": "Adding fractions with unlike denominators requires mastery of adding fractions with like denominators."
    }
  ]
}
```

### Possible actions at each level:

| Level | Possible Actions | Meaning |
|-------|-----------------|---------|
| Domain | `use_existing`, `create_new` | Almost always `use_existing` |
| Strand | `use_existing`, `create_new`, `skip` | `skip` = no strand needed |
| Concept | `use_existing`, `create_new`, `update` | Group skills under concepts |
| Skill | `new`, `update_existing`, `skip` | `skip` = duplicate/noise |

---

## Step 6: Validation — Guard Rails (`graph_decision_maker.py`)

Before writing to Neo4j, the code validates the LLM response:

1. **Skill action check:** If the action is not one of `new`, `update_existing`, `skip` → default to `new`
2. **update_existing check:** If the LLM references a skill ID that doesn't exist → convert to `new`
3. **use_existing check:** If domain/strand/concept references are empty → convert to `create_new`
4. **Edge operations filter:** Remove any edges that reference skill IDs not in the tree context (hallucinated IDs)

```python
# Filter edge_operations to only valid skill IDs + NEW_SKILL
valid_ids = all_skill_ids | {"NEW_SKILL"}
response["edge_operations"] = [
    op for op in response.get("edge_operations", [])
    if op.get("from_skill_id") in valid_ids
    and op.get("to_skill_id") in valid_ids
]
```

This prevents the LLM from inventing non-existent nodes or creating edges to skills it doesn't know about.

---

## Step 7: Write to Neo4j — Build the Graph (`orchestrator.py`)

This is where the LLM decision becomes real nodes and edges:

### 7a. Create/Reuse Domain
```python
domain_id = "domain_abc123"  # From LLM: use_existing
self.graph_store.create_domain(domain_id, title="Mathematics", ...)
# → MERGE (d:Domain {id: "domain_abc123"}) SET d.title = "Mathematics"
```

### 7b. Create/Reuse Strand
```python
strand_id = "strand_def456"
self.graph_store.create_strand(strand_id, title="Arithmetic", domain_id=domain_id, ...)
self.graph_store.create_belongs_to_relationship(strand_id, domain_id, 'Strand', 'Domain')
# → (Arithmetic) -[:BELONGS_TO]→ (Mathematics)
```

### 7c. Create/Reuse Concept
```python
concept_id = "concept_ghi789"
self.graph_store.create_concept(concept_id, title="Fractions", ...)
self.graph_store.create_belongs_to_relationship(concept_id, strand_id, 'Concept', 'Strand')
# → (Fractions) -[:BELONGS_TO]→ (Arithmetic)
```

### 7d. Create New Skill
```python
# Generate a deterministic ID from the hierarchy
hash_base = "Mathematics|Arithmetic|Fractions|Add fractions with unlike denominators"
skill_id = "skill_" + sha1(hash_base)[:12]   # e.g., "skill_7f3a2b9c1d4e"

# Generate a FRESH embedding for the skill (using its final title, not the CSV text)
emb_text = "Mathematics > Fractions > Add fractions with unlike denominators"
emb_vec = self.embedding_service.embed_text(emb_text)   # New 1024-float vector

self.graph_store.create_skill(skill_id, title="Add fractions with unlike denominators",
                               age_range=[10, 11], embedding=emb_vec, ...)
self.graph_store.create_belongs_to_relationship(skill_id, concept_id, 'Skill', 'Concept')
# → (Add fractions...) -[:BELONGS_TO]→ (Fractions)
```

### 7e. Execute Edge Operations (Prerequisites)
```python
# LLM said: NEW_SKILL depends on skill_a1b2c3
# Replace placeholder: from_id = "skill_7f3a2b9c1d4e", to_id = "skill_a1b2c3"

self.graph_store.create_prerequisite_relationship(
    dependent_id="skill_7f3a2b9c1d4e",    # This new skill
    prerequisite_id="skill_a1b2c3",        # Add fractions with like denominators
    node_type='Skill',
    order=1,
    reasoning="Adding fractions with unlike denominators requires..."
)
# → (Add unlike) -[:PREREQUISITE {order: 1}]→ (Add like)
```

### 7f. Derive Concept Prerequisites
```python
self.graph_store.derive_concept_prerequisites()
```
After writing skill-level prerequisites, this function automatically checks: *"Do most skills in Concept A depend on skills in Concept B?"* If so, it creates a `PREREQUISITE` edge between the Concepts too.

### 7g. Mark Item Completed
```python
self.embedding_store.update_curriculum_item_status(
    item_id=42,
    status='completed',
    neo4j_skill_id='skill_7f3a2b9c1d4e',
    llm_decision=decision,  # Store the full JSON for debugging
)
```

The curriculum item is now marked as `completed` in PostgreSQL, and the `llm_decision` JSON is saved so you can debug or audit why the LLM made that choice.

---

## What the Graph Looks Like After

After processing one item, this was added to Neo4j:

```
(Mathematics)                             ← Domain (reused)
    └── (Arithmetic)                      ← Strand (reused)
        └── (Fractions)                   ← Concept (reused)
            ├── Identify fractions        ← Skill (existed)
            ├── Compare fractions         ← Skill (existed)
            ├── Add like denominators     ← Skill (existed)
            └── Add unlike denominators   ← Skill (NEW!)
                    │
                    └──[PREREQUISITE]──→ Add like denominators
```

---

## Complete Call Stack

```
For each pending curriculum_item:

1. embedding_store.get_curriculum_items_by_status('pending')
   │
2. graph_store.find_similar_skills(embedding, threshold=0.85)
   │   → cosine similarity search in Neo4j
   │
3. graph_store.get_full_prerequisite_trees(seed_ids)
   │   → bidirectional DFS from matched skills
   │
4. graph_decision_maker.decide(raw_info, tree_data, ...)
   │   ├── _build_user_prompt()          # Format trees + info for LLM
   │   ├── llm_client.generate_structured_json_response()  # API call
   │   └── Validation                     # Filter bad IDs, fix actions
   │
5. orchestrator._build_graph_from_decision(item, decision, embedding)
   │   ├── create_domain()               # MERGE Domain node
   │   ├── create_strand()               # MERGE Strand node
   │   ├── create_concept()              # MERGE Concept node
   │   ├── create_skill()                # CREATE Skill node + embedding
   │   ├── create_belongs_to_relationship()  # Hierarchy edges
   │   ├── _execute_edge_operations()    # PREREQUISITE edges
   │   ├── derive_concept_prerequisites() # Concept-level prereqs
   │   └── update_curriculum_item_status('completed')
   │
   └── Next item... (can now see this skill via vector search)
```

---

## Key Insight: Why "Immediate Write" Matters

Consider 5 items being processed in order:
1. "Add integers" → No matches in graph. Creates new.
2. "Subtract integers" → Now sees "Add integers" via vector search. Creates prerequisite edge.
3. "Multiply integers" → Sees both "Add" and "Subtract". Positions itself correctly.
4. "Divide integers" → Sees all three. Creates edge to "Multiply".
5. "Order of operations" → Sees all four. Creates edges to all relevant skills.

If we batched all 5 LLM calls at once, item #2 would NOT see item #1, and the graph would have no prerequisite connections. The one-at-a-time + immediate-write pattern is what makes the learning path discovery work.
