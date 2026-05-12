# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Spark Issue Triage Platform** — automated triage of Apache Spark JIRA tickets built on a Snowflake medallion architecture (Bronze → Silver → Gold) orchestrated by dbt. Two Streamlit apps expose the results: a real-time KNN inference app and a 5-page analytics dashboard.

Source data: Kaggle Apache Spark JIRA dump (March 2025, ~49,832 SPARK tickets filtered from 1.15M total).

---

## Environment Setup

### Python environments

There are **two separate Python environments**:

| Environment | Path | Purpose |
|-------------|------|---------|
| Root venv | `venv/` (Python 3.11+) | Load scripts, Streamlit apps |
| dbt venv | `dbt_project/.venv/` (Python 3.12) | dbt-snowflake only |

Activate root env: `venv\Scripts\Activate.ps1`

Install app dependencies:
```bash
pip install -r apps/inference/requirements.txt
pip install -r apps/analytics/requirements.txt
```

### Configuration

Copy `.env.example` to `.env` and fill in Snowflake credentials:
```
SNOWFLAKE_ACCOUNT=...
SNOWFLAKE_USER=...
SNOWFLAKE_PASSWORD=...
SNOWFLAKE_ROLE=SYSADMIN
SNOWFLAKE_WAREHOUSE=PFE_WH
SNOWFLAKE_DATABASE=PFE_SPARK
ANTHROPIC_API_KEY=sk-ant-...   # optional — enables LLM analysis in inference app
```

---

## Running the Applications

### Docker (recommended)
```bash
docker-compose up --build
# inference app → http://localhost:8501
# analytics dashboard → http://localhost:8502
```

### Manual
```bash
streamlit run apps/inference/inference_app.py   # port 8501
streamlit run apps/analytics/analytics_app.py   # port 8502
```

---

## dbt Commands

All dbt commands run from `dbt_project/` using the `.venv` Python:

```bash
cd dbt_project
.venv\Scripts\dbt deps         # install dbt-utils
.venv\Scripts\dbt seed         # load seeds/issuetype_mapping.csv + resolution_mapping.csv
.venv\Scripts\dbt run          # run all 11 models
.venv\Scripts\dbt test         # 46 tests — expected: PASS=44 WARN=2 ERROR=0
.venv\Scripts\dbt run -s staging   # run a specific layer
.venv\Scripts\dbt test -s marts    # test a specific layer
```

The 2 expected WARNs are `accepted_values` tests on "Won't Fix" — the apostrophe is a known SQL quoting limitation; the label is handled correctly in the pipeline.

---

## Full Data Pipeline (first-time setup)

Run in order:

```bash
# 1. Write ~/.dbt/profiles.yml from .env
python load/write_profiles.py

# 2. Create Snowflake database PFE_SPARK, 6 schemas, warehouse PFE_WH, stage RAW.CSV_STAGE
python load/run_phase1.py

# 3. Verify CSV column positions match load/04_copy_into_raw.sql before loading
python load/inspect_headers.py

# 4. Upload CSVs to @RAW.CSV_STAGE (~30 min), then COPY INTO raw tables
python load/03_put_files.py
python load/run_phase4.py

# 5. dbt transformations (see dbt commands above)

# 6. ML pipeline: embeddings + KNN predictions + evaluation + upload to Snowflake
python load/run_ml_pipeline.py
```

`run_ml_pipeline.py` generates `results/embeddings_cache.npz` (57 MB, versioned in git). If the cache exists it is reused; delete it to force a full re-embed (~10 min on CPU).

---

## Architecture

### Medallion layers in Snowflake (`PFE_SPARK` database)

```
Bronze  RAW.*          — raw VARCHAR tables, no transformation
Silver  STAGING.*      — dbt views: filter project='SPARK', rename, cast timestamps
        INTERMEDIATE.* — dbt tables: NLP cleaning, feature engineering, temporal split
Gold    MARTS_ML.*     — MART_ML (42,083 rows, train + validation)
        MARTS_ANALYTICS.* — MART_ANALYTICS_OPS + MART_ANALYTICS_DEPS
        CORTEX.*       — MART_PREDICTIONS (3,809 validation predictions)
```

### Snowflake schemas and their dbt materializations

