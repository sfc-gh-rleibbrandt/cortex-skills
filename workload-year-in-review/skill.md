# Snowflake Workload Year-in-Review Analysis

Generate reproducible workload performance metrics comparing two time periods.

## Trigger

Use this skill when:
- User asks for "year in review" workload analysis
- User wants to compare workload performance across time periods
- User needs metrics by Product Category or use_case

## Before You Start - ASK THE USER

**REQUIRED:** Before running any queries, ask the user for:

1. **Start period** - e.g., "Feb 2025" (the older/baseline month)
2. **End period** - e.g., "Jan 2026" (the newer/comparison month)

Example prompt:
> "What time periods would you like to compare? Please provide:
> - Start month (baseline): e.g., Feb 2025
> - End month (comparison): e.g., Jan 2026"

If user says "year in review" without specifying, suggest comparing the same month one year apart (e.g., Feb 2025 → Jan 2026 for FY25 review).

---

## Quick Start

1. **Ask for periods** (see above)
2. Run analysis hierarchy (Level 1 → Level 2 → Level 3)
3. Include SPI Execution Index as separate metric
4. Save results to `results/YYYY-MM-DD_period_filter/` folder
5. Optionally run recurring-only analysis (note: recurring data starts Mar 2025)

---

## Account Filtering (Default: External Paid Only)

Join to account dimension and filter:

```sql
FROM snowscience.job_analytics.job_feature_daily_account_rollup r
JOIN snowscience.dimensions.dim_accounts_history a 
  ON r.deployment = a.snowflake_deployment 
  AND r.account_id = a.snowflake_account_id 
  AND r.ds = a.general_date
WHERE ...
  AND a.snowflake_account_type <> 'Internal'
  AND a.agreement_type NOT IN ('Trial', 'Partner Access')
```

This ensures metrics reflect customer experience, not internal testing.

---

## Analysis Hierarchy

```
Level 1: All Snowflake (no breakdown)
    │
    ├── Level 2: By Product Category
    │       │
    │       ├── Level 3a: Analytics by use_case
    │       │
    │       └── Level 3b: Data Engineering by use_case
    │
    └── SPI Execution Index (separate data source)
```

---

## Data Sources

| Source | Table | Use |
|--------|-------|-----|
| **Primary** | `snowscience.job_analytics.job_feature_daily_account_rollup` | All levels |
| **Account Filter** | `snowscience.dimensions.dim_accounts_history` | External paid filter |
| **SPI Index** | `snowscience.job_analytics.spi_tracker_index` | Execution index only |

⚠️ **DO NOT MIX** rollup and SPI data. Keep them as separate metrics.

---

## Feature Vector Keys

```sql
-- Product Category
feature_vector:"Product Category"::STRING  
-- Values: Analytics, Data Engineering, AI/ML, Platform, Apps & Collaboration, OLTP

-- Analytics use cases
feature_vector['Analytics']['use_case']::STRING
-- Values: Business Intelligence, BI Tools (3P), Interactive and Powered By Analytics, 
--         Applied Analytics, Lakehouse Analytics

-- Data Engineering use cases  
feature_vector['Data Engineering']['use_case']::STRING
-- Values: Transformation, Ingestion, Interoperable Storage

-- Recurring queries
feature_vector:is_recurrent::STRING = 'true'
-- Note: Recurring data available from March 2025 onwards
```

---

## Level 1: All Snowflake

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        SUM(r.jobs) AS total_jobs,
        SUM(r.total_credits) AS total_credits,
        SUM(r.dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup r
    JOIN snowscience.dimensions.dim_accounts_history a 
      ON r.deployment = a.snowflake_deployment 
      AND r.account_id = a.snowflake_account_id 
      AND r.ds = a.general_date
    WHERE r.ds >= '{{start_date}}' AND r.ds < '{{end_date}}'
      AND DATE_TRUNC('month', r.ds) IN ('{{start_month}}', '{{end_month}}')
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1
),
period1 AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
period2 AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    'All Snowflake' AS scope,
    -- Raw Jobs
    p1.total_jobs AS "{{start_month_label}} Jobs",
    p2.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(p2.total_jobs / NULLIF(p1.total_jobs, 0), 2) AS "Jobs X",
    -- Raw Credits
    ROUND(p1.total_credits, 0) AS "{{start_month_label}} Credits",
    ROUND(p2.total_credits, 0) AS "{{end_month_label}} Credits",
    -- Credits per 1K Jobs (efficiency)
    ROUND(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0), 2) AS "Credits X",
    -- Execution time
    ROUND(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0), 2) AS "Speed X",
    -- Jobs per Credit (composite metric)
    ROUND((p2.total_jobs / NULLIF(p1.total_jobs, 0)) * 
          ((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0)), 2) AS "Jobs/Credit X"
