# Ingestion Deep Dive — What Happens Step by Step

> This document traces exactly what happens from the moment you press Enter on the CLI command until the data lands in PostgreSQL with embeddings.

---

## Overview

When you run this command:
```bash
pipenv run python main.py run --subject mathematics --file my_data.csv --rows 5
```

The system goes through **8 sequential steps** before a single row is saved. Here is every step in detail.

---

## Step 1: CLI Parsing (`src/cli/main.py`)

The `click` library reads your terminal arguments and validates them:

```
--subject mathematics    →  subject = "mathematics"
--file my_data.csv       →  file_path = "/full/path/to/my_data.csv"
--rows 5                 →  max_rows = 5
--column-map "..."       →  column_map = {"topic": "chapter", ...}  (if provided)
```

**Validations performed:**
- ❌ If neither `--source` nor `--file` is given → error
- ❌ If both `--source` and `--file` are given → error
- ❌ If `--column-map` is given without `--file` → error
- ✅ If `--file` is provided → mode is set to `generic`

**What gets built:**
The CLI constructs a "target" dictionary — a standardized description of what to process:
```python
target = {
    'file_path': '/Users/you/Desktop/my_data.csv',
    'source_name': 'my_data.csv',
    'type': 'excel',
    'subject': 'mathematics',
    'description': 'Ad-hoc file: my_data.csv',
}
```

This target is then passed to the `orchestrator.run_pipeline()` method.

---

## Step 2: Pipeline Initialization (`orchestrator.run_pipeline`)

Before touching any data, the orchestrator connects to all three backends:

```
1. PostgreSQL  →  raw_store.connect()           # For storing rows
2. Neo4j       →  graph_store.connect()          # For building the graph
3. Neo4j       →  graph_store.initialize_schema() # Create constraints & indexes
4. PostgreSQL  →  embedding_store.connect()       # For storing vectors
5. HuggingFace →  embedding_service.load_model()  # Load the AI model into RAM
```

The HuggingFace model (`BAAI/bge-large-en-v1.5`) is ~1.3 GB. On the first run it downloads from the internet; on subsequent runs it loads from disk cache. This is the heaviest startup cost.

---

## Step 3: CSV File Reading + Header Auto-Detection (`excel_parser.py`)

The `ExcelParser` opens your file and reads it into a pandas DataFrame:

```python
parsed_excel = self.excel_parser.parse_excel_file(file_path)
```

**What happens internally:**
1. Detects the file extension (`.csv`, `.xlsx`, `.xls`)
2. For CSV files → uses `pd.read_csv()` with `header=None` (treats ALL rows as data)
3. Cleans the DataFrame (strips whitespace, removes fully empty rows/columns)
4. **Auto-detects the header row** — scans first 5 rows for known curriculum keywords (`topic`, `concept`, `subject`, `skill_name`, `unit`, `grade_label`, etc.). If a row contains ≥2 keyword matches, it gets promoted to column names and removed from data.
5. Computes a SHA-256 hash of the file contents (for deduplication)
6. Returns a `ParsedExcel` object containing the DataFrame(s) with **proper string column names**

**Header auto-detection example:**
The parser reads the raw CSV with integer columns (0, 1, 2, 3), then scans row 0:
```python
row_vals = ["subject", "unit", "topic", "concept"]  # 4 keyword matches (≥2) → header!
df.columns = ["subject", "unit", "topic", "concept"]  # Promoted to column names
df = df.iloc[1:]  # Row 0 removed from data
```

**Example — what the DataFrame looks like after this step:**

| subject | unit | topic | concept |
|---------|------|-------|---------|
| Mathematics | Algebra | Linear Equations | Solve for X |
| Mathematics | Geometry | Shapes | Identify squares |

