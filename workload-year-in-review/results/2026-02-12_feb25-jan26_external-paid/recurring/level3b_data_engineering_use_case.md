# Recurring Level 3b: Data Engineering by Use Case (External Paid)

**Period:** Mar 2025 → Jan 2026  
**Filter:** Recurring queries only, External accounts (excludes Internal, Trial, Partner Access)  
**Run Date:** 2026-02-12

## Results

| Use Case | Mar 2025 Jobs | Jan 2026 Jobs | Growth % | Credits Δ % | Exec Δ % |
|----------|---------------|---------------|----------|-------------|----------|
| Transformation | 26.3B | 51.9B | +97% | +39.5% | +31.1% |
| Ingestion | 13.6B | 30.1B | +122% | +51.2% | +44.6% |
| Interoperable Storage | 73M | 302M | +317% | +29.6% | +39.8% |

## Key Observations

- **Ingestion** best credit efficiency in recurring workloads (+51.2%)
- **Interoperable Storage** strong improvement (+39.8% exec) in stable Iceberg workloads
- **All use cases positive** across the board

## SQL Query

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
    WHERE r.ds >= '2025-03-01' AND r.ds < '2026-02-01'
      AND DATE_TRUNC('month', r.ds) IN ('2025-03-01', '2026-01-01')
      AND r.feature_vector:is_recurrent::STRING = 'true'
      AND r.feature_vector:"Product Category"::STRING = 'Data Engineering'
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
mar AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
jan AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(f.use_case, j.use_case) AS use_case,
    f.total_jobs AS "Mar Jobs",
    j.total_jobs AS "Jan Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 0) AS "Growth %",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM mar f
FULL OUTER JOIN jan j ON f.use_case = j.use_case
ORDER BY "Jan Jobs" DESC
```
