# AI + Data Engineer Challenge — n8n Workflow Deliverable (US)

This README documents the **exact** ETL + KPI workflow implemented in your n8n project and how to run, test, and extend it.

---

## 1) What this workflow does

**Ingestion path (manual run):**
1) **Set variables (Edit Fields)** → computes:
   - `name_folder = external_files/ads_spend_<YYYY_MM_DD>.csv`
   - `gcs_uri = gs://challenge-ads-etl-sl-us/external_files/ads_spend_<YYYY_MM_DD>.csv`
   - `external_folder = external_files/`
   - `name = ads_spend_<YYYY_MM_DD>.csv`
   - `proyecto = esh-home-assistant-52875`
2) **Download file (Google Drive)** → pulls CSV from Drive (the shared file URL).
3) **Create an object (GCS)** → uploads the CSV to the bucket at `name_folder`.
4) **CREATE RAW (BigQuery)** → creates dataset `agency_demo` (if missing) and partitioned `ads_spend_raw`.
5) **CREATE STAGING (BigQuery)** → builds (or replaces) **external table** `ads_spend_staging` pointing at the uploaded file URI.
6) **INSERT STAGE INTO RAW (BigQuery)** → idempotency is enforced by:
   - `DELETE FROM ads_spend_raw WHERE source_file_name = <name>`
   - followed by `INSERT ... SELECT ... CAST(...)` from the external table, plus `CURRENT_DATE()` + `source_file_name`.
7) **Create KPI Daily (BigQuery)** → creates `daily_agg` and `daily_kpis`.
8) **30 Day KPI (BigQuery)** → creates `rolling_kpis_last30_vs_prior30_by_day` (30 observed days vs 30 observed prior).

**API path (webhook run):**
- `GET /challenge/metrics?start=YYYY-MM-DD&end=YYYY-MM-DD`
  → **BigQuery** selects rolling 30‑vs‑30 rows in the requested date range and
  → **Respond to Webhook** returns JSON to the client.

> All node names, SQL, and connections mirror the `Challenge.json` workflow.

---

## 2) Prerequisites

- **Project:** `esh-home-assistant-52875` (billing enabled).
- **Locations:** keep **US** for GCS bucket and BigQuery dataset.
- **Service Account** used by n8n must have:
  - `roles/storage.objectAdmin` on the bucket
  - `roles/bigquery.jobUser` at project level
  - `roles/bigquery.dataEditor` on dataset `agency_demo`
- n8n credentials configured for:
  - **Google Drive** OAuth (to read the CSV)
  - **Google Cloud Storage** OAuth (to write the object)
  - **Google Service Account** (for BigQuery queries)

---

## 3) How to run it

### A) Ingestion (manual)
1. In n8n, open the **Challenge** workflow.
2. Click **Execute workflow** (runs the left/top branch).
3. Verify:
   - GCS object created at `gs://challenge-ads-etl-sl-us/external_files/ads_spend_<YYYY_MM_DD>.csv`
   - BigQuery tables under `esh-home-assistant-52875.agency_demo`:
     - `ads_spend_raw` (partitioned by `date`)
     - `ads_spend_staging` (external table)
     - `daily_agg`, `daily_kpis`
     - `rolling_kpis_last30_vs_prior30_by_day`

### B) API (webhook)
Call:
```bash
curl "<YOUR_N8N_URL>/webhook/challenge/metrics?start=2025-07-15&end=2025-08-15"
