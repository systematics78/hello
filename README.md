# Guide: Extracting Logic from Pentaho and Rebuilding in dbt

## 1. Understanding the Context

* **Pentaho (Kettle)** jobs (`.kjb`) are workflows that orchestrate data pipelines.
* Each job calls **transformations (`.ktr`)**, which contain the actual **SQL or mapping logic**.
* **dbt Core** executes SQL models on top of Databricks (or Snowflake, BigQuery, etc.), managing dependencies and materialization (table, view, incremental).
* Goal: **Extract SQL + lineage from Pentaho and re-implement as dbt models**.

---

## 2. Step-by-Step: Extracting from Pentaho

### A) From Jobs (`.kjb`)

1. Open job in **Spoon → Design view**.
2. Identify the sequence of transformations (boxes connected with arrows).
3. Double-click each transformation step → note the linked `.ktr` file.
4. Record the order: **staging, preparation, partition swaps, final load**.

### B) From Transformations (`.ktr`)

1. Open `.ktr` in Spoon.
2. In **View → Steps**, locate:

   * **Table Input / Execute SQL script** → contains source SQL.
   * **Table Output / Insert / Merge** → contains target schema.table.
   * **Filter Rows / Join Rows / Calculator / Select Values** → represent transformations.
3. Double-click each step → copy out:

   * SQL query text.
   * Input/Output table names.
   * Field mappings, joins, calculations.

### C) Parameters & Variables

1. Look for `${VAR_NAME}` placeholders inside SQL.
2. Resolve them from:

   * Job parameters (in the `.kjb`).
   * **Edit → Settings → Parameters** inside `.ktr`.
   * `kettle.properties` (user-level properties).
   * “Set Variables” steps inside jobs.
3. Document resolved values (schema, database connections, paths).

### D) Connections

* In Spoon: **View → Database Connections** → get DB type, host, schema, credentials.
* Note which connections point to **Vertica** (or other DBs).

---

## 3. Documenting the Lineage

Use a simple template (one per transformation):

| Pentaho Job | Transformation | Source (SQL/Table) | Transformation Logic | Target (Schema.Table) | Notes   |
| ----------- | -------------- | ------------------ | -------------------- | --------------------- | ------- |
| D_CASE.kjb  | temp_case.ktr  | SELECT … FROM src  | Filters, joins       | TMP_CASE              | staging |
| D_CASE.kjb  | load_dt.ktr    | TMP_CASE           | Add columns, derive  | D_CASE_STAGE          | silver  |
| D_CASE.kjb  | swap_partition | D_CASE_STAGE       | Partition swap       | D_CASE                | gold    |

---

## 4. Rebuilding in dbt

### A) Project Structure

```plaintext
dbt_project.yml
models/
  bronze/
    tmp_case.sql
  silver/
    d_case_stage.sql
  gold/
    d_case.sql
```

### B) Model Files

* Copy SQL logic from Pentaho into `.sql` files.
* Replace hard-coded references with `ref()`.

Example (`models/silver/d_case_stage.sql`):

```sql
with temp as (
    select * from {{ ref('tmp_case') }}
)
select
    id,
    status,
    case 
        when closing_date is null then 'OPEN'
        else 'CLOSED'
    end as case_status
from temp
```

### C) Materializations

* Define in `dbt_project.yml` or per model:

  * **staging (bronze)** → `view`
  * **intermediate (silver)** → `table`
  * **business/gold** → `table` or `incremental`

### D) Testing & Documentation

* Create `schema.yml` files to add:

  * Column descriptions.
  * Tests (unique, not null, foreign key).

---

## 5. Migration Workflow

1. **Identify high-value pipelines** (those feeding Power BI / CLIQ).
2. For each Pentaho job:

   * Extract SQL + lineage.
   * Resolve parameters.
   * Document inputs/outputs.
3. Create dbt models (bronze/silver/gold).
4. Validate row counts & key metrics against Vertica.
5. Switch BI (Power BI) to query dbt’s gold layer in Databricks.

---

## 6. Recommended Training Path (dbt Core)

* **Official dbt Core docs**: [https://docs.getdbt.com/docs/introduction](https://docs.getdbt.com/docs/introduction)
* **Learn dbt jaffle_shop tutorial**: [https://github.com/dbt-labs/jaffle_shop](https://github.com/dbt-labs/jaffle_shop)
* Suggested internal training flow:

  * Session 1: Basics (`ref()`, model structure, materializations).
  * Session 2: Build staging & transformations from extracted Pentaho SQL.
  * Session 3: Tests, documentation, and exposing gold models to BI.

---

👉 This way, you have a **repeatable process**:

1. Open `.kjb` → find `.ktr`.
2. Extract SQL/lineage.
3. Map into dbt bronze/silver/gold.
4. Validate outputs.
