# SuperrSkillGraph — How It All Works

> A beginner-friendly guide to understanding the complete system, from a raw CSV file to a live Knowledge Graph.

---

## 1. What Is This Project?

SuperrSkillGraph is a system that takes **curriculum data** (spreadsheets listing topics and skills from educational sources like IXL, Embibe, NCERT, etc.) and converts them into an **intelligent Knowledge Graph** stored in Neo4j.

The Knowledge Graph connects every skill with:
- The broader **Topic** it belongs to
- The **Subject** domain (Mathematics, Science, etc.)
- Which skills a student should **learn first** (prerequisites)
- What **age range** each skill is appropriate for

This graph powers the Superr frontend UI where students explore learning paths visually.

---

## 2. The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER / CLI                               │
│  pipenv run python main.py run --subject math --file data.csv   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 1: CSV INGESTION                        │
│                                                                 │
│  • Reads the CSV/Excel file                                     │
│  • Maps columns to internal fields:                             │
│      subject, unit, topic, concept, grade_hint                  │
│  • Generates embedding text per row                             │
│  • Computes vector embeddings (BAAI/bge-large-en-v1.5)          │
│  • Stores rows in PostgreSQL → pipeline.curriculum_items        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 2: LLM EXTRACTION                       │
│                                                                 │
│  • Reads curriculum_items from PostgreSQL                       │
│  • Sends batches to LLM (GPT-4 / Claude / Gemini)              │
│  • LLM decides:                                                 │
│      - Is this a valid standalone skill?                        │
│      - What age_range is it suitable for?                       │
│      - What are its prerequisites?                              │
│  • Stores LLM decisions in pipeline.llm_decisions               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 3: GRAPH CONSTRUCTION                   │
│                                                                 │
│  • Reads validated LLM decisions                                │
│  • Creates nodes in Neo4j:                                      │
│      Domain → Strand → Concept → Skill                         │
│  • Draws BELONGS_TO arrows (hierarchy)                          │
│  • Draws PREREQUISITE arrows (learning order)                   │
│  • Derives Concept-level prerequisites from Skill-level ones    │
│  • Runs transitive reduction (removes redundant arrows)         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   NEO4J KNOWLEDGE GRAPH                         │
│                                                                 │
│   (Mathematics)                                                 │
│       ├── (Algebra)           ← Strand                         │
│       │    ├── (Linear Equations)  ← Concept                   │
│       │    │    ├── Solve for X     ← Skill                    │
│       │    │    └── Plot on graph   ← Skill                    │
│       │    └── (Quadratics)                                     │
│       └── (Geometry)                                            │
│            ├── (Shapes)                                         │
│            │    ├── Identify squares                            │
│            │    └── Name the shape                              │
│            └── (Angles)                                         │
│                 └── Measure angles                              │
│                                                                 │
│   Arrows: Skill → Concept → Strand → Domain  (BELONGS_TO)     │
│   Arrows: Skill → Skill                      (PREREQUISITE)   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Project Structure

```
SuperrSkillGraph/
├── main.py                      # Entry point — just calls src/cli/main.py
├── docker/
│   ├── docker-compose.yml       # Starts PostgreSQL + Neo4j + SearXNG containers
│   └── init_db.sql              # Creates the database schema on first boot
├── src/
│   ├── cli/
│   │   └── main.py              # CLI commands (run, status, rebuild-graph)
│   ├── config/
│   │   ├── settings.py          # Loads .env and builds config object
│   │   ├── sources.yaml         # Registered CSV data sources (IXL, Embibe)
│   │   └── subjects.yaml        # Subject definitions (mathematics, science)
│   ├── data_sources/
│   │   └── excel_parser.py      # Reads CSV/Excel files into DataFrames
│   ├── pipeline/
│   │   ├── orchestrator.py      # The brain — runs the 5-stage pipeline
│   │   ├── source_manager.py    # Resolves YAML sources into file targets
│   │   └── state_manager.py     # Tracks which stage each document is at
│   ├── processors/
│   │   ├── llm_client.py        # Talks to OpenAI / Anthropic / Google APIs
│   │   ├── content_extractor.py # Builds LLM prompts for skill extraction
│   │   └── embedding_service.py # Generates vector embeddings (HuggingFace)
│   └── storage/
│       ├── raw_data_store.py    # PostgreSQL CRUD operations
│       ├── graph_store.py       # Neo4j CRUD + vector search + prereqs
│       └── embedding_store.py   # Stores/retrieves embedding vectors
├── data/
│   └── curriculum_excel/        # CSV files (IXL, Embibe, etc.)
├── docs/                        # Documentation (you are here!)
└── logs/                        # Pipeline execution logs
```

---

## 4. The Two Databases

### 4.1 PostgreSQL (The Raw Warehouse)

PostgreSQL stores the raw, intermediate, and processed data in tables under the `pipeline` schema:

| Table | Purpose |
|-------|---------|
| `source_documents` | Tracks every CSV file ingested (path, hash, processing state) |
| `curriculum_items` | One row per skill extracted from CSV (subject, topic, concept, embedding) |
| `llm_decisions` | What the LLM decided about each skill (age_range, prerequisites, reasoning) |
| `skill_embeddings` | Dense vector arrays for similarity search |

### 4.2 Neo4j (The Knowledge Graph)

Neo4j stores the final, clean, interconnected graph that the frontend queries.

**Node Types:**

| Node | What It Represents | Example |
|------|--------------------|---------|
| `Domain` | The subject | Mathematics |
| `Strand` | A broad area within the subject | Algebra, Geometry |
| `Concept` | A specific topic | Linear Equations, Shapes |
| `Skill` | A single testable learning objective | "Solve for X", "Identify squares" |

