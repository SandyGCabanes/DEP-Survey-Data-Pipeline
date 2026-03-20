# DEP Survey Data Pipeline
## 2025–2026 Data Engineering Pilipinas State of the Community Survey

End-to-end data cleaning, transformation, and export pipeline for the DEP annual community survey. Processes raw Google Forms responses into analysis-ready outputs across multiple formats for Tableau, SQL, and Python consumers.

**Pipeline architect and developer:** [Sandy G Cabanes](https://www.linkedin.com/in/sandygcabanes)

See the companion report: [2025–2026 DEP State of the Community Survey Results](https://sandygcabanes.github.io/2025-2026-DEP-State-of-the-Community-Survey-Results/)

---

## Tech Stack

| Purpose | Tools |
|---|---|
| Pipeline development | Python (pandas, numpy) |
| Notebook environment | Jupyter Notebook |
| Duplicate detection | Similarity scoring (custom) |
| Geographic processing | geopy, folium |
| Data mart | DuckDB, SQLite |
| Tableau-ready export | Excel (openpyxl) |
| Serialization | Parquet (pyarrow) |

---

## Pipeline Overview

The pipeline is organized into five families, each producing outputs consumed by downstream stages or directly by the report and dashboard layers.


```
RAW LAYER
    └── df_raw
SINGLE-RESPONSE FAMILY
    └── df_single_no_grps → df_single_with_grps
MULTI-RESPONSE FAMILY
    └── df_raw_multi → df_multi → {col}_exploded_cleaned
LOCATION FAMILY (AI & API Layer)
    ├── Gemini AI (Regional Classification)
    ├── Geopy API (Coordinate Enrichment)
    └── Folium (Visual HITL Validation)
DATA MART EXPORT LAYER
    └── DuckDB, SQLite, Excel, Parquet
```

---

## Stage 1: Raw Layer

**Input:** `csv_raw.csv` (raw Google Forms export)

**Operations:**
- Drop administrative columns: Timestamp, Email, End of survey
- Normalize unicode, strip whitespace, collapse spaces
- Replace open-ended "Other…" entries with blanks for downstream lookup handling
- Fix age outliers: replace 0 with minimum valid age, cap at 95
- Replace empty strings with nulls
- Add `resp_id` and `resp_id_rand` identifiers
- Apply geographic rule: if country is not Philippines, set `city_ph` to Not Applicable
- HITL - human edits for some respondents who filled up province in city_ph and PH city in city_non_ph

**Outputs:**
- `csv_outputs_dir/df_raw.csv`
- `parquet_outputs/df_raw.parquet`


---

## Stage 2: Single-Response Family

Handles all questions with a single answer per respondent.

### 2a. Value Standardization

**Input:** `df_raw` (single-response columns subset)

**Operations:**
- Export raw unique values per column to `unique_dir/*.csv`
- Manually review and create lookup tables in `lookup_dir/*_lookup.csv`
  - Some columns require human edits (free-text responses, AI-assisted matching not reliable)
  - Some columns require special label formatting
  - Some columns pass through directly without edits
- Apply `apply_lookup()` to map raw values to clean labels

### 2b. Duplicate Detection

**Operations:**
- Compute similarity scores across respondents
- Produce `df_possible_duplicates` and `df_likely_dups` for analyst review

**Outputs:**
- `csv_outputs_dir/df_single_no_grps.csv`
- `csv_outputs_dir/df_sim_sorted.csv`
- `csv_outputs_dir/table_of_likely_dups.csv`
- `csv_outputs_dir/df_possible_duplicates.csv`

### 2c. Age, Salary and PH Region Grouping

**Input:** `df_single_no_grps`

**Operations:**
- Create `age_grp` via `pd.cut()` into standard age bands
- Create `salary_broader` via match-case function for cross-role salary comparison
- Create `region_ph` using Gemini API and human inspection

**Outputs:**
- `csv_outputs_dir/df_single_with_grps.csv`
- `final_outputs_dir/df_single_with_grps.csv`
- `parquet_outputs_dir/df_single_with_grps.parquet`

**Summary output:**
- `csv_outputs_dir/summary_single_response_no_grps.csv`
- `csv_outputs_dir/summary_single_response_eda.txt`

---

## Stage 3: Multi-Response Family

Handles all questions where respondents could select multiple answers.

**Input:** `df_raw` (multi-response columns subset)

**Operations:**
- Clean MS Fabric value labels
- Clean `attended_inperson` and `attended_online` columns
- Explode each multi-response column via `explode_and_normalize()`:
  - Split by comma
  - Normalize text
  - Drop duplicates
- Clean `restofrole`: remove answers already captured in `datarole` to prevent double-counting
- Second-round cleaning available if needed

**Outputs per column:**
- `csv_outputs_dir/{col}_exploded.csv`
- `csv_outputs_dir/{col}_exploded_cleaned.csv`
- `parquet_outputs_dir/{col}_exploded_cleaned.parquet`

**Summary output:**
- `csv_outputs_dir/multi_response_summary.csv`

---

## Stage 4: Duplicate Check (Combined)

Cross-validates duplicates across both single-response and multi-response families.

**Operations:**
- Explode, sort, and join all multi-response columns into a single string per respondent
- Merge with `df_likely_dups` from the single-response stage
- Boolean similarity check across multi-response columns
- Flag respondents where `count_false > 0` as likely duplicates
- Output is inline table of duplicate



---
## Stage 5: Location Family
Builds the geographic dataset powering the interactive respondent map.

#### 5a. AI Regional Classification
**Input:** `df_single_with_grps` (Unique City/Province strings)
**Operations:**
* **Gemini AI Integration:** Utilizes the Gemini LLM to categorize unstructured location strings into five specific Philippine regions (Metro Manila, Balance Luzon, Visayas, Mindanao, or Unknown).
* **Structured Output:** Implemented using **Pydantic schemas** to ensure the AI returns valid, type-safe JSON for seamless dataframe merging.
* **Batch Processing:** Optimized API calls by batching 50 locations per request to manage latency and rate limits.

### 5b. Geospatial API Enrichment
**Operations:**
* **Geopy API:** Automated the retrieval of precise Latitude and Longitude coordinates for each AI-categorized city.
* **Validation Layer:** Cross-references the **AI-predicted region** against the **API-provided coordinates**.
* **Human-in-the-Loop (HITL):** Generates a **Folium** map for visual inspection, allowing for manual correction of spatial outliers before final export.

**Outputs:**
* `location_dir/df_all_geo_clean.csv` (Enriched with AI regions and API coordinates)
*  `location_map_2026.html`
---

## Stage 6: Data Mart Export Layer

Consolidates all cleaned outputs into analysis-ready formats for multiple consumers.
Data_mart_staging and data_mart for storage

### DuckDB
- `survey.duckdb`
- Tables: `df_single_with_grps`, all `{col}_exploded_cleaned` tables, `df_all_geo_clean`

### SQLite
- `survey.sqlite`
- Same tables as DuckDB
- Used for fast ad-hoc SQL checks during analysis

### Excel (Tableau-ready)
- Combines `df_single_with_grps`, all `*_exploded_cleaned` files, and `df_all_geo_clean`
- Output: `final_outputs_dir/for_tableau.xlsx`

### Final Consumption
- Tableau connects to DuckDB or `for_tableau.xlsx`
- SQLite used for quick SQL validation queries
- Python [report](https://sandygcabanes.github.io/2025-2026-DEP-State-of-the-Community-Survey-Results/) scripts consume `final_outputs_dir/df_single_with_grps.csv` and `*_exploded_cleaned.csv` directly

---

## Final Output Inventory

```
csv_outputs_dir/
    df_raw.csv
    df_single_no_grps.csv
    df_single_with_grps.csv
    df_raw_multi.csv
    df_multi.csv
    {col}_exploded_cleaned.csv
    multi_response_summary.csv
    df_sim_sorted.csv
    df_likely_dups.csv
    df_possible_duplicates.csv

location_dir/
    df_all_geo_clean.csv
    location_map_2026.html

parquet_outputs_dir/
    df_raw.parquet
    df_single_with_grps.parquet
    df_raw_multi.parquet
    {col}_exploded_cleaned.parquet
    {col}_exploded.parquet

final_outputs_dir
    all files for the report*

data_mart_staging/
    for_tableau.xlsx
    survey.duckdb
    survey.sqlite

data_mart/
    for_tableau.xlsx
    survey.duckdb
    survey.sqlite

```
*[report](https://sandygcabanes.github.io/2025-2026-DEP-State-of-the-Community-Survey-Results/)

---

## Privacy

Raw response data is not published in this repository. All public data releases use a synthetic dataset generated via Bayesian network model methods, preserving the community's statistical profile while protecting individual respondent privacy.

---

## Contact

Interested in survey data pipeline design, cleaning architecture, or analytics engineering for your organization?

**Sandy G Cabanes** · [LinkedIn](https://www.linkedin.com/in/sandygcabanes) · Data Analyst & Pipeline Developer
