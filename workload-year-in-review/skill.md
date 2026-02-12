# Snowflake Workload Year-in-Review Analysis

Generate reproducible workload performance metrics for Snowflake platform year-in-review reporting.

## Trigger
Use this skill when:
- User asks for "year in review" workload analysis
- User wants to compare workload performance across time periods
- User needs metrics by Product Category, use_case, or statement type

## Analysis Hierarchy

Run analyses in this order (top-down drill-down):

```
Level 1: All Snowflake (no breakdown)
    │
    ├── Level 2: By Product Category
    │       │
    │       ├── Level 3a: Analytics by use_case
    │       │
    │       └── Level 3b: Data Engineering by use_case
    │
    ├── DML Analysis (statement_type_bucket = 'DML')
    │       │
    │       ├── DML - All Snowflake
    │       │
    │       └── DML - By Product Category
    │
    └── SPI Execution Index (separate data source - single metric)
```

**IMPORTANT:** Do NOT break down SPI by statement type unless explicitly requested - it doesn't add much value.

---

## Data Sources

### PRIMARY: Job Feature Rollup Tables (Levels 1-3, DML)

**Table:** `snowscience.job_analytics.job_feature_daily_account_rollup`
- **Connection:** snowhouse
- **Granularity:** ds + deployment + account_id + feature_vector
- **Date Range:** May 2022 → Present

### SECONDARY: SPI Tables (Overall metric only)

**Tables:**
- `snowscience.job_analytics.spi_stable_warehouse_dml_queries` - All DML queries
- `snowscience.job_analytics.spi_stable_warehouse_select_queries` - All SELECT queries

⚠️ **DO NOT MIX** rollup and SPI data. Keep them as separate metrics.

⚠️ **DO NOT BREAK DOWN** SPI by statement type unless explicitly asked.

---

## Feature Vector Keys

### Core Dimensions
```sql
feature_vector:"Product Category"::STRING  
-- Values: Analytics, Data Engineering, AI/ML, Platform, Apps & Collaboration, OLTP

feature_vector:statement_type_bucket::STRING  
-- Values: SELECT, DML, COPY, CALL, Other, AUTO_RECLUSTER, SHOW, UNLOAD, etc.

feature_vector:is_recurrent::STRING  
-- Values: 'true' or NULL
```

### Use Case Breakdowns
```sql
-- Analytics use cases
feature_vector['Analytics']['use_case']::STRING
-- Values: Business Intelligence, BI Tools (3P), Interactive and Powered By Analytics, 
--         Applied Analytics, Lakehouse Analytics

-- Data Engineering use cases  
feature_vector['Data Engineering']['use_case']::STRING
-- Values: Transformation, Ingestion, Interoperable Storage
```

### Metrics Available
- `JOBS` - job count
- `TOTAL_CREDITS` - total credits consumed
- `DUR_XP_EXECUTING` - execution duration (ms)
- `DUR_COMPILING` - compilation time (ms)
- `QUEUE_TIME` - queue time (ms)

---

## Level 1: All Snowflake (No Breakdown)

Total platform performance with no dimensional breakdown.

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
    GROUP BY 1
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    'All Snowflake' AS scope,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f, jan j
```

---

## Level 2: By Product Category

Breakdown by Product Category dimension.

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        feature_vector:"Product Category"::STRING AS product_category,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
      AND feature_vector:"Product Category"::STRING IS NOT NULL
    GROUP BY 1, 2
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(f.product_category, j.product_category) AS product_category,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.product_category = j.product_category
ORDER BY "{{end_month_label}} Jobs" DESC
```

---

## Level 3a: Analytics by Use Case

