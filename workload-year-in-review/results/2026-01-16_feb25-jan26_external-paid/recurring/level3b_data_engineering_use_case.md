# Recurring Level 3b: Data Engineering by Use Case (External Paid)
**Period:** Mar 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Use Case | Mar Jobs | Jan Jobs | Growth | Credits Δ | Exec Δ |
|----------|----------|----------|--------|-----------|--------|
| Transformation | 26.3B | 51.9B | +97% | +39.5% | +31.1% |
| Ingestion | 13.6B | 30.1B | +122% | **+51.2%** | +44.6% |
| Interoperable Storage | 73M | 302M | +317% | +29.6% | +39.8% |

## Highlights
- **Best efficiency:** Ingestion (+51.2% credits)
- **Fastest growing:** Interoperable Storage (+317%)
- **Consistent with all-workloads:** Ingestion leads efficiency in both views

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
p1 AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(p1.use_case, p2.use_case) AS use_case,
    p1.total_jobs AS "Mar Jobs",
    p2.total_jobs AS "Jan Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 0) AS "Growth %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM p1
FULL OUTER JOIN p2 ON p1.use_case = p2.use_case
ORDER BY "Jan Jobs" DESC;
```
