# Python workflow represented by a block diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                   survey_data_cleaning  — Pipeline Overview         │
└─────────────────────────────────────────────────────────────────────┘

 ┌───────────────────────────────────┐
 │  STAGE 0 — Setup                  │
 │  Imports · Config · Directories   │
 │  lookup_dir/ · parquet_dir/       │
 │  csv_outputs_dir/ · location_dir/ │
 └──────────────┬────────────────────┘
                │
                ▼
 ┌───────────────────────────────────┐
 │  STAGE 1 — Ingest & Rename        │
 │  csv_raw.csv → df_raw             │
 │  rename_map: verbose → snake_case │
 └──────────────┬────────────────────┘
                │
                ▼
 ┌───────────────────────────────────┐
 │  STAGE 2 — Global Cleaning        │
 │  Strip · normalize font           │
 │  Collapse spaces · title-case     │
 │  Fill nulls → "Not Applicable"    │
 └──────────────┬────────────────────┘
                │
                ▼
 ┌───────────────────────────────────┐
 │  STAGE 3 — IDs & Location Checks  │
 │  Add resp_id, resp_id_rand        │
 │  Null audit · city integrity      │
 │  Manual spot edits (city_ph)      │
 │  → df_raw.parquet  *              │
 └──────────────┬────────────────────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
┌────────────────┐  ┌────────────────────────────────────┐
│  SINGLE-RESP   │  │  MULTI-RESP                        │
│  PIPELINE      │  │  PIPELINE                          │
│                │  │                                    │
│ STAGE 4        │  │ STAGE 7                            │
│ Lookup Files   │  │ Filter multi cols → df_multi       │
│ ┌────────────┐ │  │ Pre-clean: MS Fabric comma fix     │
│ │ Type 1:    │ │  │ Fuzzy match: attended cols         │
│ │ Direct     │ │  │                                    │
│ │ Type 2:    │ │  │ explode_and_normalize()            │
│ │ Specific   │ │  │ → one CSV per column               │
│ │ Type 3:    │ │  │                                    │
│ │ Manual     │ │  │ Theme classify:                    │
│ └────────────┘ │  │  attended_online  (substring map)  │
│                │  │  attended_inperson (substring map) │
│ STAGE 5        │  │                                    │
│ apply_lookup() │  │ datarole / restofrole dedup        │
│ Similarity     │  │                                    │
│ duplicate flag │  │ Second-pass normalize              │
│ → df_single    │  │ → *_exploded_cleaned.csv  *        │
│   _no_grps     │  └────────────┬───────────────────────┘
│   .parquet *   │               │
└───────┬────────┘               │
        │                        │
        ▼                        │
┌────────────────┐               │
│ STAGE 6        │               │
│ Derived Groups │               │
│ age_grp        │               │
│ salary_broader │               │
│ region_ph      │               │
│ (Gemini lookup │               │
│  + Nominatim)  │               │
│ → df_single    │               │
│   _with_grps   │               │
│   .parquet *   │               │
└───────┬────────┘               │
        │                        │
        ▼                        │
┌───────────────────────────────────┐
│  STAGE 7i — Full Duplicate Check  │
│  Single + multi combined scoring  │
│  → df_possible_duplicates.csv *   │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│  STAGE 8 — Location Enrichment    │
│  Capital city fallback (non-PH)   │
│  Nominatim geocode: PH + non-PH   │
│  Manual anomaly corrections       │
│  → df_all_geo_clean.csv *         │
│  → location_map_2026.html *       │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│  STAGE 9 — Data Mart Assembly     │
│  DuckDB: all tables registered    │
│  SQLite: lightweight mirror       │
│  → survey2026.duckdb *            │
│  → survey2026.sqlite *            │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│  STAGE 10 — Promotion & Export    │
│  staging/ → data_mart/            │
│  Excel export of key tables       │
│  final_outputs/ copy              │
│  → survey2026_data_mart.xlsx *    │
│  → final_outputs/ *               │
└───────────────────────────────────┘

  * = checkpoints with print()
```