Drill-down into Analytics product category by use_case.

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        feature_vector['Analytics']['use_case']::STRING AS use_case,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
      AND feature_vector:"Product Category"::STRING = 'Analytics'
    GROUP BY 1, 2
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(f.use_case, j.use_case) AS use_case,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.use_case = j.use_case
ORDER BY "{{end_month_label}} Jobs" DESC
```

**Analytics Use Cases:**
- Business Intelligence - Native Snowsight dashboards/worksheets
- BI Tools (3P) - Third-party tools (Tableau, Power BI, Looker, Sigma, etc.)
- Interactive and Powered By Analytics - Embedded analytics, APIs
- Applied Analytics - Data science, ML workloads classified under Analytics
- Lakehouse Analytics - Iceberg/open table format analytics

---

## Level 3b: Data Engineering by Use Case

Drill-down into Data Engineering product category by use_case.

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        feature_vector['Data Engineering']['use_case']::STRING AS use_case,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
      AND feature_vector:"Product Category"::STRING = 'Data Engineering'
    GROUP BY 1, 2
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(f.use_case, j.use_case) AS use_case,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.use_case = j.use_case
ORDER BY "{{end_month_label}} Jobs" DESC
```

**Data Engineering Use Cases:**
- Transformation - ELT/ETL transformations (dbt, stored procedures, etc.)
- Ingestion - Data loading (COPY, Snowpipe, connectors)
- Interoperable Storage - Iceberg tables, external tables

---

## DML Analysis

Filter rollup table by `statement_type_bucket = 'DML'`.

### DML - All Snowflake

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
      AND feature_vector:statement_type_bucket::STRING = 'DML'
    GROUP BY 1
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    'DML (All Snowflake)' AS scope,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f, jan j
```

### DML - By Product Category

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        feature_vector:"Product Category"::STRING AS product_category,
        SUM(jobs) AS total_jobs,
        SUM(total_credits) AS total_credits,
        SUM(dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup
    WHERE ds >= '{{start_date}}' AND ds < '{{end_date}}'
      AND DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
      AND feature_vector:statement_type_bucket::STRING = 'DML'
      AND feature_vector:"Product Category"::STRING IS NOT NULL
    GROUP BY 1, 2
),
feb AS (SELECT * FROM metrics WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM metrics WHERE month = '{{end_month}}')
SELECT 
    COALESCE(f.product_category, j.product_category) AS product_category,
    f.total_jobs AS "{{start_month_label}} Jobs",
    j.total_jobs AS "{{end_month_label}} Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "{{start_month_label}} Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "{{end_month_label}} Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "{{start_month_label}} Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "{{end_month_label}} Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.product_category = j.product_category
ORDER BY "{{end_month_label}} Jobs" DESC
```

---

## SPI Execution Index (Separate Data Source)

⚠️ **SEPARATE DATA SOURCE** - Do not mix with rollup tables.

⚠️ **EXECUTION INDEX ONLY** - Only show Execution Index unless explicitly asked for other metrics.

SPI Execution Index is a unitless metric with baseline = 0. More negative = faster (improvement).

**Table:** `snowscience.job_analytics.spi_tracker_index`

```sql
SELECT 
    DATE_TRUNC('month', ds) AS month,
    MAX(MONTH_INDEX_XP) AS execution_index
FROM snowscience.job_analytics.spi_tracker_index 
WHERE DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
GROUP BY 1
ORDER BY 1
```

**With improvement calculation:**

```sql
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        MAX(MONTH_INDEX_XP) AS execution_index
    FROM snowscience.job_analytics.spi_tracker_index 
    WHERE DATE_TRUNC('month', ds) IN ('{{start_month}}', '{{end_month}}')
    GROUP BY 1
),
feb AS (SELECT * FROM monthly WHERE month = '{{start_month}}'),
jan AS (SELECT * FROM monthly WHERE month = '{{end_month}}')
SELECT 
    'SPI Execution Index' AS metric,
    ROUND(f.execution_index, 1) AS "{{start_month_label}}",
    ROUND(j.execution_index, 1) AS "{{end_month_label}}",
    ROUND(ABS(j.execution_index) - ABS(f.execution_index), 1) AS "Point Δ",
    ROUND((ABS(j.execution_index) - ABS(f.execution_index)) / NULLIF(ABS(f.execution_index), 0) * 100, 0) AS "% Improvement"
FROM feb f, jan j
```

**Interpretation:**
- Baseline = 0
- More negative = faster (improvement)
- Point Δ: positive = improved (index got more negative/faster)
- % Improvement: relative improvement vs starting point

