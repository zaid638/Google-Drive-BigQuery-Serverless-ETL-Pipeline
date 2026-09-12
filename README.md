# Google Drive → BigQuery Serverless ETL Pipeline

A data pipeline that ingests Google Sheets from a Drive folder, lands them in BigQuery as standardized bronze tables, and merges them into a unified, cleaned silver table that fully automated on GCP.

## Overview

This pipeline automates the end-to-end flow of turning scattered Google Sheets into analytics-ready data in BigQuery, with no manual intervention after deployment.

![archtecture](https://github.com/zaid638/Google-Drive-BigQuery-Serverless-ETL-Pipeline/blob/main/BQ_data_warehose_project_2.png)
<!-- 
**Flow:**
```
Google Drive Folder → Python (extraction) → BigQuery Bronze Tables
                                                     │
                                          Metadata Table (tracking)
                                                     │
                                    SQL Transformation (scheduled query)
                                                     ▼
                                          BigQuery Silver Table
``` -->

## Architecture

- **Ingestion**: Python script connects to a Google Drive folder, detects new/updated Google Sheets, and extracts their data.
- **Bronze layer**: Each source file is loaded into its own BigQuery bronze table. Column names are standardized to `snake_case` and all columns are cast to `STRING`, so every bronze table has a consistent, predictable structure regardless of the source file's original schema.
- **Metadata tracking**: A dedicated metadata table logs which files have been processed, when, and their status that enabling idempotent runs and easy auditing.
- **Silver layer**: All bronze tables are combined into a single silver table using BigQuery's `FULL OUTER UNION ALL BY NAME`, which merges tables by column name rather than position — no manual column mapping required, even as the number and structure of source files changes over time. Cleaning and business-rule transformations are applied in this step.
- **Orchestration**: The pipeline runs as a **GCP Cloud Function**, triggered on a schedule via **GCP Cloud Scheduler**. The silver table transformation runs as a **BigQuery scheduled query**.

## Key Challenge & Solution

**Challenge**: Source Google Sheets varied in column count, column names, and data types, making a standard `UNION ALL` across bronze tables impossible (it requires identical schema and column order).

**Solution**: Standardizing bronze tables to `STRING`-typed, `snake_case` columns, then combining them with:

```sql
SELECT * FROM bronze_table_1
FULL OUTER UNION ALL BY NAME
SELECT * FROM bronze_table_2
FULL OUTER UNION ALL BY NAME
SELECT * FROM bronze_table_3
```

This matches columns by name, fills in `NULL` for any column missing from a given table, and preserves non-overlapping columns that eliminating the need for manual schema reconciliation or dynamic SQL generation as new files are added.

## Tech Stack

| Layer | Technology |
|---|---|
| Extraction | Python, Google Drive API |
| Storage / Warehouse | Google BigQuery |
| Transformation | SQL (BigQuery Standard SQL, Scheduled Queries) |
| Compute | GCP Cloud Functions |
| Orchestration | GCP Cloud Scheduler |

## Results

- Fully automated, serverless pipeline requiring no manual schema mapping when new source files are added.
- A single, queryable silver table combining heterogeneous source files with zero data loss from schema mismatches.
- Auditable processing history via the metadata table.

<!-- ## Possible Future Improvements

- Add data quality checks / validation before promoting bronze → silver.
- Migrate orchestration to Cloud Composer (Airflow) for more complex dependency management.
- Add a gold layer with business-level aggregations.

---
*Built with Python, SQL, and Google Cloud Platform (BigQuery, Cloud Functions, Cloud Scheduler).* -->