*(Column names are already proper strings — the orchestrator doesn't need to fix them!)*

---

## Step 4: Source Document Registration (`raw_data_store.py`)

Before parsing any rows, the pipeline registers the entire file as a "source document" in PostgreSQL:

```python
doc_id = self.raw_store.store_source_document(
    file_path=file_path,
    source_type='excel',
    subject='mathematics',
    raw_content='<full text dump of all sheets>',
    metadata={
        'source_name': 'my_data.csv',
        'title': 'my_data',
        'sheet_count': 1,
    }
)
```

This inserts one row into `pipeline.source_documents`:

| id | file_path | source_type | subject | processing_state |
|----|-----------|------------|---------|-----------------|
| 3 | /path/to/my_data.csv | excel | mathematics | pending |

The returned `doc_id` (e.g., 3) becomes the foreign key that links every curriculum row back to its original file.

---

## Step 5: Column Mapping (`orchestrator.py`)

The orchestrator needs to figure out which DataFrame column holds the "topic", which holds the "concept", etc. Since `excel_parser.py` already auto-detected the header row (Step 3), the DataFrame arrives with proper lowercase string column names.

### For Generic mode (`--file`):

The orchestrator normalises column names for safety, then resolves the mapping:

| subject | unit | topic | concept |
|---------|------|-------|---------|
| Mathematics | Algebra | Linear Equations | Solve for X |
| Mathematics | Geometry | Shapes | Identify squares |

**5b. Resolve column mapping (`_resolve_column_map`):**

The function tries three strategies in order:

```
Priority 1: EXPLICIT MAP (from --column-map)
    User said: --column-map "concept=SpecificTask"
    Result:    resolved["concept"] = "specifictask"

Priority 2: EXACT MATCH
    CSV has a column literally named "topic"?
    Result:    resolved["topic"] = "topic"

Priority 3: KNOWN ALIASES
    CSV has a column named "skill_name"?
    Our alias dictionary knows: skill_name → concept
    Result:    resolved["concept"] = "skill_name"
```

**Known alias dictionary:**
```
skill_name       → concept
skill            → concept
learning_objective → concept
objective        → concept
chapter          → topic
module           → topic
lesson           → topic
strand           → unit
area             → unit
domain           → subject
grade_label      → grade_hint
grade            → grade_hint
level            → grade_hint
class            → grade_hint
```

**5c. Validation:**
After resolution, the code checks:
- ❌ If `topic` is not resolved → **ValueError** with helpful error message
- ❌ If `concept` is not resolved → **ValueError** with helpful error message
- ✅ `subject`, `unit`, `grade_hint` are optional

**Final output example:**
```python
col_map = {
    'subject': 'subject',
    'unit': 'unit',
    'topic': 'topic',
    'concept': 'concept',
}
```

---

## Step 6: Row-by-Row Data Extraction

Now the orchestrator loops through every row of the DataFrame and extracts the actual values using the column map from Step 5:

```python
for _, row in df.iterrows():
    subject = row.get(col_map['subject'])     # "Mathematics"
    unit    = row.get(col_map.get('unit'))     # "Algebra" or None
    topic   = row.get(col_map['topic'])        # "Linear Equations"
    concept = row.get(col_map['concept'])      # "Solve for X"
    grade   = row.get(col_map.get('grade_hint'))  # "Grade 10" or None
```

**For each row, the code does:**

1. **Skip empty rows:** If `topic` or `concept` is blank, `NaN`, or `None` → skip
2. **Normalize subject:** If the CSV says "Maths" or "Math" → convert to "Mathematics"
3. **Build embedding text:**
   - If `unit` exists: `"Mathematics > Algebra > Linear Equations > Solve for X"` (4-level)
   - If no `unit`: `"Mathematics > Linear Equations > Solve for X"` (3-level)
4. **Set item_type:**
   - If `unit` column exists → `item_type = "concept"`
   - If no `unit` column → `item_type = "skill"`
5. **Append to `all_rows` list:**
```python
all_rows.append({
    'source_document_id': 3,
    'subject': 'Mathematics',
    'unit': 'Algebra',
    'topic': 'Linear Equations',
    'concept': 'Solve for X',
    'grade_hint': 'Grade 10',
    'item_type': 'concept',
    'embedding_text': 'Mathematics > Algebra > Linear Equations > Solve for X',
    'embedding': None,   # ← will be filled in Step 7
})
```

**Row limiting:**  
If you passed `--rows 5`, the code trims the list: `all_rows = all_rows[:5]`

---

## Step 7: Embedding Generation (`embedding_service.py`)

This is the AI-powered step. Each row's `embedding_text` gets converted into a 1024-dimensional dense vector.

**7a. Deduplication check:**
Before generating expensive embeddings, the code checks PostgreSQL for rows that were already embedded in a previous run:
```python
existing_keys = self.embedding_store.get_existing_curriculum_item_keys(source_document_id)
# Returns: {("Linear Equations", "Solve for X"), ("Shapes", "Identify squares")}

all_rows = [r for r in all_rows if (r['topic'], r['concept']) not in existing_keys]
```
This prevents wasting GPU/CPU time re-embedding rows that are already in the database.

**7b. Batch embedding:**
```python
embedding_texts = [r['embedding_text'] for r in all_rows]
# ["Mathematics > Algebra > Linear Equations > Solve for X",
#  "Mathematics > Geometry > Shapes > Identify squares"]

embeddings = self.embedding_service.embed_texts(embedding_texts, batch_size=128)
# Returns: [[0.0292, 0.0134, ...], [0.0189, 0.0344, ...]]
# Each inner list has exactly 1024 floats
```

The HuggingFace model `BAAI/bge-large-en-v1.5` runs **locally on your machine** (no API calls). It processes texts in batches of 128 for efficiency.

**7c. Attach embeddings to rows:**
```python
for row, emb in zip(all_rows, embeddings):
    row['embedding'] = emb  # Now each row dict has a 1024-float vector
```

---

## Step 8: PostgreSQL Insertion (`embedding_store.py`)

Finally, all rows are bulk-inserted into the `pipeline.curriculum_items` table:

```python
inserted = self.embedding_store.store_curriculum_items_batch(all_rows)
```

**What the SQL looks like internally:**
```sql
INSERT INTO pipeline.curriculum_items
    (source_document_id, subject, unit, topic, concept,
     grade_hint, item_type, embedding_text, embedding)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
ON CONFLICT (source_document_id, topic, concept) DO NOTHING;
```

**The `ON CONFLICT DO NOTHING` clause** is the safety net: if you accidentally run the same file twice, PostgreSQL silently ignores the duplicate rows instead of crashing.

---

## What Gets Stored — Final Result

After all 8 steps complete, here is exactly what sits in your PostgreSQL database:

### `pipeline.source_documents` (1 row per file)
| id | file_path | source_type | subject | processing_state |
|----|-----------|------------|---------|-----------------|
| 3 | /path/to/my_data.csv | excel | mathematics | scraped |

### `pipeline.curriculum_items` (1 row per skill)
| id | source_document_id | subject | unit | topic | concept | grade_hint | item_type | embedding_text | embedding |
|----|--------------------|---------|------|-------|---------|------------|-----------|---------------|-----------|
| 10 | 3 | Mathematics | Algebra | Linear Equations | Solve for X | Grade 10 | concept | Mathematics > Algebra > ... | [0.029, 0.013, ...] |
| 11 | 3 | Mathematics | Geometry | Shapes | Identify squares | NULL | concept | Mathematics > Geometry > ... | [0.018, 0.034, ...] |

---

## Complete Call Stack

Here is the exact function call chain from terminal to database:

```
User types: pipenv run python main.py run --subject math --file data.csv

1. main.py          → cli()               # Click parses arguments
2. main.py          → run()               # Validates, builds target dict
3. orchestrator.py  → run_pipeline()      # Connects to DBs, loads model
4. orchestrator.py  → _run_ingestion_stage()   # Loop over targets
5. orchestrator.py  → _ingest_target()         # Parse one CSV file
6. excel_parser.py  → parse_excel_file()       # Read CSV + auto-detect headers
7. raw_data_store.py → store_source_document() # Register file in PG
8. orchestrator.py  → _ingest_csv_to_curriculum_items()  # The big function
   ├── 8a. _resolve_column_map()          # Map CSV columns to internal fields
   ├── 8b. Loop rows → extract values     # Build all_rows list
   ├── 8c. Deduplication check            # Skip already-embedded rows
   ├── 8d. embedding_service.embed_texts() # Generate vectors (HuggingFace)
   └── 8e. embedding_store.store_curriculum_items_batch()  # Bulk insert to PG
```

---

## What Happens AFTER Ingestion?

Once ingestion (Stage 1) is complete, the pipeline automatically continues to the next stages:

| Stage | Name | What It Does |
|-------|------|-------------|
| **1** | **Ingestion** ← *you are here* | CSV → PostgreSQL + embeddings |
| 2 | Extraction | For each row, vector-search Neo4j → send to LLM → get skill decision |
| 3 | Graph Construction | Write LLM decisions as nodes/edges to Neo4j |
| 4 | Prerequisite Inference | Derive Concept-level prereqs from Skill-level ones |
| 5 | Validation | Check graph integrity (no cycles, no orphans) |

Each subsequent stage reads from what the previous stage wrote, creating a clean data pipeline from raw CSV to finished Knowledge Graph.
