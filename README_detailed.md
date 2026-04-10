# survey_data_cleaning

**DEP Annual Survey 2026 — Data Cleaning & Pipeline Notebook**

---

## Problem

Raw survey exports from Google Forms or similar platforms are not analysis-ready. They arrive with inconsistent column headers, free-text responses in structured fields, comma-delimited multi-select answers packed into single cells, messy location data, and no respondent identifiers. Downstream chart scripts and crosstab tables depend on clean, standardized values — anything less produces incorrect percentages or broken joins.

This notebook addresses that gap end-to-end: from raw `.csv` ingestion through a fully structured data mart ready for analytical consumers.

---

## What This Notebook Does

The notebook runs a sequential pipeline with ten numbered stages. Each stage produces intermediate checkpoints (`.csv`, `.parquet`) so individual stages can be rerun independently without re-executing the full pipeline.

| Stage | Description |
|---|---|
| **0** | Imports, config, directory scaffolding |
| **1** | Raw data ingestion, column renaming (raw → transformed) |
| **2** | First global cleaning pass — strip, normalize font, collapse whitespace, title-case, fill nulls |
| **3** | Respondent ID assignment, null consolidation, location integrity checks, manual spot edits |
| **4** | Lookup file generation for single-response columns — three edit tiers |
| **5** | Lookup application, similarity-based preliminary duplicate detection |
| **6** | Derived grouping columns — age bands, salary bands, PH region classification |
| **7** | Multi-response column processing — explode, normalize, theme-classify, duplicate check |
| **8** | Location enrichment — capital city fallback, geocoding (PH and non-PH), Folium map |
| **9** | DuckDB and SQLite data mart assembly |
| **10** | Data mart promotion and final outputs export |

---

## Key Technical Methods

### Column Standardization
Columns are renamed via an explicit `rename_map` dictionary at ingest time, mapping verbose survey question text to short, consistent snake_case keys. This keeps all downstream references stable regardless of how the survey form's export headers change year to year.

### Three-Tier Lookup System
Single-response non-numeric columns are cleaned through a structured lookup workflow:

- **Type 1 — Direct clean:** Raw value equals display value. No manual edits needed.
- **Type 2 — Specific edits:** Known remappings (e.g., `salary`, `have_comp_degree`) that require different stored vs. displayed labels. Applied programmatically via defined maps.
- **Type 3 — Manual edits:** Responses such as free-text degree fields where automated classification fails. Human-reviewed CSV files in `lookup_dir/`.

A single `apply_lookup()` function handles all three types uniformly once lookup files are finalized.

### Multi-Response Explode & Normalize
Multi-select answers (tools, platforms, methods) are pipe- or semicolon-delimited strings. Each column is exploded into one row per respondent-item pair, then normalized to a consistent casing and trimmed form. Edge cases like MS Fabric (which contains a comma in its name) are handled with targeted pre-clean substitutions before splitting.

### Substring-Based Theme Classification
Attended event columns (`attended_online`, `attended_inperson`) contain open-ended free text. A priority-ordered dictionary of substring themes maps responses to standardized categories (e.g., `"pycon"` → `pycon_python_events`). Unmatched responses fall through to `"Other"` for manual review.

### Similarity-Based Duplicate Detection
After single-response cleaning, respondents are scored for similarity across key demographic columns using a pairwise comparison approach. Pairs scoring 1.0 on single-response fields are flagged and cross-checked against multi-response columns. A respondent pair is only confirmed as a likely duplicate if their multi-response answers also match — reducing false positives from coincidental demographic overlap.

### Derived Grouping Columns
Analytical groupings are added post-cleaning rather than sourced from the raw data:

- `age_grp` — binned from numeric `age`
- `salary_broader` — collapsed salary bands using `match-case` logic
- `region_ph` — Philippine city mapped to region via a Gemini-assisted lookup CSV, with Unknown values backfilled to `"Not Applicable"`

### Geocoding Pipeline
Location data is enriched in two passes:

1. **Capital city fallback** — Non-PH respondents with suppressed or missing cities are assigned their country's capital via a lookup dictionary.
2. **Nominatim geocoding** — Both PH and non-PH cities are geocoded using `geopy` with `RateLimiter` to respect API limits. Results feed a Folium interactive map (`location_map_2026.html`). Known geocoding anomalies (e.g., misplaced Zamboanga coordinates) are corrected with manual overrides.

### Data Mart Export (DuckDB + SQLite)
All cleaned tables — `df_single_with_grps`, plus one exploded table per multi-response column — are registered into a DuckDB database (`survey2026.duckdb`). A SQLite export is also produced for lightweight side exploration. Data mart promotion follows a staging → production copy pattern controlled by an `ARTIFACTS` config dictionary.

---

## Output Artifacts

| Artifact | Description |
|---|---|
| `df_raw.parquet` | Cleaned raw respondent table with IDs |
| `df_single_with_grps.parquet / .csv` | Single-response columns with all derived group columns |
| `df_multi.csv` | Multi-response columns pre-explode |
| `*_exploded_cleaned.csv` | One file per multi-response column, post-explode |
| `df_possible_duplicates.csv` | Flagged respondent pairs for review |
| `location_map_2026.html` | Interactive Folium map of respondent locations |
| `survey2026.duckdb` | Full data mart (DuckDB) |
| `survey2026.sqlite` | Full data mart (SQLite) |
| `survey2026_data_mart.xlsx` | Excel export of key tables |
| `final_outputs/` | Promoted clean outputs for downstream consumers |

---

## Dependencies

```
pandas
numpy
unicodedata (stdlib)
pathlib (stdlib)
secrets (stdlib)
geopy
folium
duckdb
openpyxl
thefuzz (fuzzy matching)
google-genai (Gemini API, used for city-region lookup generation)
```

---

## Directory Structure

```
project_root/
├── csv_raw.csv                  # Source: raw survey export
├── lookup_dir/                  # Finalized lookup CSVs (single-response)
│   ├── type1/                   # Direct clean
│   ├── type2/                   # Specific edits
│   └── type3/                   # Manual edits
├── csv_outputs_dir/             # Intermediate CSV checkpoints
├── parquet_outputs_dir/         # Intermediate Parquet checkpoints
├── location_dir/                # Geocoding intermediates and map output
├── data_mart_staging/           # Pre-promotion data mart files
├── data_mart/                   # Promoted final data mart
│   ├── excel/
│   ├── duckdb/
│   └── sqlite/
└── final_outputs/               # Files consumed by chart/report scripts
```

---

## Usage Notes

- **Rerunning individual stages:** Each stage reads from a checkpoint file. To rerun stage 7 only, reload `df_raw.parquet` at the top of that section — no need to re-execute stages 1–6.
- **Lookup file edits:** Type 3 lookup CSVs in `lookup_dir/` require human review before `apply_lookup()` is called. The notebook will flag unmatched values in the output so gaps are visible.
- **Geocoding runtime:** Stage 8 geocoding takes approximately 35–40 minutes due to Nominatim rate limits. Run it separately or skip it if location data is not needed for the current output.
- **Duplicate resolution:** `df_possible_duplicates.csv` is a flagged list, not a removal list. Manual confirmation is required before any respondent is dropped.

---

*Part of the DEP Annual Survey 2026 data pipeline. Built by [Sandy G. Cabanes](https://linkedin.com/in/sandygcabanes).*