| dbt layer | Schema | Materialization |
|-----------|--------|-----------------|
| staging | STAGING | view |
| intermediate | INTERMEDIATE | table |
| marts/ml | MARTS_ML | table |
| marts/analytics | MARTS_ANALYTICS | table |

### ML inference pipeline (`apps/inference/inference_app.py` + `load/run_ml_pipeline.py`)

1. Input ticket formatted as `text_noco` string:
   ```
   TICKET: {summary}
   TYPE: Unknown | PRI: {priority}
   STATUS: {status}
   DESC: {description[:800]}
   ```
   truncated to 2,000 characters.

2. Embedded with `sentence-transformers/all-MiniLM-L6-v2` → 384-dim L2-normalized vector

3. Cosine similarity (dot product) against 38,274 train embeddings

4. Metadata boost: +0.10 same priority, +0.08 same status, +0.05 same reporter

5. Top-15 neighbors (k=15), weighted vote → issuetype + resolution prediction

6. Confidence thresholds: High ≥65%, Medium 45–64%, Low <45%

7. LLM analysis: `claude-haiku-4-5-20251001` via Anthropic API if `ANTHROPIC_API_KEY` is set; otherwise a deterministic template is used.

### Key design decisions (see `docs/decisions_log.md` for full rationale)

- **All RAW tables are VARCHAR** — avoids COPY INTO failures on malformed Kaggle CSV data; casts happen in staging via `TRY_TO_*` functions.
- **Temporal split** — `train` = tickets before 2023-01-01 (38,274), `validation` = 2023 tickets (3,809), `excluded` = 2024+ (~3,700). Tickets with NULL resolution are excluded (open issues have no known outcome).
- **Snowflake Cortex SQL** (`cortex/*.sql`) is the original spec but blocked on trial accounts. The Python KNN pipeline is the active implementation.
- **Embeddings cache** (`results/embeddings_cache.npz`) is versioned in git (57 MB < GitHub 100 MB limit) so Docker containers and collaborators start instantly without re-embedding.
- **Deduplication**: `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY id) = 1` in `stg_issues` handles 4 duplicate keys in the Kaggle export.

### dbt macros

- `clean_jira_text(col_expr)` — 6-step NLP pipeline: strips HTML tags, `{code}` blocks, `{noformat}` blocks, `[~user]` mentions, URLs, and collapses whitespace. Used in `int_issues_cleaned` and `int_comments_aggregated`.
- `generate_schema_name` — overrides dbt schema naming to use the target schema directly without prefixing the profile target name.
- `consolidate_label` — LEFT JOINs raw label values against seed mapping tables (`issuetype_mapping.csv`, `resolution_mapping.csv`). Values absent from the seed produce NULL and are filtered out in subsequent models.

### Label vocabularies

**9 issuetype classes**: Bug, Improvement, Sub-task, New Feature, Task, Test, Documentation, Question, Other

**7 resolution classes**: Fixed, Won't Fix, Not A Problem, Incomplete, Duplicate, Invalid, Cannot Reproduce

---

## Project Structure (key files)

```
load/
  run_phase1.py          # creates Snowflake DB + schemas + warehouse
  03_put_files.py        # uploads CSVs to @RAW.CSV_STAGE
  run_phase4.py          # COPY INTO + row count verification
  run_ml_pipeline.py     # full KNN train/eval/upload pipeline
  write_profiles.py      # generates ~/.dbt/profiles.yml from .env

dbt_project/
  models/staging/        # 4 views (1:1 with raw tables)
  models/intermediate/   # 4 tables (NLP, features, temporal split)
  models/marts/ml/       # MART_ML — the ML contract table
  models/marts/analytics/# MART_ANALYTICS_OPS + DEPS
  seeds/                 # issuetype_mapping.csv, resolution_mapping.csv
  macros/                # clean_jira_text, generate_schema_name, consolidate_label

apps/inference/
  inference_app.py       # Streamlit UI: form → predict() → display
apps/analytics/
  analytics_app.py       # landing page
  pages/1_overview.py    # monthly volumes by issuetype
  pages/2_resolution_dynamics.py
  pages/3_workload.py    # per-assignee metrics
  pages/4_relationships.py  # issuelink graph analysis

cortex/                  # Snowflake Cortex SQL pipeline (paid account only)
results/
  embeddings_cache.npz   # pre-computed train embeddings (57 MB, versioned)
```