**Relationship Types:**

| Relationship | Meaning | Example |
|-------------|---------|---------|
| `BELONGS_TO` | Hierarchy (child → parent) | (Skill) → (Concept) → (Strand) → (Domain) |
| `PREREQUISITE` | Learning dependency | (Multiplication) → (Addition) |

---

## 5. How Columns Map to Nodes

When you give the pipeline a CSV, each column maps directly to a node type in Neo4j:

| CSV Column | Internal Field | Neo4j Node | Required? |
|------------|---------------|------------|-----------|
| subject / domain | `subject` | Domain | Yes (or use `--subject` flag) |
| unit / strand / area | `unit` | Strand | No (skipped if absent) |
| topic / chapter / module | `topic` | Concept | **Yes** |
| concept / skill_name / skill | `concept` | Skill | **Yes** |
| grade / grade_label / level | `grade_hint` | *(stored as metadata for LLM)* | No |

### Auto-Detection

If your CSV uses standard column names (`topic`, `concept`, `unit`, etc.), the pipeline detects them automatically.

### Explicit Mapping

If your CSV uses non-standard column names, use `--column-map`:
```bash
--column-map "topic=ChapterTitle,concept=LearningGoal,grade_hint=StudentAge"
```

---

## 6. CLI Commands Reference

### Run the pipeline
```bash
# Generic CSV intake — auto-detect columns
pipenv run python main.py run --subject mathematics --file ./my_data.csv

# Generic CSV intake — explicit column mapping
pipenv run python main.py run --subject mathematics --file ./my_data.csv \
  --column-map "topic=Chapter,concept=Objective"

# YAML-configured source (existing workflow)
pipenv run python main.py run --subject mathematics --source embibe

# Limit to first N rows (for testing)
pipenv run python main.py run --subject mathematics --file ./my_data.csv --rows 5

# Dry run (show what would be processed, don't execute)
pipenv run python main.py run --subject mathematics --source embibe --dry-run
```

### Check status
```bash
pipenv run python main.py status
pipenv run python main.py status --subject mathematics --format json
```

### Rebuild graph from existing LLM decisions
```bash
pipenv run python main.py rebuild-graph --subject mathematics
```

---

## 7. Embedding: How AI Understands Skills

Each curriculum item gets converted into a **dense vector embedding** — a list of 1024 floating-point numbers that mathematically represents the meaning of the skill.

**How it works:**
1. The pipeline builds an embedding text string:
   - **With unit (4-level):** `"Mathematics > Algebra > Linear Equations > Solve for X"`
   - **Without unit (3-level):** `"Mathematics > Shapes > Identify squares"`
2. The HuggingFace model `BAAI/bge-large-en-v1.5` converts this string into a 1024-dimensional vector.
3. This vector is stored in both PostgreSQL (`curriculum_items.embedding`) and Neo4j (on the Skill node).

**Why it matters:**
When the LLM needs to find prerequisites for "Solve for X", it doesn't search by text. It uses **cosine similarity** to find the skills whose vectors are mathematically closest in 1024-dimensional space — which often reveals connections that text matching would miss.

---

## 8. Prerequisites: How Learning Order Is Built

The system automatically determines which skills should be learned before others:

1. **Vector Search:** For each new skill, find the most similar existing skills using embedding similarity.
2. **LLM Decision:** Send the skill + its similar matches to GPT-4/Claude and ask: *"Which of these existing skills should a student learn BEFORE this new skill?"*
3. **Graph Construction:** Draw `PREREQUISITE` arrows between skills based on the LLM's answer.
4. **Concept Rollup:** If most skills in Concept A are prerequisites of skills in Concept B, automatically draw a `PREREQUISITE` arrow between the Concepts too.
5. **Transitive Reduction:** Clean up redundant arrows. If A→B→C exists, remove the direct A→C arrow since it's already implied.

---

## 9. What the Frontend Sees

The React frontend at `superr-skill-graph.vercel.app` simply queries the Neo4j graph:

```
1,720 SKILLS  |  319 CONCEPTS  |  40 STRANDS  |  5.4 AVG SKILLS/CONCEPT
```

Each strand expands to show its concepts, and each concept expands to show individual skills. The graph view draws the prerequisite arrows visually, creating an interactive learning map.

---

## 10. Key Files to Understand

| File | What It Does |
|------|-------------|
| `src/cli/main.py` | Parses CLI arguments, validates input, calls orchestrator |
| `src/pipeline/orchestrator.py` | The brain — runs 5 pipeline stages in sequence |
| `src/storage/graph_store.py` | All Neo4j operations (create nodes, relationships, vector search) |
| `src/storage/raw_data_store.py` | All PostgreSQL operations (insert, query, state tracking) |
| `src/processors/llm_client.py` | Talks to OpenAI/Claude/Gemini APIs for skill extraction |
| `src/processors/embedding_service.py` | Generates vector embeddings using HuggingFace |
| `src/data_sources/excel_parser.py` | Reads CSV/Excel files into pandas DataFrames |
| `docker/docker-compose.yml` | Starts PostgreSQL, Neo4j, and SearXNG containers |

---

## 11. Quick Start

```bash
# 1. Start the databases
cd docker && docker-compose up -d

# 2. Ingest a CSV
pipenv run python main.py run --subject mathematics --file ./data/curriculum_excel/ixl_maths_skills.csv --rows 10

# 3. Check the data in PostgreSQL
docker exec docker-postgres-1 psql -U postgres -d curriculum_kg \
  -c "SELECT subject, topic, concept FROM pipeline.curriculum_items LIMIT 5;"

# 4. Check the graph in Neo4j (browser: http://localhost:7474)
# Username: neo4j, Password: password
```