---

## Calculating Improvement

**Improvement Formula:** `(old_value - new_value) / old_value * 100`

- For duration metrics: improvement is POSITIVE when new < old (faster = better)
- For credits/1K: improvement is POSITIVE when new < old (more efficient = better)
- For jobs growth: use `(new - old) / old * 100` (growth, not improvement)

---

## Template Variables

Replace these placeholders when running queries:

| Variable | Example | Description |
|----------|---------|-------------|
| `{{start_date}}` | `2025-02-01` | Start of analysis window |
| `{{end_date}}` | `2026-02-01` | End of analysis window (exclusive) |
| `{{start_month}}` | `2025-02-01` | First month to compare |
| `{{end_month}}` | `2026-01-01` | Second month to compare |
| `{{start_month_label}}` | `Feb 2025` | Label for first month |
| `{{end_month_label}}` | `Jan 2026` | Label for second month |

---

## Output Format

Save all analysis results to a markdown file with:
1. Summary tables in markdown format
2. The exact SQL query used to generate each result
3. Key insights/observations
4. Clear labeling of data source (Rollup vs SPI)

---

## Reference: Results from Feb 2025 → Jan 2026 Analysis

### Level 1: All Snowflake
| Scope | Feb 2025 Jobs | Jan 2026 Jobs | Growth | Feb 2025 Cr/1K | Jan 2026 Cr/1K | Credits Δ | Feb 2025 Avg | Jan 2026 Avg | Exec Δ |
|-------|---------------|---------------|--------|----------------|----------------|-----------|--------------|--------------|--------|
| **All Snowflake** | 105.9B | 245.7B | +132% | 1.22 | 0.73 | **+40.2%** | 1,519 ms | 950 ms | **+37.4%** |

### Level 2: By Product Category
| Product Category | Feb 2025 Jobs | Jan 2026 Jobs | Growth | Credits Δ | Exec Δ |
|------------------|---------------|---------------|--------|-----------|--------|
| Data Engineering | 54.5B | 119.2B | +119% | +37.9% | +32.7% |
| Analytics | 47.0B | 113.9B | +143% | +45.1% | +43.9% |
| OLTP | 2.2B | 5.5B | +149% | -9.9% | -11.9% |
| Apps & Collab | 1.5B | 4.1B | +172% | +44.4% | +46.8% |
| AI/ML | 468M | 2.2B | +381% | +47.7% | +72.8% |
| Platform | 184M | 659M | +258% | +50.1% | +25.5% |

### Level 3a: Analytics by Use Case
| Use Case | Jan 2026 Jobs | Credits Δ | Exec Δ |
|----------|---------------|-----------|--------|
| Business Intelligence | 94.9B | +47.9% | +47.5% |
| Interactive & Powered By | 9.5B | +34.3% | +24.8% |
| BI Tools (3P) | 7.5B | +58.9% | +62.0% |
| Applied Analytics | 1.5B | -16.1% | -16.1% |
| Lakehouse Analytics | 423M | +66.7% | +61.1% |

### Level 3b: Data Engineering by Use Case
| Use Case | Jan 2026 Jobs | Credits Δ | Exec Δ |
|----------|---------------|-----------|--------|
| Transformation | 71.1B | +34.9% | +30.6% |
| Ingestion | 47.7B | +51.8% | +43.6% |
| Interoperable Storage | 433M | +10.4% | +27.9% |

### DML Analysis
| Scope | Jan 2026 Jobs | Credits Δ | Exec Δ |
|-------|---------------|-----------|--------|
| DML (All Snowflake) | 33.6B | +7.8% | +13.7% |
| DML - Data Engineering | 29.4B | +1.1% | +6.4% |
| DML - OLTP | 2.8B | +72.3% | +87.7% |

### SPI Execution Index
| Metric | Feb 2025 | Jan 2026 | Point Δ | % Improvement |
|--------|----------|----------|---------|---------------|
| **Execution Index** | -45.7 | -59.0 | +13.2 | **+29%** |