FROM period1 p1, period2 p2
```

---

## Level 2: By Product Category

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        r.feature_vector:"Product Category"::STRING AS product_category,
        SUM(r.jobs) AS total_jobs,
        SUM(r.total_credits) AS total_credits,
        SUM(r.dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup r
    JOIN snowscience.dimensions.dim_accounts_history a 
      ON r.deployment = a.snowflake_deployment 
      AND r.account_id = a.snowflake_account_id 
      AND r.ds = a.general_date
    WHERE r.ds >= '{{start_date}}' AND r.ds < '{{end_date}}'
      AND DATE_TRUNC('month', r.ds) IN ('{{start_month}}', '{{end_month}}')
      AND r.feature_vector:"Product Category"::STRING IS NOT NULL
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
p1 AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
p2 AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(p1.product_category, p2.product_category) AS product_category,
    -- Raw Jobs
    p1.total_jobs AS "{{start_month_label}} Jobs",
    p2.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 1) AS "Growth %",
    ROUND(p2.total_jobs / NULLIF(p1.total_jobs, 0), 2) AS "Jobs X",
    -- Raw Credits
    ROUND(p1.total_credits, 0) AS "{{start_month_label}} Credits",
    ROUND(p2.total_credits, 0) AS "{{end_month_label}} Credits",
    -- Credits per 1K Jobs (efficiency)
    ROUND(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0), 2) AS "Credits X",
    -- Execution time
    ROUND(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) AS "{{start_month_label}} ms",
    ROUND(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0) AS "{{end_month_label}} ms",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0), 2) AS "Speed X",
    -- Jobs per Credit (composite)
    ROUND((p2.total_jobs / NULLIF(p1.total_jobs, 0)) * 
          ((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0)), 2) AS "Jobs/Credit X"
FROM p1
FULL OUTER JOIN p2 ON p1.product_category = p2.product_category
ORDER BY p2.total_jobs DESC
```

---

## Level 3a: Analytics by Use Case

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        r.feature_vector['Analytics']['use_case']::STRING AS use_case,
        SUM(r.jobs) AS total_jobs,
        SUM(r.total_credits) AS total_credits,
        SUM(r.dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup r
    JOIN snowscience.dimensions.dim_accounts_history a 
      ON r.deployment = a.snowflake_deployment 
      AND r.account_id = a.snowflake_account_id 
      AND r.ds = a.general_date
    WHERE r.ds >= '{{start_date}}' AND r.ds < '{{end_date}}'
      AND DATE_TRUNC('month', r.ds) IN ('{{start_month}}', '{{end_month}}')
      AND r.feature_vector:"Product Category"::STRING = 'Analytics'
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
p1 AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
p2 AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(p1.use_case, p2.use_case) AS use_case,
    -- Raw Jobs
    p1.total_jobs AS "{{start_month_label}} Jobs",
    p2.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 1) AS "Growth %",
    ROUND(p2.total_jobs / NULLIF(p1.total_jobs, 0), 2) AS "Jobs X",
    -- Raw Credits  
    ROUND(p1.total_credits, 0) AS "{{start_month_label}} Credits",
    ROUND(p2.total_credits, 0) AS "{{end_month_label}} Credits",
    -- Credits efficiency
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0), 2) AS "Credits X",
    -- Execution time
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0), 2) AS "Speed X",
    -- Jobs per Credit (composite)
    ROUND((p2.total_jobs / NULLIF(p1.total_jobs, 0)) * 
          ((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0)), 2) AS "Jobs/Credit X"
FROM p1
FULL OUTER JOIN p2 ON p1.use_case = p2.use_case
ORDER BY p2.total_jobs DESC
```

**Analytics Use Cases:**
- Business Intelligence - Native Snowsight dashboards/worksheets
- BI Tools (3P) - Tableau, Power BI, Looker, Sigma, etc.
- Interactive and Powered By Analytics - Embedded analytics, APIs
- Applied Analytics - Data science, ML workloads
- Lakehouse Analytics - Iceberg/open table format analytics

---

## Level 3b: Data Engineering by Use Case

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        r.feature_vector['Data Engineering']['use_case']::STRING AS use_case,
        SUM(r.jobs) AS total_jobs,
        SUM(r.total_credits) AS total_credits,
        SUM(r.dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup r
    JOIN snowscience.dimensions.dim_accounts_history a 
      ON r.deployment = a.snowflake_deployment 
      AND r.account_id = a.snowflake_account_id 
      AND r.ds = a.general_date
    WHERE r.ds >= '{{start_date}}' AND r.ds < '{{end_date}}'
      AND DATE_TRUNC('month', r.ds) IN ('{{start_month}}', '{{end_month}}')
      AND r.feature_vector:"Product Category"::STRING = 'Data Engineering'
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
p1 AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
p2 AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(p1.use_case, p2.use_case) AS use_case,
    -- Raw Jobs
    p1.total_jobs AS "{{start_month_label}} Jobs",
    p2.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 1) AS "Growth %",
    ROUND(p2.total_jobs / NULLIF(p1.total_jobs, 0), 2) AS "Jobs X",
    -- Raw Credits  
    ROUND(p1.total_credits, 0) AS "{{start_month_label}} Credits",
    ROUND(p2.total_credits, 0) AS "{{end_month_label}} Credits",
    -- Credits efficiency
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0), 2) AS "Credits X",
    -- Execution time
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0), 2) AS "Speed X",
    -- Jobs per Credit (composite)
    ROUND((p2.total_jobs / NULLIF(p1.total_jobs, 0)) * 
          ((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0)) / NULLIF(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 0)), 2) AS "Jobs/Credit X"
FROM p1
FULL OUTER JOIN p2 ON p1.use_case = p2.use_case
ORDER BY p2.total_jobs DESC
```

