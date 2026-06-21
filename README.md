
# KSAMDS-Capstone

This repository contains the Knowledge, Skills, and Abilities Multi-Dimensional Structure (KSAMDS) project developed as the Fall 2025 DAEN Capstone at George Mason University. KSAMDS is a structured knowledge representation and search system for workforce competencies that integrates multiple public and proprietary data sources, applies NLP and embedding-based inference, and loads the results into a dimensionally modeled PostgreSQL database.

**Short summary:** KSAMDS constructs a multidimensional competency database (knowledge, skills, abilities, functions, tasks, and occupations), augments records with synthetic attributes, infers semantic relationships using embeddings, and exposes a search API for downstream analytics and applications.

## Objectives

- Build a clean, idempotent ETL pipeline that maps external KSA sources (O*NET and Skills Framework) into a unified KSAMDS schema.
- Enrich and standardize KSA entities with dimensional attributes (type, level, basis, environment, mode, physicality, cognitive) to support advanced filtering and analytics.
- Infer semantic links between entities (e.g., knowledge → skill, skill → ability, function → task) using embedding similarity and store relationship confidence scores.
- Provide a searchable API and tooling to query, filter, and explore the dataset for workforce planning and training design.

## Data Sources

- O*NET (US Department of Labor) — primary raw source for knowledge, skills, abilities, tasks, and occupations.
- Skills Framework datasets — complementary competency lists and relationships.
- Project-specific web scraping (optional) — to expand coverage and support multiple languages / romanization.

All source artifacts and processed CSVs are stored under the `data/` directory. The pipeline stages read/write intermediate CSVs at `data/<source>/archive/{intermediate,mapped,relationships,embeddings}`.

## Key Methodologies & Architecture

- ETL Stages (per-source): Extract → Synthetic Attribute Generation → Mapping → Relationship Inference → Load → Validate.
- Deterministic IDs: Entities are assigned deterministic UUIDs (UUIDv5 with a fixed namespace) so mapping is idempotent across runs.
- Dimensions Model: A set of dimension tables (`type_dim`, `level_dim`, `basis_dim`, `environment_dim`, `mode_dim`, `physicality_dim`, `cognitive_dim`) enables faceted filtering and consistent joins between entities and occupations.
- Embedding-based Inference: Uses Google Generative Embeddings (Gemini embeddings) to generate semantic vectors, then computes cosine similarity to infer relationships; embeddings are cached to disk to reduce API calls.
- Synthetic Attribute Assignment: A lightweight classifier-like approach assigns `type` / `basis` / `mode` / `environment` attributes by comparing entity embeddings to reference category embeddings.
- Fuzzy Deduplication: Basic fuzzy string matching groups near-duplicate entity names during mapping to reduce redundancy.
- Loader: Uses batched `psycopg2` inserts with `ON CONFLICT DO NOTHING` to support safe incremental loads.
- Validation: Post-load validators run a suite of checks (entity counts, uniqueness, dimension coverage, orphaned relationships, inferred-relationship quality) and write human-readable and JSON reports.

## Repository Structure (high level)

- `data/` — raw, intermediate and processed CSVs, embedding caches, logs, and reports
- `src/` — source code
	- `src/backend/api/` — FastAPI app, routers, services, and DB helpers
	- `src/backend/onet_etl/` — O*NET ETL pipeline modules (extract, datagen, mapper, relations, loader, validator, orchestrator)
	- `src/backend/skillsframework_etl/` — Skills Framework ETL (extract, datagen, mapper, relations, loader, validator, orchestrator)
	- `src/database/schema/` — database DDL (`KSAMDS_DB.sql`)
    - `src/frontend/` - Frontend HTML, CSS, and JavaScript Code
- `notebooks/` — exploratory notebooks and visualizations
- `docs/` — longer-form reports and references

## Quickstart / Local Setup

Prerequisites:

- Python 3.10+ (recommend using a virtual environment)
- PostgreSQL (12+ recommended) with a database and a user for KSAMDS
- A valid `GOOGLE_API_KEY` to run embedding-based datagen or relationship inference

1. Create and activate a virtual environment, install dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

2. Create the KSAMDS schema in your PostgreSQL instance (run from repository root):

```bash
# set env vars used by psql or pass connection details directly
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f src/database/schema/KSAMDS_DB.sql
```

3. Create a `.env` file (or export env vars) for the API and loaders. Example `.env` values:

```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ksamds
DB_USER=ksamds_user
DB_PASSWORD=secret_password
DB_SCHEMA=ksamds

# Google embeddings (required for datagen/relations)
GOOGLE_API_KEY=your_google_api_key_here
```

4. Run ETL pipelines (examples):

- O*NET pipeline (from `src/backend/onet_etl`):

```bash
cd src/backend/onet_etl
python onet_orchestrator.py
```

- Skills Framework pipeline (from `src/backend/skillsframework_etl`):

```bash
cd src/backend/skillsframework_etl
python skillsframework_orchestrator.py
```

Notes:
- The orchestrators accept CLI flags to skip stages, reuse cached embeddings, or avoid cleanup (`--skip-datagen`, `--skip-mapping`, `--no-cleanup`, etc.).
- Embedding steps require a valid `GOOGLE_API_KEY` set as an environment variable.
- Loading step requires a valid `DB_USER` and `DB_PASSWORD` set as an environment variable.

## Running the API (development)

From the repository root you can run the FastAPI app (development mode):

```bash
# using the repository root so imports resolve correctly
python -m uvicorn src.backend.api.main:app --reload --host 0.0.0.0 --port 8000
```

API endpoints include search, filter listing, and entity relationship lookups under the configured API prefix (see `src/backend/api/config.py`).

## Frontend (development)

A small static frontend is included in the repository at `src/frontend/` (contains `index.html`, `script.js`, and `style.css`). It is suitable for development and demo purposes and communicates with the backend API when the API is running.

To serve the frontend locally on port `3000` using Python's built-in HTTP server, run from the repository root:

```bash
cd src/frontend
# Python 3: serves on http://0.0.0.0:3000/
python -m http.server 3000
```

Open `http://localhost:3000` in your browser. If you run the FastAPI backend locally on port `8000` (see above), the frontend should be able to reach the API endpoints — confirm the API host/port in `src/frontend/script.js` if needed.

## Validation & Reports

- After loading, the ETL writes validation and pipeline reports to `data/<source>/reports/` (e.g., `data/onet/reports/latest_validation_report.txt` and `data/skillsframework/reports/latest_sf_pipeline_report.txt`). Review these for data quality metrics and pipeline diagnostics.

## Development notes

- Deterministic UUIDs ensure idempotent mapping — re-running mappers will reuse the same entity IDs when input names are unchanged.
- Embeddings are cached in `data/*/archive/embeddings/` to minimize API usage and support reproducibility.
- Loader scripts use batched inserts and `ON CONFLICT DO NOTHING` to safely re-run loads without duplicates.

## Contributing

If you'd like to contribute, please open issues or pull requests. Suggested next steps:

- Add unit tests for mapping and loader logic.
- Improve fuzzy duplicate detection and entity normalization.
- Add CI jobs to run a lightweight integration test that uses an in-memory or temporary Postgres instance.

## Contact

Project maintainers: see repository owner. For questions about running the ETL or API, open an issue with a short description and logs.

## License

This project is licensed under the Apache License 2.0.

