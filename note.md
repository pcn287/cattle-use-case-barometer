# DECIDE barometer — GD API and pipeline notes

## GD API download (local)

**Folder:** `GD API data query/` (repo root, next to `pipeline/`)

- **`GD API.ipynb`** — pulls lab results from the GD Animal Health DECIDE API and writes:
  - `downloads/gd_labresults.csv`
  - `downloads/gd_labresults_raw.json`
- Credentials live in **`.env`** (copy from `.env.example`; do not commit `.env`).
- Before overwriting, the notebook compares **SHA-256** of the new CSV to the existing file:
  - Same hash → message that no new download is needed.
  - Different hash → saves files and reports a newer version.
- Run **Cell 2** then **Cell 3** for routine updates (Cell 1 only if packages are missing).

**Upload to Databricks:** copy `gd_labresults.csv` to the volume (e.g. via Teams):

`/Volumes/decide_catalog/decide_schema/decide_volume/gd_labresults.csv`

---

## Pipeline changes (`pipeline/`)

### New / updated source file

| File | Role |
|------|------|
| `gd_labresults.csv` | GD API export (CSV). **Required** in the volume, same as the xlsx/csv sources. |

### `00_check_source_changes`

- `gd_labresults.csv` is in **`SOURCE_FILES`**.
- Any change to its **size** or **modification time** sets `should_run=true` and runs the full job (same rule as other sources).

### `03_clean_gd`

Builds **`decide_catalog.decide_schema.barometer_gd`** from **two** inputs, then **combines** them:

1. **`250808_data_RGD_DECIDE.xlsx`** (Excel, wide → long, same logic as `GD.Rmd` on `main`).
   - `Samplenumber` = SHA-256 of Excel **`sample_id`** (1, 2, 3… within each `Dossier_ID`).

2. **`gd_labresults.csv`** (API, already long).
   - No `Dossier_ID` or `sample_id` in the API spec.
   - **`Samplenumber` = SHA-256 of** `farmID|testDate|diagnosticTest|sampleType`  
     (documented **API-derived** sample key; not GD internal `sample_id`).
   - `Farm_ID` = API `farmID` as-is (already encrypted by GD).
   - `testDate` → `Date`, plus `Date_month` / `Date_week`.

Both paths are **required** in the volume. Rows are merged with `pd.concat` and written to one Delta table. Logs include row counts per source (`excel` / `api`).

### `07_union_all`

- Updates `source_file_state` including **`gd_labresults.csv`** after a successful run (so the next `00` check is correct).

### Union

- **`07_union_all`** is unchanged in role: it still unions the six `barometer_*` Delta tables (including `barometer_gd`) into `barometer_combined`.

---

## Data model reminders

| Layer | GD file | Notes |
|-------|---------|--------|
| Excel source | `250808_data_RGD_DECIDE.xlsx` | Wide format; `sample_id` + `Dossier_ID` |
| API source | `gd_labresults.csv` | 10 API fields only; long format |
| Cleaned output | `barometer_gd` | Long; same columns as `barometer_GD.csv` reference |

The acceptance API may return few rows (test data). Production is expected to return more; schema stays the same per GD API v1.0 PDF in `main/Doc/`.

---

## Operational flow

1. Run `GD API.ipynb` locally when you need a fresh pull.
2. Upload `gd_labresults.csv` to the Databricks volume (Teams → volume).
3. Job **`00`** detects the change → cleaners **`01`–`06`** and **`07`** run.
4. **`03`** refreshes `barometer_gd` from Excel **and** API CSV together.

---

*Last updated: May 2026 — GD API + `03_clean_gd` dual-source integration.*
