# Workload Year-in-Review

Reproducible workload performance analysis for Snowflake platform reporting.

## Purpose

Generate consistent, comparable metrics showing how Snowflake platform performance has improved over time. This skill produces year-in-review style reports comparing two time periods across:

- **Job volume** - How many queries are running
- **Credit efficiency** - Credits consumed per 1,000 jobs (lower = better)
- **Execution speed** - Average execution time per job (lower = better)

## Why This Exists

Platform engineering teams need to demonstrate performance improvements to leadership. This skill ensures:

1. **Consistency** - Same methodology every time
2. **Reproducibility** - SQL queries are documented and versioned
3. **Auditability** - Results are saved with the queries that generated them
4. **Comparability** - Metrics are normalized (per-job) so growth doesn't skew results

## Analysis Hierarchy

```
Level 1: All Snowflake (total platform)
    │
    ├── Level 2: By Product Category
    │       ├── Analytics
    │       ├── Data Engineering  
    │       ├── AI/ML
    │       ├── OLTP
    │       ├── Apps & Collaboration
    │       └── Platform
    │
    ├── Level 3a: Analytics Use Cases
    │       ├── Business Intelligence
    │       ├── BI Tools (3P)
    │       ├── Interactive & Powered By
    │       ├── Applied Analytics
    │       └── Lakehouse Analytics
    │
    ├── Level 3b: Data Engineering Use Cases
    │       ├── Transformation
    │       ├── Ingestion
    │       └── Interoperable Storage
    │
    └── SPI Execution Index (separate metric)
```

## Account Filtering

By default, exclude non-customer workloads:
```sql
AND a.snowflake_account_type <> 'Internal'
AND a.agreement_type NOT IN ('Trial', 'Partner Access')
```

This ensures metrics reflect actual customer experience, not internal testing.

## Results Structure

Results are saved in dated folders:
```
results/
└── YYYY-MM-DD_period_filter/
    ├── level1_all_snowflake.md
    ├── level2_product_category.md
    ├── level3a_analytics_use_case.md
    ├── level3b_data_engineering_use_case.md
    ├── spi_execution_index.md
    └── recurring/
        └── (same structure for recurring queries only)
```

Each result file contains:
- Results table
- Key observations
- Full SQL query used

## Key Metrics

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **Credits Δ %** | `(old - new) / old × 100` | Positive = more efficient |
| **Exec Δ %** | `(old - new) / old × 100` | Positive = faster |
| **Jobs Growth %** | `(new - old) / old × 100` | Growth rate |
| **SPI Point Δ** | `ABS(new) - ABS(old)` | Positive = improved |

## Data Sources

- **Primary:** `snowscience.job_analytics.job_feature_daily_account_rollup`
- **Account filter:** `snowscience.dimensions.dim_accounts_history`
- **SPI Index:** `snowscience.job_analytics.spi_tracker_index`

## Usage

Ask: "Run year-in-review analysis for Feb 2025 to Jan 2026, external paid customers only"

The skill will:
1. Run all levels of analysis
2. Save results to `results/` folder
3. Optionally generate a Marp presentation
