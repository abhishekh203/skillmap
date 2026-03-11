# Proposal: Store Source Names in Neo4j (instead of IDs)

## Problem

Right now, every node in Neo4j (Domain, Concept, Skill) stores a `source_ids` list like:

```
source_ids: ["1", "2", "3"]
```

These are just PostgreSQL integer IDs — **meaningless on their own**. To find out which curriculum source a skill came from, you'd have to go back to PostgreSQL and look up the `source_documents` table.

## Solution

Replace numeric IDs with **domain-style source names**:

```
// Before
source_ids: ["1", "2"]

// After
source_names: ["in.ixl.com/maths", "embibe.com/maths"]
```

### Naming Convention

Use `domain/subject` format — short, recognizable, consistent:

| Source | Name |
|--------|------|
| IXL India Maths | `in.ixl.com/maths` |
| Embibe Maths | `embibe.com/maths` |
| Khan Academy (future) | `khanacademy.org/maths` |
| NCERT (future) | `ncert.nic.in/maths` |

When a skill is contributed by multiple sources, Neo4j shows all of them:
```
source_names: ["in.ixl.com/maths", "khanacademy.org/maths"]
```

## Two Ways to Ingest (both work)

### 1. Ad-hoc CSV via `--file` (no YAML config needed)

```bash
# Source name defaults to filename
python main.py run --subject mathematics --file ./khan_maths.csv
# → source_name = "khan_maths.csv"

# With explicit source name (NEW --source-name flag)
python main.py run --subject mathematics --file ./khan_maths.csv --source-name "khanacademy.org/maths"
# → source_name = "khanacademy.org/maths"
```

### 2. YAML-configured source via `--source`

```yaml
# sources.yaml
ixl_maths_curriculum:
  name: "in.ixl.com/maths"          # ← domain-style name
  type: "excel"
  file_path: "./data/curriculum_excel/ixl_maths_skills.csv"
  subjects: ["mathematics"]
```

```bash
python main.py run --subject mathematics --source ixl
# → source_name read from sources.yaml = "in.ixl.com/maths"
```

## What Changes

| File | What happens |
|------|-------------|
| `src/config/sources.yaml` | Update `name` fields to domain-style |
| `src/cli/main.py` | Add `--source-name` option for `--file` mode |
| `src/storage/graph_store.py` | Rename `source_ids` → `source_names` in Cypher |
| `src/pipeline/orchestrator.py` | Look up source name from PG, pass to graph_store |
| `src/storage/raw_data_store.py` | Add `get_source_name(doc_id)` helper |
| `data/neo4j_export/*.csv` | Update seed data column if needed |

## Data Flow

```
CLI: --source-name "khanacademy.org/maths" (or from sources.yaml name field)
  ↓
Stored in source_documents.metadata['source_name']
  ↓
Graph construction reads it back
  ↓
Neo4j node: source_names: ["khanacademy.org/maths"]
```

## Migration

Existing Neo4j data needs a fresh run (`--clean`) or a one-time Cypher rename query.
