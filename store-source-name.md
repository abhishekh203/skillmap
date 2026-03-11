# Proposal: Store Source Names in Neo4j (instead of IDs)

## Problem

Right now, every node in Neo4j (Domain, Concept, Skill) stores a `source_ids` list like:

```
source_ids: ["1", "2", "3"]
```

These are just PostgreSQL integer IDs — **meaningless on their own**. To find out which curriculum source a skill came from, you'd have to go back to PostgreSQL and look up the `source_documents` table.

## Solution

Replace those numeric IDs with **actual source names**:

```
// Before
source_ids: ["1", "2"]

// After
source_names: ["IXL Mathematics Curriculum", "Embibe Math Skills"]
```

Now anyone querying Neo4j can immediately see **where** a skill came from without needing PostgreSQL.

## What Changes

| File | What happens |
|------|-------------|
| `src/storage/graph_store.py` | Rename `source_ids` → `source_names` in all Cypher queries |
| `src/pipeline/orchestrator.py` | Look up source name from PostgreSQL before writing to Neo4j |
| `src/storage/raw_data_store.py` | Add helper to fetch source name by document ID |
| `data/neo4j_export/*.csv` | Update seed data column if needed |

## How It Works (Before vs After)

**Before:**
```
CSV row ingested → stored with source_document_id = 5
→ Neo4j skill gets source_ids: ["5"]
→ "5" means nothing without PostgreSQL lookup
```

**After:**
```
CSV row ingested → stored with source_document_id = 5
→ Look up source_documents table → metadata.source_name = "IXL Mathematics Curriculum"
→ Neo4j skill gets source_names: ["IXL Mathematics Curriculum"]
→ Self-descriptive, no extra lookup needed
```

## Migration

Existing Neo4j data needs a fresh run (`--clean`) or a one-time Cypher rename query.