**Data Engineering Use Cases:**
- Transformation - ELT/ETL (dbt, stored procedures)
- Ingestion - Data loading (COPY, Snowpipe, connectors)
- Interoperable Storage - Iceberg tables, external tables

---

## SPI Execution Index

⚠️ **SEPARATE DATA SOURCE** - Do not mix with rollup tables.

SPI Execution Index is unitless with baseline = 0. More negative = faster.

```sql
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        MAX(MONTH_INDEX_XP) AS execution_index
    FROM snowscience.job_analytics.spi_tracker_index 
    WHERE DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
    GROUP BY 1
),
period1 AS (SELECT * FROM monthly WHERE month = '{{start_month}}'),
period2 AS (SELECT * FROM monthly WHERE month = '{{end_month}}')
SELECT 
    'SPI Execution Index' AS metric,
    ROUND(p1.execution_index, 1) AS "{{start_month_label}}",
    ROUND(p2.execution_index, 1) AS "{{end_month_label}}",
    ROUND(ABS(p2.execution_index) - ABS(p1.execution_index), 1) AS "Point Δ",
    ROUND((ABS(p2.execution_index) - ABS(p1.execution_index)) / NULLIF(ABS(p1.execution_index), 0) * 100, 0) AS "% Improvement"
FROM period1 p1, period2 p2
```

---

## Recurring Queries Filter

Add to WHERE clause for stable production workloads:
```sql
AND r.feature_vector:is_recurrent::STRING = 'true'
```

**Note:** Recurring data available from March 2025 onwards. Use Mar 2025 as start for recurring analysis.

---

## Improvement Formulas

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **Credits Δ %** | `(old - new) / old × 100` | Positive = more efficient |
| **Exec Δ %** | `(old - new) / old × 100` | Positive = faster |
| **Jobs Growth %** | `(new - old) / old × 100` | Growth (not improvement) |
| **SPI Point Δ** | `ABS(new) - ABS(old)` | Positive = improved |
| **SPI % Improvement** | `Point Δ / ABS(old) × 100` | Relative improvement |

---

## X Multiplier Calculations

After running queries, calculate and present "X times" multipliers for executive-friendly summaries:

| Metric | Formula | Example |
|--------|---------|---------|
| **Jobs X** | `new_jobs / old_jobs` | 222.8B / 98.1B = **2.27x** |
| **Credits X** | `old_cr_per_1k / new_cr_per_1k` | 1.149 / 0.693 = **1.66x** cheaper |
| **Speed X** | `old_ms / new_ms` | 1516 / 946 = **1.60x** faster |
| **Jobs/Credit X** | `Jobs X × Credits X` | 2.27 × 1.66 = **3.77x** more jobs per credit |

### Presenting X Multipliers

Show a combined table with all multipliers:

```
┌─────────────────────────┬──────────┬────────────┬───────────┬─────────────────┐
│ Category                │ Jobs X   │ Credits X  │ Speed X   │  Jobs/Credit X  │
├─────────────────────────┼──────────┼────────────┼───────────┼─────────────────┤
│ All Snowflake           │   2.27x  │    1.66x   │   1.60x   │     3.77x       │
│ ...                     │   ...    │    ...     │   ...     │     ...         │
└─────────────────────────┴──────────┴────────────┴───────────┴─────────────────┘
```

**Interpretation:**
- `Jobs X` = Growth (how many more jobs)
- `Credits X` = Efficiency (how much cheaper per job)
- `Speed X` = Performance (how much faster)
- `Jobs/Credit X` = **Best overall metric** - total productivity gain per credit spent

---

## Template Variables

| Variable | Example | Description |
|----------|---------|-------------|
| `{{start_date}}` | `2025-02-01` | Start of analysis window |
| `{{end_date}}` | `2026-02-01` | End of analysis window (exclusive) |
| `{{start_month}}` | `2025-02-01` | First month to compare |
| `{{end_month}}` | `2026-01-01` | Second month to compare |
| `{{start_month_label}}` | `Feb 2025` | Label for first month |
| `{{end_month_label}}` | `Jan 2026` | Label for second month |

---

## Output

Save results to: `results/YYYY-MM-DD_period_filter/`

Structure:
```
results/2026-02-12_feb25-jan26_external-paid/
├── level1_all_snowflake.md
├── level2_product_category.md
├── level3a_analytics_use_case.md
├── level3b_data_engineering_use_case.md
├── spi_execution_index.md
└── recurring/
    ├── level1_all_snowflake.md
    ├── level2_product_category.md
    ├── level3a_analytics_use_case.md
    └── level3b_data_engineering_use_case.md
```

Each file should contain:
1. Period and filter metadata
2. Results table in markdown
3. Key observations
4. Full SQL query used
