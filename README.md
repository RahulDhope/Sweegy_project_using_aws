# Sweegy_project_using_aws

## Overview

**Sweegy** is an end-to-end data warehouse project implemented on Snowflake with ingestion from CSV files (staged in Snowflake internal stage) and orchestrated using AWS-native components where applicable. The pipeline implements a multi-layer architecture (Stage → Clean → Consumption) and supports a mix of SCD Type 1 and SCD Type 2 dimension handling, streams for change capture, masking/tagging for data governance, and consumption-ready fact tables and KPI views.

This README documents repository structure, required infrastructure, how the pipeline works, deployment and operational notes, and testing/checklist items.

## High-level architecture

1. **Source**: CSV files (initial loads + delta directories). In AWS setup these can be uploaded to an S3 bucket (or directly to Snowflake internal stage for PoC).
2. **Stage Layer**: Lightweight raw tables `stage_sch.*` ingest data via `COPY INTO` from the stage (internal/external). Streams are created on stage tables for change capture.
3. **Clean Layer**: Type-casted, normalized tables under `clean_sch.*` that enforce data types, basic quality rules and apply light transformations (e.g., state → state_code, city tiering). Streams track changes.
4. **Consumption Layer**: Dimension and fact tables (`consumption_sch.*`) — SCD Type 1 (overwrite) for some dims and SCD Type 2 (versioned) for others. Views for KPIs are created here.
5. **Orchestration / Scheduling**: AWS (Lambda + Step Functions / Glue / Airflow / Prefect) can orchestrate file arrival, COPY execution, and downstream MERGE workloads.
6. **Governance & Security**: Snowflake tags, masking policies and column-level tags are used to classify PII and sensitive attributes.

---

## Key components implemented in the SQL

* **Warehouse**: `adhoc_wh` (x-small, auto_suspend, auto_resume) for running the loads and merges.
* **Databases & Schemas**: `sandbox` DB with schemas: `stage_sch`, `clean_sch`, `consumption_sch`, `common`.
* **File Format & Internal Stage**: `stage_sch.csv_file_format` and `stage_sch.csv_stg` for CSV ingestion.
* **Tags & Masking Policies**: `common.pii_policy_tag`, `common.pii_masking_policy`, `common.email_masking_policy`, `common.phone_masking_policy`.
* **Streams & Merge Logic**: Stage tables have append-only streams (e.g., `stage_sch.location_stm`) and clean layer tables have streams to detect insert/updates. MERGE statements implement SCD Type 1 & Type 2 behaviors.
* **SCD Handling**:

  * **Type 1 (overwrite)**: `restaurant_location_dim`, `menu`, `customer` clean pipeline for dimensions that only need current attributes.
  * **Type 2 (history)**: `restaurant_dim`, `customer_dim`, `menu_dim`, `delivery_agent_dim`, `customer_address_dim` implement change history with `eff_start_date`, `eff_end_date`, and `is_current` flags.
* **Fact table**: `consumption_sch.order_item_fact` — snapshot fact with dimensional surrogate keys and degenerate attributes. Foreign-key constraints are added for metadata/consistency.
* **Date dimension**: `consumption_sch.date_dim` populated via a recursive CTE from orders date range.
* **Views for KPIs**: `vw_yearly_revenue_kpis`, `vw_monthly_revenue_kpis`, `vw_daily_revenue_kpis`, `vw_day_revenue_kpis`, `vw_monthly_revenue_by_restaurant`.

---

## Prerequisites

* Snowflake account with appropriate rights to create warehouses, databases, schemas, stages, file formats, tables, streams, tasks, and masking policies.
* If using S3: an AWS account with an S3 bucket and an IAM user/role configured for Snowflake external stage access (or use Snowflake internal stage for PoC).
* Optional: Terraform/CloudFormation templates for reproducible infra.

---

## Quickstart / Setup

1. Create or confirm Snowflake role with required privileges.
2. Run the provided SQL (e.g., `sql/setup_all.sql`) in the following order (script is idempotent in many places):

   * Warehouse and DB/Schema creation
   * File format and stage creation
   * Tag and masking policy creation
   * Stage table creation and COPY commands for initial loads
   * Clean layer table creation and MERGE logic
   * Consumption layer table creation and MERGE logic for SCDs
   * Fact table and views
3. Upload CSV files to the stage paths used in the script:

   * `@stage_sch/csv_stg/initial/...`
   * `@stage_sch/csv_stg/delta/...`
     (If using S3, create external stage pointing to S3 and update COPY source paths.)
4. Execute initial `COPY INTO` commands to populate stage tables, then run the MERGE statements for clean and consumption layers.

---

## Orchestration recommendations (AWS)

* Use S3 event notifications (on `PUT`) to trigger a Lambda or Step Function which:

  1. Validates file naming and paths
  2. Triggers an Airflow/Step Function workflow or invokes Snowflake using the Snowflake Python connector / Snowpipe REST API to run COPY and MERGE jobs
* For real-time ingestion, consider Snowpipe for continuous loads from S3.
* Monitor query performance and warehouse credit usage — scale warehouse size/policy for production.

---

## Data governance & security

* Tag PII columns with `common.pii_policy_tag` and apply masking policies to sensitive columns (phone, email, mobile, etc.).
* Restrict role-based access to consumption schemas; create a `read_only` role for BI users.
* Audit via Snowflake Access History and Query History.

---

## Testing & validation checklist

* Confirm files land in the stage (list the stage path and check `metadata$filename`).
* Run COPY commands and validate row counts in `stage_sch.*` tables.
* Validate streaming: check stream offsets and `metadata$action` flags.
* After MERGE, assert surrogate/hash keys and row counts in `clean_sch.*` and `consumption_sch.*`.
* Run sample KPI views and compare totals with raw totals to ensure aggregations are consistent.

---

## Common issues & troubleshooting

* **COPY errors**: Use `ON_ERROR = CONTINUE` temporarily for debugging and check `VALIDATION_MODE` to preview parse issues.
* **Type casting failures**: Use `TRY_CAST`/`TRY_TO_TIMESTAMP` to surface data problems without aborting whole loads.
* **Stream stale**: Streams progress only when DML executes against base table; ensure the stream is read or offset is consumed during MERGE.
* **SCD2 overlaps**: Ensure `IS_CURRENT` is enforced and `EFF_END_DATE` is set when expiring records.

---

## Conventions & coding standards

* Object naming: `<schema>.<object>` where schema indicates layer (`stage_sch`, `clean_sch`, `consumption_sch`).
* Use `hash(SHA1_hex(...))` for deterministic business hash keys (HK) in SCD2 dims.
* Streams must be consumed (MERGE or SELECT) to advance offsets — plan tasks accordingly.

---

## Next steps / enhancements

* Convert internal stage usage to S3 + Snowpipe for continuous production ingestion.
* Add Snowflake Tasks for automating MERGE workflows on schedule.
* Add unit tests (dbt or custom SQL-based assertions) for data quality checks.
* Add Terraform to codify Snowflake and AWS infra.
